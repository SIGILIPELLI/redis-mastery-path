---
description: "Design Redis key names, iterate the keyspace safely with SCAN instead of KEYS, and delete big keys or key patterns without blocking the server."
---

# 04 · The Keyspace: Naming, SCAN & Safe Deletes

Redis has no tables, schemas or folders. There's one flat namespace of keys per database, and the only
structure is whatever you put in the names. That makes key design the closest thing Redis has to schema
design. It also means operational tasks like "find all the cache keys", "delete this tenant" or "what's using
the memory?" all come down to walking that flat namespace, and walking it the wrong way blocks the server.

## Naming keys

The near-universal convention is colon-separated segments, from general to specific:

```text
user:1042:profile        a string or hash for one user
user:1042:sessions       a set of that user's session ids
session:9f3c2a           one session
cache:api:v2:/products?page=3
rate:login:203.0.113.7
leaderboard:2026-10
```

Some guidelines that pay off later:

- **Lead with the object type.** Then `SCAN MATCH cache:*` finds all cache entries, and memory reports can be
  grouped by prefix.
- **Put the variable part where a pattern can match it.** `cache:api:*` is easy to match. A key that buries the
  type in the middle is not.
- **Add a version segment to cached data** (`cache:api:v2:…`). When the format changes you bump it, old keys
  simply age out through their TTLs, and you never have to mass-delete.
- **Keep keys reasonably short.** Every key is stored in full, once per key. With 100 million keys, 30 extra
  bytes per name is 3 GB. Don't shorten names until they're unreadable, though; `u:1:p` costs more in
  debugging time than it saves.
- **Plan for Cluster now.** In Redis Cluster, keys are spread over nodes by a hash of the name, and multi-key
  commands only work when all their keys hash to the same slot. Braces mark the part that gets hashed:
  `user:{1042}:profile` and `user:{1042}:sessions` always land together (Level 4 · 05).

## Inspecting keys

```text
127.0.0.1:6379> DBSIZE
(integer) 43
127.0.0.1:6379> TYPE user:1:prefs
hash
127.0.0.1:6379> TYPE queue:emails
list
127.0.0.1:6379> TYPE nope
none
127.0.0.1:6379> EXISTS user:1:profile user:2:profile nope
(integer) 2
```

`EXISTS` with several keys returns how many of them exist, not a yes/no answer. `OBJECT IDLETIME key` says how
many seconds have passed since the key was last read or written, which helps you spot data nobody uses.

Renaming and copying:

```text
127.0.0.1:6379> RENAME session:1 session:one
OK
127.0.0.1:6379> RENAMENX session:2 session:one
(integer) 0
127.0.0.1:6379> COPY user:1:prefs user:2:prefs
(integer) 1
127.0.0.1:6379> HGETALL user:2:prefs
1) "theme"
2) "dark"
```

`RENAME` silently **overwrites** an existing destination; `RENAMENX` refuses to. `COPY` duplicates a value of
any type, and it also refuses to overwrite unless you add `REPLACE`.

## Why not `KEYS`?

`KEYS pattern` returns every matching key in a single reply. To do that it has to visit **every key in the
database**, whatever the pattern, and it does it in one command on the one command thread. Lesson 01 measured a
`KEYS *` on a million keys holding up an unrelated `PING` for 149 ms. `KEYS` is fine on your laptop, but it
doesn't belong in application code. Many production setups rename or block it with ACLs (Level 4 · 07).

## SCAN: iterate in small steps

`SCAN` returns a small batch of keys plus a **cursor**. You pass the cursor back to get the next batch, and you
start and stop at cursor `0`:

```text
127.0.0.1:6379> SCAN 0
1) "18"
2)  1) "user:15:profile"
    2) "session:5"
    3) "session:7"
    4) "user:9:profile"
    5) "session:11"
    6) "session:14"
    7) "user:3:profile"
    8) "user:5:profile"
    9) "user:1:profile"
   10) "session:2"
```

The next call would be `SCAN 18`. When a call returns cursor `"0"`, the iteration is complete. Each call does a
bounded amount of work, so other clients get served between calls.

The options:

```text
127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 100
1) "0"
2)  1) "user:15:profile"
    2) "user:9:profile"
    ...
   21) "user:14:profile"
127.0.0.1:6379> SCAN 0 TYPE hash COUNT 100
1) "0"
2) 1) "user:1:prefs"
```

`COUNT` is a hint for how much of the table to examine per call (default 10), not how many results you get back.
`MATCH` and `TYPE` filter **after** the examining is done. With a pattern that matches little, you get pages that
are empty even though the iteration isn't finished:

```text
127.0.0.1:6379> SCAN 0 MATCH session:* COUNT 5
1) "40"
2) 1) "session:5"
   2) "session:7"
   3) "session:11"
127.0.0.1:6379> SCAN 0 MATCH nomatch:* COUNT 5
1) "40"
2) (empty array)
```

An empty page with a non-zero cursor means "keep going". A loop that stops at the first empty page is a classic
bug. **Only cursor 0 means done.**

What `SCAN` guarantees:

- A key that exists for the **whole** iteration will be returned **at least once**.
- A key might be returned **more than once** (de-duplicate if that matters).
- Keys added or removed during the iteration may or may not appear.

The same cursor interface exists inside collections: `HSCAN`, `SSCAN` and `ZSCAN` iterate one big hash, set or
sorted set without blocking.

## Deleting safely

### Big keys: DEL vs UNLINK

Freeing a value with millions of elements means freeing millions of allocations. `DEL` does that on the command
thread. `UNLINK` removes the key from the keyspace immediately and hands the freeing to a background thread.
This script builds a 2-million-field hash, deletes it each way, and pings from another connection the whole
time:

```python
import redis, time, threading
r = redis.Redis()

def build(key, n):
    p = r.pipeline(transaction=False)
    for i in range(n):
        p.hset(key, f"f{i}", "v" * 20)
        if i % 10000 == 0: p.execute()
    p.execute()

for cmd in ("DEL", "UNLINK"):
    build("big:hash", 2_000_000)
    worst = 0.0; stop = threading.Event()
    def pinger():
        global worst
        c = redis.Redis(); c.ping()
        while not stop.is_set():
            t = time.perf_counter(); c.ping()
            worst = max(worst, time.perf_counter() - t); time.sleep(0.0005)
    th = threading.Thread(target=pinger); th.start(); time.sleep(0.2); worst = 0.0
    t = time.perf_counter(); r.execute_command(cmd, "big:hash"); el = time.perf_counter() - t
    time.sleep(0.5); stop.set(); th.join()
    print(f"{cmd:6} returned after {el*1000:7.1f} ms; worst PING meanwhile {worst*1000:6.1f} ms")
```

```text
DEL    returned after   281.1 ms; worst PING meanwhile  280.2 ms
UNLINK returned after     0.5 ms; worst PING meanwhile    3.7 ms
```

`DEL` froze the server for 281 ms. `UNLINK` returned in half a millisecond and the server stayed responsive.
Afterwards `INFO memory` shows the background work:

```text
lazyfree_pending_objects:0
lazyfreed_objects:1
```

On this Redis 8.10 build, the `lazyfree-*` settings that would make `DEL`, eviction, expiry and `FLUSHALL`
behave like `UNLINK` automatically are all `no`:

```text
lazyfree-lazy-user-del      no
lazyfree-lazy-user-flush    no
lazyfree-lazy-expire        no
lazyfree-lazy-eviction      no
lazyfree-lazy-server-del    no
```

So use `UNLINK` explicitly, and `FLUSHALL ASYNC` / `FLUSHDB ASYNC` for whole databases, or turn those settings on
in your config. (Defaults differ between projects and versions. Level 4 · 09 compares Valkey's. Always check
with `CONFIG GET lazyfree*`.)

### Many keys: SCAN + UNLINK in batches

To delete every key matching a pattern without blocking, scan in pages and unlink in batches:

```python
def delete_matching(pattern, batch=500):
    deleted, buf = 0, []
    for key in r.scan_iter(match=pattern, count=1000):
        buf.append(key)
        if len(buf) >= batch:
            deleted += r.unlink(*buf); buf.clear()
    if buf:
        deleted += r.unlink(*buf)
    return deleted
```

On a database with 500,000 `user:*` and 500,000 `cache:*` keys:

```text
deleted 500000 keys in 0.81s; worst PING meanwhile 1.2 ms
remaining: 500000
```

It's slower overall than a single blocking command would be, but the worst latency any other client saw was
1.2 ms. That's the trade you want on a live server. `scan_iter` hides the cursor loop, and it correctly keeps
going through empty pages.

If you control the design, you can avoid mass deletes altogether. Give data that will need bulk removal a TTL,
use a version segment in the key name, or keep such data in its own logical structure (a hash per tenant,
for example) so that one `UNLINK` removes all of it.

## How It Actually Works

The keyspace is a hash table whose size is a power of two. A key's bucket is `hash(key) & (size - 1)`. When the
table fills up, Redis allocates a larger one and **rehashes incrementally**: each command and a periodic timer
move a few buckets from the old table to the new one, so growing a table with millions of keys never causes one
long pause. During a rehash, both tables are live.

`SCAN`'s cursor is a bucket position, but it isn't incremented normally. Redis increments the cursor's **reversed
bits**, so the scan visits buckets in an order where every bucket of a smaller table corresponds to a
contiguous group of buckets in a larger one. That's what lets `SCAN` keep its "every key present for the whole
time is returned" guarantee even when the table grows or shrinks mid-iteration. It's also why it sometimes
returns duplicates: after a shrink, two old buckets may map to one new bucket that gets visited again. The
cursor holds no server-side state at all. Redis doesn't remember your iteration, so abandoning one costs nothing,
and many clients can scan at once.

`COUNT` controls how many buckets one call visits, roughly. `MATCH` is applied to the keys in those buckets
afterwards, which is why a rare pattern gives empty pages.

`UNLINK` takes the value out of the dictionary (instantly, from the point of view of every client) and, if
freeing it would be expensive, puts the object on a queue for a background "bio" lazy-free thread. Freeing small
values in the background would cost more than it saves, so small ones are still freed inline. The
`lazyfree_pending_objects` metric is that queue's length.

## Common mistakes

- **`KEYS` in application code.** It's O(total keys) and blocks everyone. Use `SCAN`, or better, maintain an
  index set of the keys you'll need to find.
- **Stopping a `SCAN` loop on an empty page.** Only cursor `0` ends the iteration.
- **Assuming `SCAN` returns each key once.** De-duplicate when the operation isn't idempotent.
- **`DEL` on huge collections.** Use `UNLINK`, or turn on `lazyfree-lazy-user-del`.
- **`FLUSHALL` without `ASYNC`** on a big instance. It blocks for as long as freeing everything takes.
- **`RENAME` over an existing key** by accident. It silently overwrites. Use `RENAMENX` when that matters.
- **Mixing tenants or environments in numbered databases.** Use prefixes or separate instances.

## Exercise

1. Populate 200,000 keys and write a `SCAN` loop with `COUNT 10` and `MATCH user:1*`. Count how many calls return
   an empty page before the cursor reaches 0. Repeat with `COUNT 1000`.
2. Start a `SCAN` iteration, add 300,000 more keys halfway through (forcing a rehash), and finish it. Record how
   many keys came back and how many were duplicates.
3. Build a 1-million-member sorted set and compare `DEL` with `UNLINK` using the ping-latency script. Then set
   `lazyfree-lazy-user-del yes` and measure `DEL` again.
4. Design key names for a multi-tenant SaaS: tenants, users, sessions, a per-tenant rate limit and a cached
   report. Show the `SCAN` pattern that finds all keys of one tenant, and explain how you'd delete a tenant
   without scanning at all.
