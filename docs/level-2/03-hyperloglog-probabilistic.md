---
description: "Approximate counting and membership in Redis: HyperLogLog in 12 KB, Bloom and Cuckoo filters, Count-Min and Top-K, with error rates measured."
---

# 03 · HyperLogLog, Bloom Filters & Other Sketches

Some questions don't need exact answers. "About how many unique visitors did this page get?" "Have we
*probably* seen this email before?" "What are roughly the five most-viewed pages?" Exact answers mean keeping every
item, like the 125 MB set of user IDs in the last lesson. **Probabilistic data structures** (sketches) keep a small
summary instead and answer with a known, bounded error. Trading a little accuracy for orders of magnitude less
memory is often the best deal in the system.

HyperLogLog is built into Redis core. The others (Bloom, Cuckoo, Count-Min Sketch, Top-K, t-digest) come from the
probabilistic module that ships with Redis 8. Load it with the lab config from Level 1 · 01, or every `BF.*`
command fails as unknown.

## HyperLogLog: counting distinct things

```text
127.0.0.1:6379> PFADD visitors:home alice bob carol alice
(integer) 1
127.0.0.1:6379> PFADD visitors:home alice
(integer) 0
127.0.0.1:6379> PFCOUNT visitors:home
(integer) 3
127.0.0.1:6379> PFADD visitors:about bob dave
(integer) 1
127.0.0.1:6379> PFCOUNT visitors:home visitors:about
(integer) 4
127.0.0.1:6379> PFMERGE visitors:site visitors:home visitors:about
OK
127.0.0.1:6379> PFCOUNT visitors:site
(integer) 4
```

`PFADD` returns 1 if the internal estimate changed, which roughly means "probably new", and 0 otherwise.
`PFCOUNT` over several keys counts the **union** without changing anything, and `PFMERGE` stores the union.
Duplicates across pages (`bob`) are counted once. You can't list the members, check whether a specific one is
present, or remove one. A HyperLogLog only counts.

(The `PF` prefix honours Philippe Flajolet, who co-invented the algorithm.)

### How accurate, and how big?

Add 10 million distinct users and compare the estimate with the truth at each power of ten:

```python
import redis
r = redis.Redis()
checkpoints = {1_000, 10_000, 100_000, 1_000_000, 10_000_000}
p = r.pipeline(transaction=False); batch = []
for i in range(1, 10_000_001):
    batch.append(f"user-{i}")
    if len(batch) == 5000:
        p.pfadd("hll", *batch); batch = []; p.execute()
    if i in checkpoints:
        if batch: r.pfadd("hll", *batch); batch = []
        est = r.pfcount("hll")
        print(f"true {i:>10,}  estimate {est:>10,}  error {100*(est-i)/i:+6.2f}%  bytes {r.strlen('hll'):>6}")
```

```text
true      1,000  estimate      1,004  error  +0.40%  bytes   1902
true     10,000  estimate      9,959  error  -0.41%  bytes  12304
true    100,000  estimate    100,165  error  +0.17%  bytes  12304
true  1,000,000  estimate  1,003,993  error  +0.40%  bytes  12304
true 10,000,000  estimate 10,015,838  error  +0.16%  bytes  12304
```

The key **never grew beyond 12,304 bytes**, whether it counted ten thousand or ten million. One run is just one
sample of the error, though. Forty independent HyperLogLogs of 100,000 items each:

```text
40 trials of 100,000 distinct items: mean error +0.03%, std dev 0.70%, worst +1.72%
```

Redis documents a standard error of **0.81%**, and the observed spread (0.70%) is consistent with that. Errors of
1–2% happen in some trials, and the estimate is unbiased on average. For "unique visitors per page per day" on a
dashboard, that's excellent. For billing, it isn't acceptable.

At 1,000 items the key was only 1,902 bytes. Small HyperLogLogs use a **sparse** representation and switch to
the fixed 12 KB **dense** one as they fill up. A HyperLogLog is a string (`TYPE` says `string`), so it replicates,
persists and expires like any other key.

## Bloom filters: "have I seen this before?"

A Bloom filter answers set-membership questions with **no false negatives** and a tunable rate of false
positives. If it says "no", the item was definitely never added. If it says "yes", it probably was.

Reserve a filter for 1 million usernames at a 1% false-positive rate, add them, and test 100,000 names that were
never added:

```python
import redis
r = redis.Redis(decode_responses=True)
print(r.execute_command("BF.RESERVE", "bf:usernames", 0.01, 1_000_000))
names = [f"user{i}" for i in range(1_000_000)]
for i in range(0, len(names), 10000):
    r.execute_command("BF.MADD", "bf:usernames", *names[i:i+10000])
    r.sadd("set:usernames", *names[i:i+10000])
print("BF.EXISTS user42:", r.execute_command("BF.EXISTS", "bf:usernames", "user42"))
probe = [f"other{i}" for i in range(100_000)]
fp = sum(sum(r.execute_command("BF.MEXISTS", "bf:usernames", *probe[i:i+10000]))
         for i in range(0, len(probe), 10000))
print(f"false positives: {fp} of {len(probe)} = {100*fp/len(probe):.2f}%")
print(r.execute_command("BF.INFO", "bf:usernames"))
print("memory: bloom", r.memory_usage("bf:usernames"), " set", r.memory_usage("set:usernames"))
```

```text
OK
BF.EXISTS user42: True
false positives: 516 of 100000 = 0.52%
{'Capacity': 1000000, 'Size': 1378568, 'Number of filters': 1, 'Number of items inserted': 999258, 'Expansion rate': 2}
memory: bloom 1378600  set 36277586
```

- The measured false-positive rate was **0.52%**, inside the 1% target. The target is an upper bound at full
  capacity, and filters do better below capacity.
- **1.4 MB versus 36 MB** for an exact set of the same names.
- "Number of items inserted" is **999,258**, not 1,000,000. When `BF.MADD` adds an item whose bits are all set
  already (a false positive at insert time), it can't tell that apart from a duplicate, so it doesn't count it. The
  filter still answers "yes" for those items.

Typical uses are checking "is this username taken?" before a database lookup (most names aren't, and a "no" avoids
the query), skipping URLs a crawler has already fetched, and filtering already-recommended items.

### Overfilling

Add another million items to the 1-million-capacity filter:

```text
after 2x capacity: {'Capacity': 3000000, 'Size': 4496248, 'Number of filters': 2, 'Number of items inserted': 1994225, 'Expansion rate': 2}
false positives after scaling: 521 of 100000 = 0.52%
```

Redis's Bloom filter is **scalable** by default. When the first filter filled up, it added a second one twice as
large (expansion rate 2), keeping the error rate in check at the cost of memory (now 4.5 MB) and a lookup across two
filters. Create a filter with `NONSCALING` to make it fail instead of growing, if you'd rather find out that your
capacity estimate was wrong.

### Deleting: Bloom can't, Cuckoo can

```text
BF.DEL: unknown command 'BF.DEL', with args beginning with: 'bf:usernames' 'user1'
```

A Bloom filter has no delete, because clearing an item's bits could clear bits shared with other items and create
false negatives. A **Cuckoo filter** stores small fingerprints in buckets instead of bits, and it supports deletion:

```python
print(r.execute_command("CF.RESERVE", "cf:sessions", 1000))
print(r.execute_command("CF.ADD", "cf:sessions", "s1"), r.execute_command("CF.EXISTS", "cf:sessions", "s1"),
      r.execute_command("CF.DEL", "cf:sessions", "s1"), r.execute_command("CF.EXISTS", "cf:sessions", "s1"))
```

```text
OK
True True True False
```

Only delete items you know were added. Deleting a never-added item that happens to share a fingerprint removes
someone else's entry.

## Count-Min Sketch and Top-K: approximate frequencies

Here are 500,000 page views with a long-tail distribution (a few pages get most of the traffic, 633 distinct pages
in all), counted with a Count-Min Sketch and tracked with a Top-K structure:

```python
import redis, random, collections
r = redis.Redis(decode_responses=True)
random.seed(3)
pages = [f"/page-{min(int(random.paretovariate(1.2)), 5000)}" for _ in range(500_000)]
truth = collections.Counter(pages)

r.execute_command("CMS.INITBYPROB", "cms:pages", 0.001, 0.01)   # error 0.1% of total, 1% failure prob
r.execute_command("TOPK.RESERVE", "topk:pages", 5)
for i in range(0, len(pages), 5000):
    chunk = pages[i:i+5000]
    r.execute_command("CMS.INCRBY", "cms:pages", *[x for p in chunk for x in (p, 1)])
    r.execute_command("TOPK.ADD", "topk:pages", *chunk)
for page in ["/page-1", "/page-2", "/page-50", "/page-4000"]:
    est = r.execute_command("CMS.QUERY", "cms:pages", page)[0]
    print(f"{page:11} true {truth[page]:>7}  CMS {est:>7}  over by {est-truth[page]}")
print("TOPK.LIST:", r.execute_command("TOPK.LIST", "topk:pages", "WITHCOUNT"))
print("true top5:", truth.most_common(5))
print("memory: cms", r.memory_usage("cms:pages"), " topk", r.memory_usage("topk:pages"))
```

```text
/page-1     true  282210  CMS  282210  over by 0
/page-2     true   83855  CMS   83855  over by 0
/page-50    true     122  CMS     122  over by 0
/page-4000  true       0  CMS       0  over by 0
TOPK.LIST: ['/page-1', 282210, '/page-2', 83855, '/page-3', 39099, '/page-4', 22201, '/page-5', 14498]
true top5: [('/page-1', 282210), ('/page-2', 83855), ('/page-3', 39101), ('/page-4', 22201), ('/page-5', 14498)]
memory: cms 56064  topk 2688
```

With only 633 distinct pages and a sketch sized for much more, the counts came out exact. Top-K found the right
five pages, with `/page-3` undercounted by 2. To see the error a Count-Min Sketch *can* make, shrink it to 100
counters × 3 rows (`CMS.INITBYDIM cms:small 100 3`, same data):

```text
distinct pages: 633
/page-1     true  282210  CMS(100x3)  282222  over by 12
/page-2     true   83855  CMS(100x3)   84268  over by 413
/page-50    true     122  CMS(100x3)     130  over by 8
/page-4000  true       0  CMS(100x3)     121  over by 121
memory 1264
```

A Count-Min Sketch **only overestimates**. Every page shares counters with others, and a never-seen page reports
121 views because all three of its counters were bumped by other pages. The sizing parameters let you bound the
overestimate as a fraction of total traffic. Relative to that bound, big counts are accurate and small counts are
noise.

The module also has **t-digest** (`TDIGEST.*`), which estimates percentiles such as p50 or p99 latency from a
stream of values in bounded memory. It wasn't exercised in this course.

## Choosing

| Question | Structure | Memory here | Error |
|---|---|---|---|
| How many distinct? | HyperLogLog | 12 KB at 10M items | ~0.81% std error, no listing |
| Seen before? (append-only) | Bloom filter | 1.4 MB per 1M items at 1% | false positives only |
| Seen before? (with deletes) | Cuckoo filter | similar | false positives only |
| How often has X occurred? | Count-Min Sketch | 56 KB here | overestimates only |
| What are the heaviest hitters? | Top-K | 2.7 KB for top 5 | may miss near-ties |
| Exact anything | set / hash / sorted set | 36–125 MB here | none |

## How It Actually Works

**HyperLogLog** hashes each item to 64 bits. It uses 14 bits to choose one of 16,384 registers and looks at the
remaining 50 bits for the position of the first 1 bit, which is like counting how many coin flips came up tails
before the first head. Each register remembers the longest run it has seen. Long runs are rare, so the longest run
seen tells you roughly how many distinct hashes went in (a run of 20 suggests about a million). Averaging across
16,384 registers with a harmonic mean, plus bias corrections, gives the estimate and its 1.04/√16384 ≈ 0.81%
standard error. Each register is 6 bits: 16,384 × 6 bits = 12,288 bytes, plus a 16-byte header, which is exactly
the 12,304 bytes measured. Merging two HyperLogLogs takes the register-wise maximum, which is why unions are cheap
and exact (relative to the sketches) while intersections aren't supported.

A **Bloom filter** is a bit array plus k hash functions. Adding an item sets k bits, and testing checks whether all
k are set. With m bits and n items, the false-positive rate is about (1 − e^(−kn/m))^k. Redis's `BF.RESERVE`
derives m and k from your error rate and capacity. The textbook minimum for 1% is about 9.6 bits per item; this
filter used 1,378,568 bytes for 1M items, about 11 bits per item, a little above that minimum.

A **Count-Min Sketch** is d rows of w counters, each row with its own hash. Incrementing adds to one counter per
row, and querying returns the minimum across rows. The minimum is the least-polluted estimate, but it can never be
below the truth, so the sketch only overestimates. Width bounds the error as a fraction of the total count, and depth
bounds the probability of exceeding it.

## Common mistakes

- **Using HyperLogLog where exact counts matter** (billing, quotas, inventory).
- **Expecting to check membership in a HyperLogLog.** It can only count.
- **Assuming Bloom answers are exact.** A "yes" must still be confirmed against the source of truth when it matters.
- **Undersizing Bloom filters** and silently scaling into many sub-filters. Size from real numbers, or use
  `NONSCALING` and alert on errors.
- **Deleting from a Cuckoo filter items that were never added.**
- **Reading small Count-Min counts as meaningful.** They're dominated by collisions.
- **Forgetting the module.** `BF.*`, `CF.*`, `CMS.*`, `TOPK.*` and `TDIGEST.*` need the probabilistic module loaded.
  HyperLogLog (`PF*`) is always there.

## Exercise

1. Count daily unique visitors for 30 days with one HyperLogLog per day, then compute the 30-day unique count with
   `PFCOUNT` over all 30 keys. Compare memory and accuracy with a set per day.
2. Measure a Bloom filter's false-positive rate at 25%, 50%, 100% and 150% of its capacity, with and without
   `NONSCALING`. Plot rate against fill.
3. Build "has this user already been shown this article?" with a Cuckoo filter per user and 30-day expiry. What
   happens to correctness if an article is shown, deleted from the filter, and shown again?
4. Feed 1 million request latencies (drawn from a log-normal distribution) into a `TDIGEST` and compare its p50,
   p95 and p99 with exact percentiles computed in Python.
