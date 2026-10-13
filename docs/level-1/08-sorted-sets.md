---
description: "Redis sorted sets for leaderboards, time windows and delayed jobs: ZADD flags, ranks, score and lex ranges, unions, and O(log N) costs measured."
---

# 08 · Sorted Sets: Rankings, Ranges & Time Windows

A **sorted set** (zset) is a set where every member has a numeric **score**, and members are kept ordered by
score. You can ask "what's Ana's rank?", "who are the top 10?", "which members score between X and Y?" and "remove
everything older than five minutes", and each of these is a logarithmic-time operation on a structure with
millions of members. If you learn one Redis type beyond strings and hashes, make it this one. Leaderboards,
sliding-window rate limiters, delayed job schedulers, time-ordered indexes and priority queues are all sorted sets
underneath.

## Scores and order

```text
127.0.0.1:6379> ZADD lb:week41 1200 ana 980 ben 1500 caro 980 dan
(integer) 4
127.0.0.1:6379> ZRANGE lb:week41 0 -1 WITHSCORES
1) "ben"
2) "980"
3) "dan"
4) "980"
5) "ana"
6) "1200"
7) "caro"
8) "1500"
```

`ZADD` takes **score first, then member**. That's the opposite of most mental models, so expect to get it
backwards once. Order is ascending by score, and ties (`ben` and `dan` at 980) are broken by comparing the member
strings byte by byte. That tie-break is deterministic but probably not what your game wants ("whoever got there
first"). There's a fix in the exercises.

For a leaderboard you want the top first:

```text
127.0.0.1:6379> ZRANGE lb:week41 0 2 REV WITHSCORES
1) "caro"
2) "1500"
3) "ana"
4) "1200"
5) "dan"
6) "980"
```

Since Redis 6.2, `ZRANGE` with `REV`, `BYSCORE`, `BYLEX` and `LIMIT` replaces the older family of
`ZREVRANGE`, `ZRANGEBYSCORE` and so on. Those still work, but you only need to learn one command.

## Changing scores, reading ranks

```text
127.0.0.1:6379> ZINCRBY lb:week41 50 ben
"1030"
127.0.0.1:6379> ZSCORE lb:week41 ben
"1030"
127.0.0.1:6379> ZMSCORE lb:week41 ana nobody
1) "1200"
2) (nil)
127.0.0.1:6379> ZREVRANK lb:week41 ana
(integer) 1
127.0.0.1:6379> ZREVRANK lb:week41 ana WITHSCORE
1) (integer) 1
2) "1200"
127.0.0.1:6379> ZCARD lb:week41
(integer) 4
```

Ranks are **0-based**: Ana is second place, rank 1. `ZRANK` counts from the lowest score, `ZREVRANK` from the
highest. The `WITHSCORE` option (7.2+) returns rank and score together, which is exactly what a "your position"
widget needs, in one round trip.

### ZADD's flags

`ZADD` has flags that turn it into conditional updates:

```text
127.0.0.1:6379> ZADD lb:week41 GT 1100 ana
(integer) 0
127.0.0.1:6379> ZADD lb:week41 GT 1300 ana
(integer) 0
127.0.0.1:6379> ZADD lb:week41 GT CH 1400 ana
(integer) 1
127.0.0.1:6379> ZADD lb:week41 NX 9999 ana
(integer) 0
127.0.0.1:6379> ZADD lb:week41 LT 1 eve
(integer) 1
127.0.0.1:6379> ZADD lb:week41 INCR 10 eve
"11"
```

Read those return values carefully:

- `GT 1100` didn't change anything, because 1100 isn't greater than Ana's 1200.
- `GT 1300` **did** update Ana to 1300, yet it returned 0. By default `ZADD` returns the number of members
  *added*, not changed. Add `CH` to count changes too, as the third line shows.
- `NX` only adds new members and never updates existing ones. `XX` only updates.
- `GT` and `LT` don't stop *new* members from being added: `eve` didn't exist, so `LT 1` added her.
- `INCR` makes `ZADD` behave like `ZINCRBY` and return the new score.

`ZADD … GT` is the "personal best" operation. Submit every game score and the set keeps each player's maximum,
atomically, with no read first.

## Range queries

By score, with `(` meaning exclusive and `-inf`/`+inf` meaning unbounded:

```text
127.0.0.1:6379> ZCOUNT lb:week41 1000 +inf
(integer) 3
127.0.0.1:6379> ZRANGE lb:week41 1000 1500 BYSCORE WITHSCORES
1) "ben"
2) "1030"
3) "ana"
4) "1200"
5) "caro"
6) "1500"
127.0.0.1:6379> ZRANGE lb:week41 (1200 +inf BYSCORE
1) "caro"
127.0.0.1:6379> ZRANGE lb:week41 +inf -inf BYSCORE REV LIMIT 0 2
1) "caro"
2) "ana"
```

With `REV` the range bounds are given high-to-low (`+inf -inf`). `LIMIT offset count` pages through a range.

By member name, when every score is equal (usually 0), the order is lexicographic, and `BYLEX` gives you prefix
search:

```text
127.0.0.1:6379> ZADD names 0 banana 0 apple 0 cherry 0 apricot
(integer) 4
127.0.0.1:6379> ZRANGE names [ap (b BYLEX
1) "apple"
2) "apricot"
```

`[` means inclusive and `(` means exclusive. `[ap (b` means "from `ap` up to but not including `b`", i.e. every
member starting with `ap`. That's a simple autocomplete index.

## Time as the score

Scores are 64-bit floats, and a Unix timestamp fits comfortably. Using time as the score turns a sorted set into a
time index.

**Delayed jobs.** Schedule a job for later by scoring it with its due time. A poller grabs whatever is due:

```text
127.0.0.1:6379> ZADD jobs:delayed 1791856000 send-email:1 1791855000 resize-img:7 1791859999 report:3
(integer) 3
127.0.0.1:6379> ZRANGE jobs:delayed -inf 1791856500 BYSCORE
1) "resize-img:7"
2) "send-email:1"
127.0.0.1:6379> ZPOPMIN jobs:delayed
1) "resize-img:7"
2) "1791855000"
```

`ZPOPMIN` atomically removes the lowest-scored member, so two pollers can't both take the same job. (A poller
should check that the popped score is actually due and put it back if not. A Lua script does both in one step,
as Level 2 · 09 shows.) `BZPOPMIN` blocks like `BLPOP` does.

**Sliding windows.** "How many requests did this IP make in the last 60 seconds?" Add each event scored by its
time, drop everything older than the window, and count:

```text
ZADD   rate:ip:203.0.113.7  <now_ms> <unique-request-id>
ZREMRANGEBYSCORE rate:ip:203.0.113.7 -inf <now_ms - 60000>
ZCARD  rate:ip:203.0.113.7
```

With three requests at 0 s, 30 s and 55 s and "now" at 70 s, the window starts at 10 s:

```text
127.0.0.1:6379> ZADD rate:ip:203.0.113.7 1791856000000 req-a 1791856030000 req-b 1791856055000 req-c
(integer) 3
127.0.0.1:6379> ZREMRANGEBYSCORE rate:ip:203.0.113.7 -inf (1791856010000
(integer) 1
127.0.0.1:6379> ZCARD rate:ip:203.0.113.7
(integer) 2
```

The request at 0 s fell out of the window, and two remain.

Level 3 · 04 builds a correct rate limiter from this pattern and compares it with cheaper approximations.

## Combining sorted sets

Monthly totals from weekly boards, best single week, and players present in both weeks:

```text
127.0.0.1:6379> ZADD lb:week42 300 ana 900 fred
(integer) 2
127.0.0.1:6379> ZUNIONSTORE lb:month 2 lb:week41 lb:week42 AGGREGATE SUM
(integer) 6
127.0.0.1:6379> ZRANGE lb:month 0 -1 REV WITHSCORES
 1) "ana"
 2) "1700"
 3) "caro"
 4) "1500"
 5) "ben"
 6) "1030"
 7) "dan"
 8) "980"
 9) "fred"
10) "900"
11) "eve"
12) "11"
127.0.0.1:6379> ZUNIONSTORE lb:best 2 lb:week41 lb:week42 AGGREGATE MAX
(integer) 6
127.0.0.1:6379> ZRANGE lb:best 0 0 REV WITHSCORES
1) "caro"
2) "1500"
127.0.0.1:6379> ZINTERSTORE lb:both 2 lb:week41 lb:week42 WEIGHTS 1 2
(integer) 1
127.0.0.1:6379> ZRANGE lb:both 0 -1 WITHSCORES
1) "ana"
2) "2000"
```

`WEIGHTS` multiplies each input's scores before aggregating: Ana's 1400 × 1 + 300 × 2 = 2000. `AGGREGATE` can be
`SUM` (the default), `MIN` or `MAX`. `ZRANGESTORE` saves a slice, such as a cached top 3:

```text
127.0.0.1:6379> ZRANGESTORE lb:top3 lb:month 0 2 REV
(integer) 3
127.0.0.1:6379> ZRANGE lb:top3 0 -1 WITHSCORES
1) "ben"
2) "1030"
3) "caro"
4) "1500"
5) "ana"
6) "1700"
```

The stored set is still ordered by score ascending, so read it back with `REV` for top-first.

## What it costs at a million members

```python
import redis, time, random
r = redis.Redis()
random.seed(7)
p = r.pipeline(transaction=False)
for i in range(0, 1_000_000, 5000):
    p.zadd("lb:global", {f"player:{j}": random.randint(0, 1_000_000) for j in range(i, i + 5000)})
p.execute()
print("ZCARD", r.zcard("lb:global"), "encoding", r.object("encoding", "lb:global"),
      "memory", r.memory_usage("lb:global"))

def t(label, fn, n=500):
    s = time.perf_counter()
    for _ in range(n): res = fn()
    print(f"{label:40} {(time.perf_counter()-s)/n*1e6:8.1f} us  -> {str(res)[:60]}")

t("ZINCRBY (random player)", lambda: r.zincrby("lb:global", 5, f"player:{random.randrange(1_000_000)}"))
t("ZREVRANK player:4242", lambda: r.zrevrank("lb:global", "player:4242"))
t("ZRANGE top 10 REV", lambda: len(r.zrange("lb:global", 0, 9, desc=True)))
t("ZRANGE ranks 500000..500009", lambda: len(r.zrange("lb:global", 500000, 500009)))
t("ZCOUNT 400000..600000", lambda: r.zcount("lb:global", 400000, 600000))
t("ZRANGE BYSCORE 400000..600000 (all)", lambda: len(r.zrangebyscore("lb:global", 400000, 600000)), n=5)
t("echo baseline PING", lambda: r.ping())
```

```text
ZCARD 1000000 encoding b'skiplist' memory 77722996
ZINCRBY (random player)                     137.5 us  -> 74390.0
ZREVRANK player:4242                        120.0 us  -> 954015
ZRANGE top 10 REV                           124.8 us  -> 10
ZRANGE ranks 500000..500009                 112.1 us  -> 10
ZCOUNT 400000..600000                       117.9 us  -> 200168
ZRANGE BYSCORE 400000..600000 (all)       27322.5 us  -> 200168
echo baseline PING                          110.3 us  -> True
```

Updating a score, getting a rank, reading ten entries from the middle and **counting** 200,000 members in a score
range all cost about the same as a `PING`, so the server work is a few microseconds. *Returning* those 200,000
members took 27 ms, because that work is proportional to the output size. The rule for sorted sets: every lookup is
O(log N), plus O(M) for the M members you return. Keep M small with `LIMIT` and paging.

The million-member set used 77.7 MB, about 78 bytes per member. That's more than a plain set, because each
member is in two structures at once (see below).

## How It Actually Works

Small sorted sets (up to `zset-max-listpack-entries`, 128, members of up to 64 bytes) are a **listpack** of
alternating member/score entries kept in score order, as `lb:month` showed. Inserting is a linear scan and a
memmove, which is fast at that size and very compact.

Large sorted sets use **two structures together**:

- a **hash table** from member to score, which makes `ZSCORE` and "does this member exist?" O(1);
- a **skip list** ordered by (score, member). A skip list is a linked list where each node also has a random
  number of "express lane" pointers that skip ahead. Searching starts in the top lane and drops a level whenever
  the next hop would overshoot, which takes O(log N) steps on average.

Redis's skip list stores a **span** on every forward pointer, meaning how many nodes that pointer skips. Summing
the spans along a search path gives a node's rank, which is how `ZRANK` is O(log N) rather than a count from the
start, and how `ZRANGE 500000 500009` jumps straight to rank 500,000. `ZCOUNT` finds the first and last node of the
range and subtracts their ranks, so it never visits the 200,000 members in between.

`ZINCRBY` looks up the old score in the hash table, removes the node from the skip list, and reinserts it at its
new position. If the new score keeps it in the same place, the node is updated in place.

Scores are IEEE-754 doubles, so integers are exact only up to 2^53. Millisecond timestamps are fine, but packing
a big score and a timestamp into one number for tie-breaking can silently lose precision. Do the arithmetic
before you rely on it.

## Common mistakes

- **Swapping score and member** in `ZADD`. It's `ZADD key score member`.
- **Reading `ZADD`'s return value as "changed".** Without `CH` it only counts additions.
- **Unbounded range reads** (`ZRANGE key 0 -1` on a million-member set). Use `LIMIT`, or count with `ZCOUNT`.
- **Ties resolved alphabetically** when your product expects "first come first served". Encode a tie-breaker
  into the score, or keep a secondary structure.
- **Never trimming time-scored sets.** Event windows need `ZREMRANGEBYSCORE` (or `ZREMRANGEBYRANK`) on every write.
- **Using float scores for money or exact counts** beyond 2^53.
- **Forgetting that `ZRANGESTORE` output is ascending** and then reading it back the wrong way round.

## Exercise

1. Build a game leaderboard where equal scores are ordered by who reached that score **first**. Hint: score =
   points × 10^10 + (10^10 − seconds since season start). Check the arithmetic stays below 2^53 for your
   maximum points.
2. Implement "players near me": given a player, return the 5 above and 5 below them with ranks and scores, in two
   commands at most.
3. Write a delayed-job poller with `ZRANGE … BYSCORE LIMIT 0 10` followed by `ZREM` per job. Run two pollers at
   once and show that a job can be processed twice. Then fix it using `ZREM`'s return value as the claim.
4. Use a `BYLEX` sorted set to autocomplete 10,000 city names. Return the first 5 matches for a prefix, and make
   matching case-insensitive without losing the original spelling.
