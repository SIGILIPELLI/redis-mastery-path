---
description: "maxmemory and eviction measured: noeviction OOM errors, volatile-* with no TTLs, LFU beating LRU 51% to 43%, and how expired keys are freed."
---

# 02 · maxmemory, Eviction & How Expiry Really Works

A cache has to forget things, and Redis forgets in two ways. **Expiry** removes keys whose TTL has passed.
**Eviction** removes keys, TTL or not, when memory reaches the `maxmemory` limit. Both run quietly in the
background, and both behave differently from what most people assume. This lesson measures them on a scratch
server on port 6430.

## The default: no limit, no eviction

```text
127.0.0.1:6430> CONFIG GET maxmemory
1) "maxmemory"
2) "0"
127.0.0.1:6430> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "noeviction"
```

`maxmemory 0` means "no limit" on 64-bit builds. Redis grows until the operating system or a container limit stops
it, and the OOM killer is a much worse eviction policy than any of Redis's. **Always set `maxmemory`** on a
server that holds a cache, leaving headroom below the machine or container limit for fragmentation, client buffers
and the copy-on-write memory of a snapshot (Level 4 · 01).

What happens at the limit depends on the policy. With the default `noeviction`, set a 20 MB limit and keep writing:

```python
r.config_set("maxmemory", "20mb")
n = 0
try:
    while True:
        r.set(f"k:{n}", "v" * 64); n += 1
except redis.ResponseError as e:
    print(f"after {n} keys: {e}")
print("GET still works:", r.get("k:1")[:8], "| DEL works:", r.delete("k:1"))
```

```text
after 152861 keys: command not allowed when used memory > 'maxmemory'.
used_memory_human: 20.00M
GET still works: b'vvvvvvvv' | DEL works: 1
INFO stats evicted_keys: 0
```

Writes that could add memory **fail with an error**. Reads and deletes keep working. That's the right behaviour for
Redis used as a primary store, a queue or a lock service, where silently dropping data would be a bug. It's the wrong
behaviour for a cache, where you want old entries to make room for new ones.

## The eviction policies

| Policy | Evicts from | Chooses |
|---|---|---|
| `noeviction` | — | refuses writes |
| `allkeys-lru` | all keys | least recently used (approximate) |
| `allkeys-lfu` | all keys | least frequently used (approximate) |
| `allkeys-random` | all keys | random |
| `volatile-lru` / `volatile-lfu` / `volatile-random` | only keys with a TTL | as above |
| `volatile-ttl` | only keys with a TTL | shortest remaining TTL |

The `volatile-*` policies are meant for a server that mixes cache entries (with TTLs) and data that must not be
evicted (without TTLs). Their trap is that if no keys have a TTL, there is nothing they're allowed to evict:

```text
volatile-lru, keys without TTL: failed after 66031 keys: command not allowed when used memory > 'maxmemory'.
```

That's the same OOM error as `noeviction`. If a code path forgets to set TTLs, a `volatile-*` server behaves like an
un-evicting one. A cleaner design is to keep durable data and cache data on separate instances, each with the
policy that suits it.

## LRU vs LFU, measured

This benchmark uses 200,000 items with Zipf-like popularity (item *k* requested with weight 1/k^0.9), a 12 MB limit
that holds about 10,700 of the 1 KB values (about 5% of the items), and a one-off batch job in the middle that reads
30,000 cold keys once each, as a report or a crawler might. Each request is a cache-aside `GET`, followed by a `SET`
on a miss:

```python
N_ITEMS = 200_000
WEIGHTS = [1 / (k ** 0.9) for k in range(1, N_ITEMS + 1)]      # Zipf-like popularity
def workload(seed):
    rnd = random.Random(seed)
    hot = rnd.choices(range(1, N_ITEMS + 1), weights=WEIGHTS, k=270_000)
    # requests 100k..130k are a one-off batch job reading 30k distinct cold keys once each
    return ([f"item:{k}" for k in hot[:100_000]] + [f"cold:{i}" for i in range(30_000)]
            + [f"item:{k}" for k in hot[100_000:]])

for policy in ("allkeys-lru", "allkeys-lfu", "allkeys-random"):
    r.flushall(); r.config_set("maxmemory", "12mb"); r.config_set("maxmemory-policy", policy)
    ...   # MGET 500 keys at a time, SET the misses, count hits
```

```text
allkeys-lru     overall hit  43.3%  just after the batch job  41.4%  evicted  157407  keys now 10762
allkeys-lfu     overall hit  51.1%  just after the batch job  56.5%  evicted  135662  keys now 10768
allkeys-random  overall hit  43.0%  just after the batch job  42.3%  evicted  158256  keys now 10747
```

- **LFU won clearly**: 51.1% against 43.3% overall, and 56.5% against 41.4% in the 30,000 requests right after the
  batch job. LFU remembers that popular items are popular, so it didn't let 30,000 one-time reads push them out. LRU
  sees only "recently touched", and the batch job touched everything recently.
- **LRU did no better than random** here. With a long-tailed workload and a cache holding 5% of items, recency alone
  is a weak signal, and Redis's LRU is an approximation (below).
- On a less skewed workload, an earlier run of this script saw all three policies within 0.6 points of each other.
  Measure with your own access pattern before switching. `INFO stats` `keyspace_hits`/`misses` and `evicted_keys`
  are all you need.

## How expiry really works

A key whose TTL has passed is never *returned*. Any command that touches it first checks the deadline and deletes it
(**lazy expiry**). Keys nobody touches are removed by the **active expiry cycle**, which samples keys that have TTLs
several times a second. Here 500,000 keys get a 1-second TTL and are never read, while `DBSIZE` is polled every
0.5 s:

```text
hz=10: (seconds after writing, keys still in memory) -> [(0.5, 119707), (1.0, 18234), (1.5, 0)]
hz=100: (seconds after writing, keys still in memory) -> [(0.5, 121322), (1.0, 41087), (1.5, 0)]
```

(Writing the keys took longer than a second, so the earliest ones had already expired at the first sample.) All
500,000 were gone within about half a second of the last deadline, at both `hz` settings. When the sampler finds a
high share of expired keys, it keeps going instead of stopping at its normal time budget, so a mass expiry is
cleaned up quickly. What `hz` didn't visibly change here, it changes in the opposite case: a few expired keys
scattered among millions of live ones are found more slowly, which is why Level 2 · 07's `expired` notifications
arrived 130–324 ms late. `INFO stats` reports `expired_keys` and `expired_stale_perc` (the sampler's estimate of how
much of the keyspace is expired but not yet deleted).

Two consequences:

- **Memory isn't freed at the deadline.** Expired keys still count towards `used_memory` until they're reclaimed.
  Under memory pressure, eviction may remove live keys while expired ones are still waiting.
- **Expiry is not a timer.** Don't build "do X when this key expires" on it if X has to happen on time.

## How It Actually Works

**Eviction** runs before each command that might use memory. If `used_memory` is above `maxmemory`, Redis frees
keys until it's below, or fails the command if the policy finds nothing to evict. It doesn't keep a global LRU list,
because that would cost two pointers per key and a list update on every access. Instead, each object has a 24-bit
clock field. For LRU it's a coarse "last accessed" timestamp. For LFU it's an 8-bit **logarithmic counter** plus a
16-bit decay time: the counter grows probabilistically (each hit is less likely to increment it as it gets
higher, controlled by `lfu-log-factor`), and it decays over time (`lfu-decay-time`), so yesterday's hot keys cool off.
`OBJECT FREQ key` shows a key's counter when an LFU policy is active.

To pick a victim, Redis samples `maxmemory-samples` keys (default 5) and adds the best candidates to a small
**eviction pool** of 16 entries kept sorted by idle time or frequency, then evicts the worst key in the pool. The
pool carries good candidates over from earlier samples, which makes the approximation surprisingly close to true
LRU/LFU. Raising `maxmemory-samples` to 10 makes it closer still, at some CPU cost.

**Active expiry** runs from `serverCron` (`hz` times per second) and also in a fast cycle before the event loop
sleeps. Each pass samples keys from the expires dictionary of each database, 20 at a time, deletes the expired ones,
and repeats while more than 10% of a sample was expired, within a CPU-time budget. `active-expire-effort` (1–10)
scales those numbers. Since Redis 6 the cycle also remembers where it stopped in each database's expires table,
so it eventually visits everything instead of relying purely on luck. Deletions caused by expiry and eviction are
synchronous unless `lazyfree-lazy-expire` / `lazyfree-lazy-eviction` are on (Level 1 · 04), so evicting huge keys
can cause latency spikes.

On replicas, keys aren't expired independently. The primary sends a `DEL` when it expires a key, and in the meantime a
replica hides logically expired keys from reads so clients don't see them.

## Common mistakes

- **No `maxmemory`.** The OS will "evict" the whole process.
- **`noeviction` on a cache.** Writes start failing at the limit, which usually surfaces as application errors
  rather than a cache problem.
- **`volatile-*` with keys that lack TTLs.** It behaves like `noeviction` once those keys fill memory.
- **Mixing a cache and durable data under `allkeys-*`.** Your queue or lock keys can be evicted.
- **Assuming LRU is best.** Measure. LFU handled one-off scans much better here.
- **Using expiry as a scheduler**, or expecting memory to drop the instant TTLs pass.
- **`maxmemory` equal to the container limit.** Fork-based persistence, buffers and fragmentation need headroom.

## Exercise

1. Rerun the policy benchmark with the cache sized at 1%, 5% and 20% of the items. At which size does the choice of
   policy stop mattering?
2. With `allkeys-lfu`, request one key 1,000 times and another 10 times, then compare `OBJECT FREQ`. Wait a few
   minutes (or lower `lfu-decay-time` to 1) and check again.
3. Fill a 50 MB server with 1-hour-TTL keys under `volatile-ttl`, then write keys with 10-second TTLs. Which keys get
   evicted first, and is that what a session store would want?
4. Set 1,000 keys with a 1-second TTL among 2 million keys without TTLs. Measure how long until all 1,000 are gone
   with `hz 10` and `hz 100`, without ever reading them.
