---
description: "Redis hashes for objects: HSET, HGET and HINCRBY, per-field expiry with HEXPIRE and HSETEX, and why a small hash beats separate keys on memory."
---

# 05 · Hashes & Per-Field Expiry

A **hash** is a key whose value is a small map from field names to string values. It's the natural shape for an
object, such as a user profile, a product, a shopping cart or a feature-flag set. You can read or change one
field without touching the rest, and since Redis 7.4 each field can even have its own expiry.

## The basics

```text
127.0.0.1:6379> HSET user:1042 name "Ana Lima" email ana@example.com plan free logins 0
(integer) 4
127.0.0.1:6379> HGET user:1042 email
"ana@example.com"
127.0.0.1:6379> HMGET user:1042 name plan missing
1) "Ana Lima"
2) "free"
3) (nil)
127.0.0.1:6379> HGETALL user:1042
1) "name"
2) "Ana Lima"
3) "email"
4) "ana@example.com"
5) "plan"
6) "free"
7) "logins"
8) "0"
```

`HSET` takes any number of field/value pairs and returns how many **new** fields it created. The old `HMSET` is
just a deprecated alias. In RESP2, `HGETALL` returns a flat list of alternating fields and values (RESP3 returns
a real map; see Level 1 · 02). The field order you see is not something you can rely on.

Updating:

```text
127.0.0.1:6379> HINCRBY user:1042 logins 1
(integer) 1
127.0.0.1:6379> HSET user:1042 plan pro
(integer) 0
127.0.0.1:6379> HSETNX user:1042 plan enterprise
(integer) 0
127.0.0.1:6379> HSETNX user:1042 created 2026-10-13
(integer) 1
```

`HSET … plan pro` returned 0 because it updated an existing field rather than creating one. `HSETNX` writes a
field only if it's absent, which is handy for "first seen" timestamps. `HINCRBY` and `HINCRBYFLOAT` are the
per-field versions of the counters from Level 1 · 03, with the same atomicity.

Inspecting and removing:

```text
127.0.0.1:6379> HLEN user:1042
(integer) 5
127.0.0.1:6379> HEXISTS user:1042 phone
(integer) 0
127.0.0.1:6379> HDEL user:1042 created phone
(integer) 1
127.0.0.1:6379> HSTRLEN user:1042 email
(integer) 15
127.0.0.1:6379> HKEYS user:1042
1) "name"
2) "email"
3) "plan"
4) "logins"
127.0.0.1:6379> HRANDFIELD user:1042 2 WITHVALUES
1) "email"
2) "ana@example.com"
3) "logins"
4) "1"
```

When the last field is removed, the key itself disappears. Redis never keeps empty collections around.

## Expiry: the whole key or single fields

`EXPIRE` works on a hash like any other key, and changing fields doesn't reset it:

```text
127.0.0.1:6379> EXPIRE user:1042 60
(integer) 1
127.0.0.1:6379> HSET user:1042 plan team
(integer) 0
127.0.0.1:6379> TTL user:1042
(integer) 60
```

That differs from strings, where a plain `SET` clears the TTL, because `HSET` modifies the hash in place.

**Per-field expiry** arrived in Redis 7.4. `HEXPIRE` gives one or more fields a TTL in seconds (`HPEXPIRE` uses
milliseconds). `HTTL` reads it back, with the same `-1` for "no TTL" and `-2` for "no such field":

```text
127.0.0.1:6379> HSET cart:7 sku:pen 2 sku:ink 1 sku:pad 4
(integer) 3
127.0.0.1:6379> HEXPIRE cart:7 60 FIELDS 1 sku:ink
1) (integer) 1
127.0.0.1:6379> HTTL cart:7 FIELDS 3 sku:pen sku:ink sku:nope
1) (integer) -1
2) (integer) 60
3) (integer) -2
127.0.0.1:6379> HPERSIST cart:7 FIELDS 1 sku:ink
1) (integer) 1
127.0.0.1:6379> HTTL cart:7 FIELDS 1 sku:ink
1) (integer) -1
```

The `FIELDS <count>` part is required, and replies come back as an array with one result per field.

Redis 8.0 added commands that combine reading or writing with expiry, the same way `GETEX` and `GETDEL` do for
strings:

```text
127.0.0.1:6379> HSETEX otp:ana EX 300 FIELDS 1 code 482913
(integer) 1
127.0.0.1:6379> HTTL otp:ana FIELDS 1 code
1) (integer) 300
127.0.0.1:6379> HGETDEL otp:ana FIELDS 1 code
1) "482913"
127.0.0.1:6379> EXISTS otp:ana
(integer) 0
127.0.0.1:6379> HSET prefs:9 theme dark
(integer) 1
127.0.0.1:6379> HGETEX prefs:9 EX 3600 FIELDS 1 theme
1) "dark"
127.0.0.1:6379> HTTL prefs:9 FIELDS 1 theme
1) (integer) 3600
127.0.0.1:6379> TTL prefs:9
(integer) -1
```

`HGETDEL` reads a one-time code and removes it in one step, and because that was the only field, the key is
gone. `HGETEX` slides one field's expiry forward while the key as a whole stays persistent (`TTL` is `-1`).

When a field expires, it disappears from the hash. When the last field expires, the key goes with it:

```bash
redis-cli HSET c a 1 b 2
redis-cli HPEXPIRE c 1500 FIELDS 1 a
sleep 2; redis-cli HGETALL c
redis-cli HPEXPIRE c 500 FIELDS 1 b
sleep 1; redis-cli EXISTS c
```

```text
2
1
b
2
1
0
```

Where this helps: a hash of a user's active device tokens where each device expires on its own, a cart whose line
items hold a reservation for 15 minutes, or a per-user rate-limit hash with one field per window. Before 7.4,
each of those needed one key per field, or a cleanup job. On older servers and on other forks, check that the
commands exist before designing around them.

## Hash or separate keys or a JSON string?

There are three common ways to store a user with five attributes:

1. Five string keys: `user:42:name`, `user:42:email`, …
2. One string containing JSON.
3. One hash with five fields.

This script stores 100,000 users each way and measures `used_memory` per user:

```python
import redis, json
r = redis.Redis()
N = 100_000
fields = lambda i: {"name": f"User {i}", "email": f"user{i}@example.com",
                    "plan": "free", "logins": str(i % 50), "country": "PT"}

def measure(label, write):
    r.flushall(); base = r.info("memory")["used_memory"]
    p = r.pipeline(transaction=False)
    for i in range(N):
        write(p, i)
        if i % 5000 == 0: p.execute()
    p.execute()
    per = (r.info("memory")["used_memory"] - base) / N
    print(f"{label:34} {per:6.0f} bytes/user  keys={r.dbsize()}")

measure("5 string keys per user", lambda p, i: p.mset({f"user:{i}:{k}": v for k, v in fields(i).items()}))
measure("1 JSON string per user", lambda p, i: p.set(f"user:{i}", json.dumps(fields(i))))
measure("1 hash per user (listpack)", lambda p, i: p.hset(f"user:{i}", mapping=fields(i)))
r.config_set("hash-max-listpack-entries", 0)      # force the big-hash encoding
measure("1 hash per user (hashtable)", lambda p, i: p.hset(f"user:{i}", mapping=fields(i)))
r.config_set("hash-max-listpack-entries", 512)
```

```text
5 string keys per user                327 bytes/user  keys=500000
1 JSON string per user                160 bytes/user  keys=100000
1 hash per user (listpack)            143 bytes/user  keys=100000
1 hash per user (hashtable)           319 bytes/user  keys=100000
```

A small hash was the most compact option, smaller even than JSON (which repeats quotes and braces), and less
than half the size of five separate keys. The last row shows where that saving comes from: force the same hash
into its general-purpose encoding and it costs almost as much as separate keys.

Beyond memory, the trade-offs look like this:

| | Separate keys | JSON string | Hash |
|---|---|---|---|
| Read one attribute | `GET` | fetch and parse all of it | `HGET` |
| Update one attribute atomically | `SET` | needs a script or `WATCH` | `HSET` / `HINCRBY` |
| TTL granularity | per attribute | whole object | whole key, or per field (7.4+) |
| Nested data | no | yes | no (one level, values are strings) |

A hash is the default for flat objects. Use a JSON string, or the JSON type from Level 3 · 08, when the data
is nested and you always read it whole.

## How It Actually Works

A hash has two internal encodings:

```text
127.0.0.1:6379> OBJECT ENCODING user:1042
"listpack"
```

```bash
redis-cli CONFIG GET 'hash-max-listpack*'
```

```text
hash-max-listpack-entries
512
hash-max-listpack-value
64
```

While a hash has at most `hash-max-listpack-entries` fields and every field and value is at most
`hash-max-listpack-value` bytes, Redis stores it as a **listpack**: one contiguous block of memory with fields and
values laid out one after another, each with a tiny length header. There are no pointers, no per-entry
allocations and no hash table. Lookup is a linear scan, which is fast for a few hundred short entries that sit in
one or two CPU cache lines. Once either limit is exceeded, Redis converts the hash to a real **hash table**
(`hashtable` encoding). The conversion is one-way: deleting fields later doesn't shrink it back. On this 8.10
build the entries limit defaults to 512; older releases used 128, so check yours.

Per-field TTLs need extra metadata. As soon as a field gets an expiry, a small hash switches to a `listpackex`
encoding that stores expiry times alongside the entries:

```text
listpack      (before HPEXPIRE)
listpackex    (after)
```

Expired fields are removed the same two ways expired keys are: lazily, when the hash is accessed, and actively,
by the background expiry cycle, which tracks hashes that contain fields with TTLs.

The separate-keys layout costs more because every key pays for an entry in the global keyspace table, a key
string, a value object and allocator rounding. That's the ~100 bytes of overhead from Level 1 · 01, paid five
times per user instead of once.

## Common mistakes

- **One giant hash for everything** (`HSET users <id> <json>` for ten million users). You lose per-object
  expiry (unless you use field TTLs), the hash outgrows listpack, `HGETALL` becomes a blocking O(N) call, and
  in Cluster the whole thing lives on one node. Use one hash per object.
- **`HGETALL` on big hashes in hot paths.** Read the fields you need with `HMGET`, or iterate with `HSCAN`.
- **Expecting nested values.** Field values are strings. `HSET user:1 address "{…}"` stores JSON text you'll
  have to parse yourself.
- **Assuming `HSET` resets the key's TTL** the way `SET` does on strings. It doesn't, and a hash that should
  expire on every write needs an explicit `EXPIRE`.
- **Raising the listpack limits blindly.** Bigger listpacks save memory but make every access a longer linear
  scan. Measure with your real field counts.
- **Using `HEXPIRE` without the `FIELDS n` count**, or forgetting that it returns an array of per-field results.

## Exercise

1. Model a product with name, price in cents, stock and a "views" counter. Write the commands for: show the
   product, record a view, and reserve 2 units, where the last one must fail if stock would go below zero.
   Which of these can a plain hash command do atomically, and which needs something more (Level 2 · 08–09)?
2. Store a user's logged-in devices in one hash (`devices:<user>`, field = device id, value = last-seen time).
   Give each device a 30-day field TTL that's refreshed on every request with `HGETEX`. Demonstrate with 2-second
   TTLs that devices drop off independently.
3. Rerun the memory script with 20 fields per user, then 600 fields per user. Where does the hash stop being the
   smallest option, and why?
4. Use `OBJECT ENCODING` to show a hash converting from `listpack` to `hashtable` when one value exceeds
   64 bytes. Delete that field. Does the encoding change back?
