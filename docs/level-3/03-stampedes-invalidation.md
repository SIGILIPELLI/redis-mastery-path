---
description: "Cache stampedes measured, 50 DB calls for one expired key, and the fixes compared: rebuild locks, XFetch, stale-while-revalidate and jitter."
---

# 03 · Cache Stampedes & Invalidation

A cache protects a slow backend only while it holds the answer. The moment a popular key expires, every request
that arrives before someone rebuilds it misses at the same time, and each one runs the expensive query. That's a
**cache stampede** (or thundering herd), and it tends to strike exactly when traffic is highest. A cache sized to
cut database load 50× can, for a few hundred milliseconds, multiply it.

This lesson reproduces a stampede against a "report query" that takes 200 ms, then compares three defences.

## The experiment

```python
import json, math, random, threading, time, uuid
import redis
r = redis.Redis(decode_responses=True)
db_calls = 0; lock = threading.Lock()

def expensive_query():
    global db_calls
    with lock: db_calls += 1
    time.sleep(0.2)                         # a 200 ms report query
    return {"total": 42, "at": time.time()}

# 1. naive cache-aside
def get_naive(key):
    v = r.get(key)
    if v: return json.loads(v)
    val = expensive_query(); r.set(key, json.dumps(val), ex=60); return val

# 2. one rebuilder at a time, everyone else waits for the result
def get_locked(key, ttl=60, lock_ms=5000):
    while True:
        v = r.get(key)
        if v: return json.loads(v)
        token = uuid.uuid4().hex
        if r.set(f"{key}:lock", token, nx=True, px=lock_ms):
            try:
                val = expensive_query(); r.set(key, json.dumps(val), ex=ttl); return val
            finally:
                r.eval("if redis.call('GET',KEYS[1])==ARGV[1] then return redis.call('DEL',KEYS[1]) end return 0",
                       1, f"{key}:lock", token)            # release only our own lock
        time.sleep(0.02)                    # someone else is rebuilding; poll

# 3. probabilistic early expiration (XFetch): recompute *before* expiry with rising probability
def get_xfetch(key, ttl=60, beta=1.0):
    p = r.pipeline(transaction=False); p.get(key); p.pttl(key)
    v, pttl = p.execute()
    if v:
        data = json.loads(v)
        delta = data["compute_s"]           # how long the last rebuild took
        if -delta * beta * math.log(random.random()) < pttl / 1000:
            return data["value"]            # not yet: serve cached
    t = time.perf_counter(); val = expensive_query()
    r.set(key, json.dumps({"value": val, "compute_s": time.perf_counter() - t}), ex=ttl)
    return val

# 4. stale-while-revalidate: a soft TTL inside the value, a longer hard TTL on the key
def get_swr(key, ttl=60, grace=300):
    v = r.get(key)
    if v:
        data = json.loads(v)
        if time.time() < data["fresh_until"]:
            return data["value"]
        # stale: one caller refreshes in the background, everyone returns the stale value now
        if r.set(f"{key}:lock", 1, nx=True, px=5000):
            def refresh():
                val = expensive_query()
                r.set(key, json.dumps({"value": val, "fresh_until": time.time() + ttl}), ex=ttl + grace)
                r.delete(f"{key}:lock")
            threading.Thread(target=refresh).start()
        return data["value"]
    return get_locked_swr(key, ttl, grace)       # cold start: fall back to the lock

def get_locked_swr(key, ttl, grace):
    while True:
        v = r.get(key)
        if v: return json.loads(v)["value"]
        if r.set(f"{key}:lock", 1, nx=True, px=5000):
            val = expensive_query()
            r.set(key, json.dumps({"value": val, "fresh_until": time.time() + ttl}), ex=ttl + grace)
            r.delete(f"{key}:lock"); return val
        time.sleep(0.02)

def herd(fn, key, n=50):
    global db_calls
    r.delete(key, f"{key}:lock"); db_calls = 0
    lat = []
    def one():
        t = time.perf_counter(); fn(key); lat.append(time.perf_counter() - t)
    ts = [threading.Thread(target=one) for _ in range(n)]
    [t.start() for t in ts]; [t.join() for t in ts]
    lat.sort()
    return db_calls, lat[len(lat)//2] * 1000, lat[-1] * 1000

print("50 concurrent requests for a key that just expired:")
print("  naive      : db calls %2d   p50 %4.0f ms  max %4.0f ms" % herd(get_naive, "report:naive"))
print("  lock       : db calls %2d   p50 %4.0f ms  max %4.0f ms" % herd(get_locked, "report:lock"))

# XFetch: simulate steady traffic around expiry with a short TTL
global_calls = 0
def steady(fn, key, ttl, seconds, rps=200):
    global db_calls
    r.delete(key, f"{key}:lock"); db_calls = 0
    end = time.time() + seconds; misses_blocking = 0
    def client():
        nonlocal misses_blocking
        while time.time() < end:
            t = time.perf_counter(); fn(key, ttl=ttl)
            if time.perf_counter() - t > 0.1: misses_blocking += 1
            time.sleep(threading.active_count() / rps)
    ts = [threading.Thread(target=client) for _ in range(20)]
    [t.start() for t in ts]; [t.join() for t in ts]
    return db_calls, misses_blocking
print("20 clients, 10 s, TTL 2 s:")
print("  lock       : db calls %2d   requests that waited >100 ms: %d" % steady(get_locked, "s:lock", 2, 10))
print("  xfetch     : db calls %2d   requests that waited >100 ms: %d" % steady(get_xfetch, "s:xf", 2, 10))
print("  swr        : db calls %2d   requests that waited >100 ms: %d" % steady(get_swr, "s:swr", 2, 10))
```

```text
50 concurrent requests for a key that just expired:
  naive      : db calls 50   p50  222 ms  max  231 ms
  lock       : db calls  1   p50  209 ms  max  230 ms
20 clients, 10 s, TTL 2 s:
  lock       : db calls  5   requests that waited >100 ms: 100
  xfetch     : db calls 35   requests that waited >100 ms: 35
  swr        : db calls  5   requests that waited >100 ms: 20
```

## 1. The stampede

Fifty requests arrive for a key that has just expired:

- **Naive cache-aside: 50 database calls.** Every request missed, and every one ran the 200 ms query. For a
  heavier query, or with hundreds of requests, this is how a database falls over the moment a cache entry expires.
- **With a rebuild lock: 1 database call.** The first request takes `SET key:lock <token> NX PX 5000` and rebuilds,
  and the other 49 poll every 20 ms until the value appears. Latency was about the same (p50 209 ms against 222 ms; the query itself takes 200 ms),
  but the database did one-fiftieth of the work.

The lock has three details that matter. It has an **expiry** (`PX 5000`), so a rebuilder that crashes doesn't block
everyone forever. It holds a **random token**, and the release only deletes the lock if the token still matches, so
a slow rebuilder whose lock already expired can't delete someone else's. And waiters **re-check the cache** in a
loop rather than assuming the lock holder succeeded. Level 3 · 05 looks at locks in more depth.

## 2. Steady traffic: who waits?

The lock stops the herd, but someone still waits for every rebuild. With 20 clients making requests for 10 seconds
against a 2-second TTL:

| Strategy | DB calls | Requests that waited > 100 ms |
|---|---|---|
| Lock | 5 | 100 |
| XFetch (probabilistic early expiry) | 35 | 35 |
| Stale-while-revalidate | 5 | 20 |

**Lock.** Few rebuilds, but each expiry made every concurrent request wait for the query, 100 slow requests in all.

**XFetch** ("optimal probabilistic cache stampede prevention", Vattani, Chierichetti and Lowenstein, 2015). Each
reader recomputes *early*, with a probability that rises as expiry approaches and as the rebuild cost (`delta`)
grows: recompute if `−delta · beta · ln(rand) ≥ remaining TTL`. Requests spread the rebuilds out, so there's no
synchronized miss, and only the reader that decides to recompute waits. Its weakness showed in the numbers: the run
started with an empty cache, so all 20 clients missed at once and rebuilt together. With a 2-second TTL against a
200 ms rebuild it also recomputed often. XFetch shines with longer TTLs relative to rebuild time, and it should be
combined with a lock for the cold start.

**Stale-while-revalidate** won on both counts. The value carries a *soft* expiry (`fresh_until`), and the key has a
longer *hard* TTL (soft + grace). When a reader finds the value stale, it returns the stale value **immediately**,
and one reader (the one that gets the lock) refreshes it in the background. The only slow requests were the 20 at
cold start, when there was nothing stale to serve. Every later refresh was invisible to users.

The trade-off is explicit: SWR serves data that's up to `ttl + refresh time` old, and the grace period bounds how
stale it can get if refreshing keeps failing. For dashboards, catalogues and recommendation lists that's an
excellent deal. For anything where stale is harmful, it isn't, and Level 3 · 01's versioned write-back is the tool.

## 3. Synchronized expiry: add jitter

Stampedes also happen without a single hot key. A cache warmed by a batch job sets 100,000 keys in a minute, all
with `EX 3600`, and an hour later they all expire within the same minute. Spread them out:

```python
ttl = 3600 + random.randint(0, 600)    # 1 h plus up to 10 min of jitter
r.set(key, value, ex=ttl)
```

That costs nothing and turns one expiry cliff into a gentle slope.

## 4. Invalidating groups of keys

Invalidation often targets a *group*: "everything cached for tenant 42" or "all pages that show product 7". Three
ways to do it, from best to worst:

1. **Version in the key name.** Keep `cache:ver:tenant:42` and build keys as `cache:t42:v{ver}:...`. Invalidation is
   `INCR cache:ver:tenant:42`. Old keys become unreachable and expire through their TTLs. It's O(1), with no
   scanning and no race, at the cost of a little memory until they expire.
2. **Tag sets.** On each cache write, `SADD tag:product:7 <cache key>` (with a TTL on the tag set). To invalidate,
   read the set and `UNLINK` its members, then the set itself. This is precise, but the tag sets need maintenance
   and can grow large.
3. **`SCAN MATCH` + `UNLINK`.** Fine for a one-off cleanup (Level 1 · 04), but too slow and too heavy for routine
   invalidation of large keyspaces.

## How It Actually Works

A stampede is a coordination problem. Many independent requests discover the same fact (the value is missing) at
the same moment, and none of them knows the others exist. Every fix gives them a way to coordinate through Redis:

- The **lock** uses `SET NX`, which atomically elects exactly one rebuilder.
- **XFetch** coordinates statistically. Each reader makes an independent random decision, tuned so the *expected*
  number of early recomputations is small and they're spread over time before the deadline.
- **SWR** separates "is there something to serve?" (yes, until the hard TTL) from "should it be refreshed?" (yes,
  after the soft TTL), and uses `SET NX` to elect one refresher.

All of them rely on Redis being a single, linearizable point of coordination. If the cache is a Cluster, the lock key
and the data key should share a hash tag so they live on the same node (`report:{q1}` and `report:{q1}:lock`).

## Common mistakes

- **No stampede protection on expensive keys.** It works in testing and fails at peak.
- **Locks without expiry**, or released without checking the token.
- **Waiters that block forever.** Always loop with a deadline, then fall back to computing it yourself or returning an
  error.
- **Identical TTLs on keys written together.** Add jitter.
- **Serving stale data where staleness is unacceptable** (prices at checkout, permissions).
- **Invalidating by scanning** on every write. Use key versions or tags.

## Exercise

1. Run the steady-traffic test with TTLs of 2, 10 and 60 seconds. How do XFetch's database-call count and waits
   change as the TTL grows relative to the 200 ms rebuild?
2. Add a lock to XFetch's cold-start path and rerun. How many requests still wait?
3. Make SWR's background refresh fail half the time (raise an exception). What do clients see during the grace
   period, and after it ends?
4. Implement tenant-wide invalidation with a version counter and show that invalidating 100,000 cached entries takes a
   single `INCR`. How much memory do the orphaned entries hold until they expire?
