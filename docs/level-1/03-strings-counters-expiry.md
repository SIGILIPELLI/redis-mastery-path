---
description: "Redis strings in practice: SET options (NX, XX, GET, KEEPTTL, IFEQ), atomic counters, float pitfalls, TTLs and the int/embstr/raw encodings."
---

# 03 · Strings, Counters & Expiry

The Redis **string** is the simplest type and the one you'll use most. A string is any sequence of bytes up to
512 MB: text, a JSON blob, a serialized protobuf, an image thumbnail, or an integer that Redis knows how to
increment. Most of what makes strings useful lies in the options on `SET` and in a handful of atomic
operations, and almost every key in a real deployment also has an **expiry**.

## SET and its options

```text
127.0.0.1:6379> SET page:home "<h1>Home</h1>"
OK
127.0.0.1:6379> GET page:home
"<h1>Home</h1>"
```

`SET` overwrites unconditionally. Its options turn it into several different operations:

```text
127.0.0.1:6379> SET page:home "<h1>Welcome</h1>" NX
(nil)
127.0.0.1:6379> SET page:about "<h1>About</h1>" XX
(nil)
127.0.0.1:6379> GET page:about
(nil)
```

- **`NX`** — set only if the key does **not** exist. It returned `(nil)` because `page:home` was already there.
  This is "create once", the basis of idempotency keys and locks.
- **`XX`** — set only if the key **does** exist. `page:about` didn't exist, so nothing was written.

**`GET`** returns the old value in the same step, which is useful when you need to know what you replaced:

```text
127.0.0.1:6379> SET config:mode blue GET
(nil)
127.0.0.1:6379> SET config:mode green GET
"blue"
```

Since **Redis 8.4**, `SET` can also compare before writing. `IFEQ` writes only if the current value equals
the one you give, which makes it an atomic compare-and-set:

```text
127.0.0.1:6379> SET lock owner-1
OK
127.0.0.1:6379> SET lock owner-2 IFEQ owner-9
(nil)
127.0.0.1:6379> SET lock owner-2 IFEQ owner-1
OK
127.0.0.1:6379> GET lock
"owner-2"
```

Its partner `DELEX` (also 8.4) deletes only if the value matches, which is exactly what releasing a lock needs:

```text
127.0.0.1:6379> DELEX lock IFEQ owner-1
(integer) 0
127.0.0.1:6379> DELEX lock IFEQ owner-2
(integer) 1
```

`IFNE` means "if not equal". `IFDEQ` and `IFDNE` compare against a *digest* of the value (the `DIGEST` command
returns it), so you don't have to send a large value back just to compare it. On older servers, check `HELP SET`
before relying on these. Valkey 9.1.2 accepted `SET … IFEQ` but has no `DELEX` or `DIGEST` (Level 4 · 09). Level 2 · 09
shows the Lua-script equivalent that works everywhere.

## Many keys at once

```text
127.0.0.1:6379> MSET a 1 b 2 c 3
OK
127.0.0.1:6379> MGET a b nope c
1) "1"
2) "2"
3) (nil)
4) "3"
127.0.0.1:6379> MSETNX c 30 d 40
(integer) 0
127.0.0.1:6379> MGET c d
1) "3"
2) (nil)
```

`MGET` returns one slot per requested key, with `(nil)` holding the place of missing ones, so results line up
with your input. `MSETNX` is all-or-nothing: because `c` existed, **neither** key was written, including `d`.
Multi-key commands save round trips, but in Redis Cluster every key in one command has to live in the same hash
slot (Level 4 · 05).

## Partial updates

```text
127.0.0.1:6379> APPEND log "line1;"
(integer) 6
127.0.0.1:6379> APPEND log "line2;"
(integer) 12
127.0.0.1:6379> GETRANGE log 0 4
"line1"
127.0.0.1:6379> STRLEN log
(integer) 12
127.0.0.1:6379> SETRANGE log 0 LINE1
(integer) 12
127.0.0.1:6379> GET log
"LINE1;line2;"
```

`APPEND` creates the key if it's missing and returns the new length. `GETRANGE` and `SETRANGE` work on byte
offsets, not characters, so be careful with UTF-8 text. These commands are handy for fixed-layout binary
records, but if you're appending log lines you probably want a list or a stream (Level 1 · 06, Level 2 · 05).

## Counters

A string holding a decimal integer can be changed in place:

```text
127.0.0.1:6379> SET visits 0
OK
127.0.0.1:6379> INCR visits
(integer) 1
127.0.0.1:6379> INCRBY visits 10
(integer) 11
127.0.0.1:6379> DECRBY visits 3
(integer) 8
127.0.0.1:6379> GET visits
"8"
```

`INCR` on a missing key treats it as 0, so you don't have to initialize counters. The reason to use `INCR`
rather than reading the value and writing back value + 1 shows up under concurrency. This script has 16 threads
that each add one like 5,000 times, both ways:

```python
from concurrent.futures import ThreadPoolExecutor
import redis

r = redis.Redis(decode_responses=True)
r.mset({"likes:rmw": 0, "likes:incr": 0})

def rmw(_):
    n = int(r.get("likes:rmw"))
    r.set("likes:rmw", n + 1)

def incr(_):
    r.incr("likes:incr")

with ThreadPoolExecutor(16) as pool:
    list(pool.map(rmw, range(5000)))
    list(pool.map(incr, range(5000)))

print(r.mget("likes:rmw", "likes:incr"))
```

```text
['570', '5000']
```

A second run printed `['556', '5000']`. With read-modify-write, close to **89% of the likes vanished**: many
threads read the same number and wrote back the same "+1". `INCR` was exact both times because the whole
read-add-write happens inside one command on Redis's single command thread. If you remember one thing from this
lesson, make it this: **push the change to Redis instead of the result.**

Counters are 64-bit signed integers and refuse to wrap around:

```text
127.0.0.1:6379> SET big 9223372036854775807
OK
127.0.0.1:6379> INCR big
(error) ERR increment or decrement would overflow
```

### Floats

`INCRBYFLOAT` exists, but read its output carefully:

```text
127.0.0.1:6379> INCRBYFLOAT price 0.1
"0.10000000000000001"
127.0.0.1:6379> INCRBYFLOAT price 0.2
"0.30000000000000004"
```

Redis does the arithmetic in binary floating point and stores the result as a string. On this version, the
string shows the full 17-significant-digit representation, so the binary rounding error that some other tools
hide is plainly visible. Never keep money in floats. Store integer cents with `INCRBY`, and format at the edge
of your application.

## Expiry

Any key can be given a time to live. When it runs out, the key disappears as if it had been deleted.

```text
127.0.0.1:6379> SET token abc123 EX 30
OK
127.0.0.1:6379> TTL token
(integer) 30
127.0.0.1:6379> PTTL token
(integer) 29884
```

`EX` sets the TTL in seconds and `PX` in milliseconds. `EXAT` and `PXAT` take an absolute Unix time. `TTL` and
`PTTL` report what's left, and they have two special answers:

```text
127.0.0.1:6379> PERSIST token
(integer) 1
127.0.0.1:6379> TTL token
(integer) -1
127.0.0.1:6379> TTL nosuchkey
(integer) -2
```

`-1` means the key exists with no expiry. `-2` means there's no such key.

### The overwrite trap

This catches everyone at least once:

```text
127.0.0.1:6379> SET token abc123 EX 30
OK
127.0.0.1:6379> SET token xyz789
OK
127.0.0.1:6379> TTL token
(integer) -1
```

A plain `SET` **removes the TTL**. A session that was supposed to expire in 30 minutes now lives forever, and
because "forever" in a cache means "until memory runs out", the leak is slow and quiet. If you mean to change
the value and keep the expiry, say so:

```text
127.0.0.1:6379> SET token abc123 EX 30
OK
127.0.0.1:6379> SET token xyz789 KEEPTTL
OK
127.0.0.1:6379> TTL token
(integer) 30
```

`INCR`, `APPEND`, `SETRANGE` and the other in-place modifiers keep the TTL. Only commands that *replace* the
value (`SET`, `GETSET`, `MSET`) reset it.

### Conditional and absolute expiry

`EXPIRE` takes the flags `NX` (only if the key has no TTL), `XX` (only if it has one), `GT` (only if the new
TTL is longer) and `LT` (only if it's shorter):

```text
127.0.0.1:6379> EXPIRE token 10 GT
(integer) 0
127.0.0.1:6379> EXPIRE token 100 GT
(integer) 1
127.0.0.1:6379> TTL token
(integer) 100
```

`GT` lets many writers each extend a deadline without any of them accidentally shortening it. `EXPIRETIME`
returns the absolute Unix time of expiry, which is useful when you copy a key's deadline elsewhere:

```text
127.0.0.1:6379> EXPIRE visits 60
(integer) 1
127.0.0.1:6379> EXPIRETIME visits
(integer) 1791855038
```

### Read and change expiry in one step

```text
127.0.0.1:6379> GETEX visits PERSIST
"8"
127.0.0.1:6379> TTL visits
(integer) -1
127.0.0.1:6379> GETDEL config:mode
"green"
127.0.0.1:6379> EXISTS config:mode
(integer) 0
```

`GETEX key EX 1800` is the "sliding session" pattern: read the session and push its expiry forward, atomically.
`GETDEL` reads and removes, which suits single-use values like a one-time login code.

## How It Actually Works

A string value is stored in one of three **encodings**, which you can see with `OBJECT ENCODING`:

```text
127.0.0.1:6379> OBJECT ENCODING visits
"int"
127.0.0.1:6379> OBJECT ENCODING page:home
"embstr"
127.0.0.1:6379> SET long "aaaa…(87 characters)…aaa"
OK
127.0.0.1:6379> OBJECT ENCODING long
"raw"
127.0.0.1:6379> SET n 12345
OK
127.0.0.1:6379> OBJECT ENCODING n
"int"
127.0.0.1:6379> APPEND n 6
(integer) 6
127.0.0.1:6379> OBJECT ENCODING n
"raw"
```

- **`int`** — a value that parses as a 64-bit integer is stored as the integer itself, inside the object header,
  with no separate string allocation. This is why `INCR` is cheap: it adds to a machine word.
- **`embstr`** — short strings (up to 44 bytes on 64-bit builds) are allocated in a single block together with
  the object header. That's one allocation and good cache locality, but the block can't be resized.
- **`raw`** — longer strings, or any string that has been modified in place, get a separately allocated SDS
  buffer that can grow. `APPEND n 6` had to turn the integer into real text and then extend it, so the encoding
  changed for good.

The encoding is invisible to clients (`GET` always returns bytes) but it affects memory and speed.

**Expiry** is kept in a second hash table, the *expires dictionary*, that maps keys to an absolute expiry time
in milliseconds. `TTL` subtracts the current time from that entry, and `PERSIST` deletes it. Expired keys are
removed in two ways. **Lazily**, any command that touches a key first checks whether it has expired and deletes
it if so, so you can never read an expired value. **Actively**, a background cycle samples keys from the
expires table several times a second and deletes the expired ones it finds, keeping its CPU use bounded. So an
expired key that nobody reads can stay in memory for a while after its deadline. Level 3 · 02 measures this.

Because an absolute time is stored, a replica or a reloaded snapshot computes the same deadline. Expiry
depends on the server's clock, so a clock jump on the server changes when keys expire.

## Common mistakes

- **Read-modify-write counters.** The race above lost about 89% of increments. Use `INCR`/`INCRBY`, or a
  transaction or script for anything more complex (Level 2 · 08–09).
- **Plain `SET` on a key that should expire.** It silently removes the TTL. Use `SET … EX` again or
  `SET … KEEPTTL`.
- **Floats for money.** `INCRBYFLOAT` shows binary rounding error. Use integer minor units.
- **Huge strings.** A single 100 MB value is read, copied and sent on the one command thread. Split large blobs,
  or keep them in object storage and store only a pointer in Redis.
- **Forgetting that `MSETNX` is all-or-nothing.** One existing key blocks the whole write.
- **Assuming `GETRANGE` counts characters.** It counts bytes, and a multibyte UTF-8 character can be cut in half.

## Exercise

1. Build an idempotency check. A payment request carries an ID. `SET idem:<id> processing NX EX 86400`
   succeeds only for the first attempt. Simulate three retries of the same ID and show that only one
   "processes". What should you store in the key once processing finishes, and why?
2. Make the race script lose updates with `INCRBYFLOAT` and an integer counter at the same time. Then fix both
   with server-side operations.
3. Write a sliding session: `SET session:<id> <json> EX 30`, then in a loop every 10 seconds call `GETEX … EX 30`
   and print the `TTL`. Stop the loop and confirm the key disappears about 30 seconds later.
4. Find the exact length at which `OBJECT ENCODING` switches from `embstr` to `raw` on your server by
   setting strings of increasing length.
