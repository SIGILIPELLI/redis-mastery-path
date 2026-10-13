---
description: "Redis sets for tags, membership and deduplication: SADD/SISMEMBER, intersections and unions, SINTERCARD, random picks, and the cost of algebra on big sets."
---

# 07 · Sets: Membership, Set Algebra & Random Picks

A **set** is an unordered collection of unique strings. Adding a member that's already there does nothing,
checking membership is O(1), and Redis can intersect, union and subtract sets on the server. That makes sets the
right tool for tags, "who liked this", deduplicating events, matching skills to jobs, and anywhere you'd
otherwise write "check if it's already in the list" by hand.

## Adding, checking, removing

```text
127.0.0.1:6379> SADD post:1:tags redis database cache
(integer) 3
127.0.0.1:6379> SADD post:1:tags redis nosql
(integer) 1
127.0.0.1:6379> SMEMBERS post:1:tags
1) "redis"
2) "database"
3) "cache"
4) "nosql"
127.0.0.1:6379> SISMEMBER post:1:tags cache
(integer) 1
127.0.0.1:6379> SMISMEMBER post:1:tags cache sql nosql
1) (integer) 1
2) (integer) 0
3) (integer) 1
127.0.0.1:6379> SCARD post:1:tags
(integer) 4
127.0.0.1:6379> SREM post:1:tags nosql
(integer) 1
```

`SADD` returns how many members were *new*: the second call added only `nosql` because `redis` was already
there. That return value is useful on its own. `SADD seen:emails <message-id>` returning 1 means "first time
we've seen this message, process it", and 0 means "duplicate, skip it". It's an atomic dedupe check in a single
command.

`SMISMEMBER` checks several candidates in one round trip. `SCARD` is O(1) because the count is stored, not
computed.

## Set algebra on the server

Two candidates and a job:

```text
127.0.0.1:6379> SADD user:ana:skills python redis sql docker
(integer) 4
127.0.0.1:6379> SADD user:ben:skills go redis kubernetes docker
(integer) 4
127.0.0.1:6379> SADD job:42:needs redis docker kubernetes
(integer) 3
```

What each candidate has in common with the job, and what Ana is missing:

```text
127.0.0.1:6379> SINTER user:ana:skills job:42:needs
1) "redis"
2) "docker"
127.0.0.1:6379> SINTER user:ben:skills job:42:needs
1) "redis"
2) "docker"
3) "kubernetes"
127.0.0.1:6379> SDIFF job:42:needs user:ana:skills
1) "kubernetes"
127.0.0.1:6379> SUNION user:ana:skills user:ben:skills
1) "redis"
2) "sql"
3) "python"
4) "go"
5) "docker"
6) "kubernetes"
```

`SDIFF A B` means "in A but not in B", so the order of the arguments matters. When you only need a **count**, use
`SINTERCARD` (Redis 7.0+), which doesn't build or send the members. Its `LIMIT` stops early once it has found
enough:

```text
127.0.0.1:6379> SINTERCARD 2 user:ben:skills job:42:needs
(integer) 3
127.0.0.1:6379> SINTERCARD 2 user:ben:skills job:42:needs LIMIT 2
(integer) 2
```

"Does this candidate match at least 2 of the requirements?" is a `LIMIT 2` question. The `*STORE` variants
write the result into a new key instead of returning it, which is useful for caching an expensive computation or
for chaining operations:

```text
127.0.0.1:6379> SUNIONSTORE team:skills user:ana:skills user:ben:skills
(integer) 6
```

## Random members

```text
127.0.0.1:6379> SADD raffle alice bob carol dave erin
(integer) 5
127.0.0.1:6379> SRANDMEMBER raffle 2
1) "alice"
2) "dave"
127.0.0.1:6379> SRANDMEMBER raffle -4
1) "bob"
2) "bob"
3) "dave"
4) "alice"
127.0.0.1:6379> SPOP raffle
"erin"
127.0.0.1:6379> SCARD raffle
(integer) 4
127.0.0.1:6379> SMOVE raffle winners bob
(integer) 1
```

- `SRANDMEMBER key n` with a **positive** n returns up to n *distinct* members and leaves the set alone.
- With a **negative** n it returns exactly |n| members and *may repeat* (`bob` twice above), like sampling with
  replacement.
- `SPOP` removes and returns random members. It's the atomic "draw a winner", and two clients can never draw
  the same one.
- `SMOVE` moves a member from one set to another atomically.

Randomness here is good enough for picking samples or shuffling work. It isn't a cryptographic or audited
lottery.

## How expensive is set algebra?

Two sets of a million visitor IDs each (half overlapping) and a 4-member `vip` set, timed from Python:

```python
import redis, time
r = redis.Redis()
p = r.pipeline(transaction=False)
for i in range(0, 1_000_000, 5000):
    p.sadd("visitors:mon", *range(i, i + 5000))
    p.sadd("visitors:tue", *range(i + 500_000, i + 505_000))
p.execute()
r.sadd("vip", 7, 123456, 750000, 2_000_000)

def t(label, fn):
    s = time.perf_counter(); res = fn()
    print(f"{label:46} {(time.perf_counter()-s)*1000:8.2f} ms -> {res}")

t("SINTER vip visitors:mon visitors:tue (size)", lambda: len(r.sinter("vip", "visitors:mon", "visitors:tue")))
t("SINTER visitors:mon visitors:tue vip (size)", lambda: len(r.sinter("visitors:mon", "visitors:tue", "vip")))
t("SINTERCARD 2 visitors:mon visitors:tue", lambda: r.sintercard(2, ["visitors:mon", "visitors:tue"]))
t("SINTERSTORE both visitors:mon visitors:tue", lambda: r.sinterstore("both", ["visitors:mon", "visitors:tue"]))
t("SUNIONSTORE either visitors:mon visitors:tue", lambda: r.sunionstore("either", ["visitors:mon", "visitors:tue"]))
t("SISMEMBER visitors:mon 999999", lambda: r.sismember("visitors:mon", 999999))
```

```text
SINTER vip visitors:mon visitors:tue (size)        0.23 ms -> 1
SINTER visitors:mon visitors:tue vip (size)        0.21 ms -> 1
SINTERCARD 2 visitors:mon visitors:tue           131.70 ms -> 500000
SINTERSTORE both visitors:mon visitors:tue       232.29 ms -> 500000
SUNIONSTORE either visitors:mon visitors:tue     337.91 ms -> 1500000
SISMEMBER visitors:mon 999999                      0.33 ms -> 1
```

- An intersection that includes a **small** set is fast regardless of argument order (0.2 ms both ways): Redis
  sorts the inputs by size and iterates the smallest one, probing the others.
- Intersecting two **large** sets costs time proportional to the smaller one. That's 132 ms just to *count*
  500,000 common members, and the server was busy for all of it.
- A union has to touch every member of every input. Storing 1.5 million members took 338 ms.

Those numbers are fine for an analytics job at 3 a.m. and a problem on a hot request path. For big sets, run the
algebra rarely and cache the result with `*STORE` + `EXPIRE`, keep the sets smaller (per day rather than per
year), or switch to an approximate structure. HyperLogLog (Level 2 · 03) counts unique visitors in 12 KB instead
of the 32 MB that one of these sets occupies:

```text
memory visitors:mon 32277585 bytes
```

## How It Actually Works

Sets have three encodings:

```text
127.0.0.1:6379> SADD ids 1 2 3 1000
(integer) 4
127.0.0.1:6379> OBJECT ENCODING ids
"intset"
127.0.0.1:6379> SADD ids hello
(integer) 1
127.0.0.1:6379> OBJECT ENCODING ids
"listpack"
127.0.0.1:6379> OBJECT ENCODING user:ana:skills
"listpack"
```

The limits, from `CONFIG GET 'set-max*'` (shown as name and value pairs):

```text
set-max-listpack-entries   128
set-max-listpack-value     64
set-max-intset-entries     512
```

- **intset** — when every member is an integer and there are at most 512 of them, the set is a *sorted array of
  integers* using the smallest width that fits (16, 32 or 64 bits). Membership is a binary search. It's
  extremely compact: 500 integers took **1,032 bytes**.
- **listpack** — small sets of short strings (at most 128 members of at most 64 bytes each) are a flat listpack
  with linear search, as with small hashes.
- **hashtable** — anything bigger is a hash table whose keys are the members and whose values are empty. That's
  O(1) membership, but every member pays for a table entry and its own string: 500 short strings took 15,567
  bytes, fifteen times the intset.

Conversions go one way, towards the general encoding, as soon as a limit is crossed. Adding `hello` to an intset
converted it on the spot.

`SINTER` sorts its inputs by cardinality, then for each member of the smallest set checks membership in each of
the others, stopping at the first miss. That's why its cost depends on the smallest set. `SUNION` and `SDIFF`
have to walk all of the inputs. `SRANDMEMBER` with a positive count picks distinct members with a strategy that
depends on how large the count is relative to the set. For small counts it picks random buckets, so the
distribution is close to uniform but not exactly uniform.

## Common mistakes

- **`SMEMBERS` on a huge set.** It's O(N) and sends everything. Use `SSCAN` to iterate, `SCARD` to count,
  `SRANDMEMBER` to sample.
- **Set algebra on million-member sets in request handlers.** Precompute with `*STORE` and cache it, or use
  smaller per-period sets.
- **Using a set when order matters.** Sets are unordered. If you need "most recent" or "top", use a sorted set
  (next lesson).
- **Expecting `SDIFF` to be symmetric.** `SDIFF A B` ≠ `SDIFF B A`.
- **Counting unique events forever in a set.** It grows without bound. Use a TTL per period, or HyperLogLog when
  an approximate count is enough.
- **Storing IDs as strings like `"id42"`** when plain integers would let small sets use the intset encoding.

## Exercise

1. Build a tag system: posts have tag sets, and each tag has a set of post IDs. Write commands to add a tag to a
   post (both directions), find posts tagged both `redis` and `python`, and find posts tagged `redis` but not
   `beginner`. Why should the two-direction update run inside `MULTI`?
2. Use `SADD` as a dedupe filter for a stream of 100,000 event IDs where 10% are repeats. Process each event only
   when `SADD` returns 1, and confirm the processed count. How would you stop the dedupe set from growing
   forever?
3. Make the timing script's `vip` set 100,000 members and rerun the first `SINTER`. How does the time scale with
   the smallest set?
4. Find the point where a set of integers switches from `intset` to `hashtable` on your server, and record
   `MEMORY USAGE` just before and just after.
