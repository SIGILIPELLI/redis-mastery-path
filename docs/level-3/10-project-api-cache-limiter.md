---
description: "Level 3 project: a FastAPI service guarded by Redis with rate limits and 429s, stampede-proof caching, stale-while-revalidate and tests."
---

# 10 · Project — Caching & Rate-Limiting Layer for an API

This project puts Level 3 in front of a real HTTP API. A small FastAPI service has a deliberately slow "database"
(50 ms for a product, 300 ms for a report), and a reusable `Guard` class adds:

- **rate limiting** per API key (sliding window counter, Level 3 · 04) with `429`, `Retry-After` and
  `RateLimit-Remaining` headers;
- **cache-aside** with a **rebuild lock** so a cold key costs one database call, not thirty (Level 3 · 03);
- **stale-while-revalidate** for the expensive report;
- **versioned invalidation** that can't be undone by a slow reader writing back stale data (Level 3 · 01).

Versions used: FastAPI 0.143.0, Starlette 1.8.0, Uvicorn 0.54.0, httpx 0.28.1, redis-py 8.1.0, Redis 8.10.2.

## The Guard

`guard.py`:

```python
"""Redis-backed caching (cache-aside + stale-while-revalidate + versioned invalidation) and rate limiting."""
import json, threading, time, uuid
import redis

LIMIT_LUA = """
-- sliding window counter using the server clock
-- KEYS[1] = current window key, KEYS[2] = previous window key ; ARGV = limit, window_ms
local t = redis.call('TIME')
local now = t[1] * 1000 + math.floor(t[2] / 1000)
local w, limit = tonumber(ARGV[2]), tonumber(ARGV[1])
local prev = tonumber(redis.call('GET', KEYS[2]) or '0')
local cur = tonumber(redis.call('GET', KEYS[1]) or '0')
local weight = 1 - (now % w) / w
local used = prev * weight + cur
if used >= limit then
  return {0, 0, w - (now % w)}
end
redis.call('INCR', KEYS[1]); redis.call('PEXPIRE', KEYS[1], w * 2)
return {1, math.floor(limit - used - 1), 0}
"""

SET_IF_VERSION_LUA = """
if (redis.call('GET', KEYS[2]) or '0') == ARGV[1] then
  return redis.call('SET', KEYS[1], ARGV[2], 'EX', ARGV[3])
end
return false
"""


class Guard:
    def __init__(self, r: redis.Redis, prefix="api"):
        self.r, self.p = r, prefix
        self._limit = r.register_script(LIMIT_LUA)
        self._set_if_version = r.register_script(SET_IF_VERSION_LUA)

    # ---------------- rate limiting ----------------
    def allow(self, client: str, limit: int, window_ms: int = 1000, now_ms: int | None = None):
        """Returns (allowed, remaining, retry_after_ms). Window keys share a hash tag per client."""
        now_ms = now_ms or int(time.time() * 1000)            # only used to name the windows
        win = now_ms // window_ms
        cur, prev = f"{self.p}:rl:{{{client}}}:{win}", f"{self.p}:rl:{{{client}}}:{win - 1}"
        ok, remaining, retry = self._limit(keys=[cur, prev], args=[limit, window_ms])
        return bool(ok), int(remaining), int(retry)

    # ---------------- caching ----------------
    def _ver_key(self, tag):
        return f"{self.p}:ver:{tag}"

    def invalidate(self, tag):
        """Bump the tag's version: every cached entry built from the old version becomes unreachable."""
        self.r.incr(self._ver_key(tag))

    def cached(self, tag: str, name: str, loader, ttl: int, grace: int = 0, lock_ms: int = 5000):
        """Return (value, status) where status is HIT, MISS, STALE or WAIT."""
        ver = self.r.get(self._ver_key(tag)) or "0"
        key = f"{self.p}:cache:{tag}:v{ver}:{name}"
        lock = f"{key}:lock"
        raw = self.r.get(key)
        if raw is not None:
            entry = json.loads(raw)
            if entry["fresh_until"] > time.time():
                return entry["value"], "HIT"
            if grace and self.r.set(lock, 1, nx=True, px=lock_ms):  # stale: refresh in background
                threading.Thread(target=self._fill, args=(tag, ver, key, lock, loader, ttl, grace), daemon=True).start()
            return entry["value"], "STALE"
        # cold miss: one loader, everyone else waits for it
        deadline = time.time() + lock_ms / 1000
        while time.time() < deadline:
            if self.r.set(lock, 1, nx=True, px=lock_ms):
                raw = self.r.get(key)                         # re-check: the previous holder may have
                if raw is not None:                           # just filled it and released the lock
                    self.r.delete(lock)
                    return json.loads(raw)["value"], "WAIT"
                return self._fill(tag, ver, key, lock, loader, ttl, grace), "MISS"
            time.sleep(0.01)
            raw = self.r.get(key)
            if raw is not None:
                return json.loads(raw)["value"], "WAIT"
        return loader(), "MISS"                                   # lock holder died: degrade gracefully

    def _fill(self, tag, ver, key, lock, loader, ttl, grace):
        try:
            value = loader()
            entry = json.dumps({"value": value, "fresh_until": time.time() + ttl})
            # write back only if nobody invalidated the tag while we were loading
            self._set_if_version(keys=[key, self._ver_key(tag)], args=[ver, entry, ttl + grace])
            return value
        finally:
            self.r.delete(lock)
```

Design notes:

- **Rate limiting uses Redis's clock** (`TIME` inside the script) for the sliding-window weight, so app servers with
  skewed clocks still agree on how full a window is. The window *key names* are computed by the caller, because
  scripts must receive their keys in `KEYS`. A caller whose clock is far off would pick a different window key. With
  NTP-synced servers that's a non-issue, but it's a trade-off worth knowing about. The client ID sits in a hash tag
  (`{client}`), so both window keys live in one Cluster slot.
- **Every cache key contains the tag's version** (`api:cache:product:7:v3:detail`). `invalidate()` is a single `INCR`.
  Old entries become unreachable at once and expire on their own.
- **The write-back is conditional on the version** that was current when loading started, so a load that overlaps
  an update can't plant stale data (`test_write_during_refresh_is_not_cached`).
- **`X-Cache` reports what happened**: `HIT`, `MISS` (this request loaded it), `WAIT` (waited for another
  request's load) or `STALE` (served old data while a background refresh runs).

## The app

`app.py`:

```python
"""A small API with a slow 'database', protected by Guard."""
import threading, time
import redis
from fastapi import FastAPI, HTTPException, Request, Response

r = redis.Redis(decode_responses=True)
from guard import Guard
guard = Guard(r)
app = FastAPI()

DB = {i: {"id": i, "name": f"Product {i}", "price_cents": 1000 + i} for i in range(1, 101)}
DB_CALLS = {"product": 0, "report": 0}
_lock = threading.Lock()

def db_product(pid):
    with _lock: DB_CALLS["product"] += 1
    time.sleep(0.05)                                    # 50 ms query
    return DB.get(pid)

def db_report():
    with _lock: DB_CALLS["report"] += 1
    time.sleep(0.3)                                     # 300 ms aggregate
    return {"top": sorted(DB.values(), key=lambda p: -p["price_cents"])[:3], "built_at": time.time()}

@app.middleware("http")
async def rate_limit(request: Request, call_next):
    client = request.headers.get("x-api-key", "anon")
    ok, remaining, retry_ms = guard.allow(client, limit=20, window_ms=1000)
    if not ok:
        return Response(status_code=429, headers={"Retry-After": str(max(1, -(-retry_ms // 1000))),
                                                  "RateLimit-Remaining": "0"})
    response = await call_next(request)
    response.headers["RateLimit-Remaining"] = str(remaining)
    return response

@app.get("/products/{pid}")
def get_product(pid: int, response: Response):
    value, status = guard.cached(f"product:{pid}", "detail", lambda: db_product(pid), ttl=60)
    if value is None:
        raise HTTPException(404)
    response.headers["X-Cache"] = status
    return value

@app.put("/products/{pid}")
def put_product(pid: int, body: dict):
    if pid not in DB: raise HTTPException(404)
    DB[pid] = {**DB[pid], **body}                       # 1. write the source of truth
    guard.invalidate(f"product:{pid}")                  # 2. then invalidate by version
    guard.invalidate("report")
    return DB[pid]

@app.get("/reports/top")
def top_report(response: Response):
    value, status = guard.cached("report", "top", db_report, ttl=2, grace=60)
    response.headers["X-Cache"] = status
    return value
```

The order in `put_product` matters: change the source of truth first, then invalidate. Invalidating first would let
a concurrent read reload the *old* row and cache it under the new version.

## Tests

`test_api.py` uses FastAPI's `TestClient` and a distinct API key per test, so rate limits don't interfere:

```python
import threading, time
import pytest
from fastapi.testclient import TestClient
import app as appmod

@pytest.fixture
def client():
    for k in appmod.r.scan_iter("api:*"): appmod.r.unlink(k)
    appmod.DB_CALLS.update(product=0, report=0)
    with TestClient(appmod.app) as c:
        yield c

def key(i):            # a distinct API key per test so rate limits don't interfere
    return {"x-api-key": f"test-{i}-{time.time_ns()}"}

def test_miss_then_hit(client):
    h = key(1)
    r1 = client.get("/products/7", headers=h); r2 = client.get("/products/7", headers=h)
    assert (r1.headers["x-cache"], r2.headers["x-cache"]) == ("MISS", "HIT")
    assert r2.json()["price_cents"] == 1007 and appmod.DB_CALLS["product"] == 1

def test_not_found_is_404(client):
    assert client.get("/products/999", headers=key(2)).status_code == 404

def test_update_invalidates(client):
    h = key(3)
    client.get("/products/5", headers=h)
    client.put("/products/5", json={"price_cents": 1}, headers=h)
    r = client.get("/products/5", headers=h)
    assert r.headers["x-cache"] == "MISS" and r.json()["price_cents"] == 1

def test_stampede_single_db_call(client):
    results = []
    def hit(i):
        results.append(client.get("/products/42", headers=key(100 + i)).headers["x-cache"])
    ts = [threading.Thread(target=hit, args=(i,)) for i in range(30)]
    [t.start() for t in ts]; [t.join() for t in ts]
    assert appmod.DB_CALLS["product"] == 1
    assert results.count("MISS") == 1 and set(results) <= {"MISS", "WAIT", "HIT"}

def test_stale_while_revalidate(client):
    h = key(4)
    assert client.get("/reports/top", headers=h).headers["x-cache"] == "MISS"
    time.sleep(2.1)                                         # past the 2 s soft TTL
    t = time.perf_counter(); r = client.get("/reports/top", headers=h); el = time.perf_counter() - t
    assert r.headers["x-cache"] == "STALE" and el < 0.1     # served instantly, not after 300 ms
    time.sleep(0.5)                                         # background refresh finishes
    assert client.get("/reports/top", headers=h).headers["x-cache"] == "HIT"
    assert appmod.DB_CALLS["report"] == 2

def test_write_during_refresh_is_not_cached(client):
    h = key(5)
    def slow_get():
        client.get("/products/9", headers=h)               # loader takes 50 ms
    t = threading.Thread(target=slow_get); t.start(); time.sleep(0.02)
    client.put("/products/9", json={"price_cents": 5}, headers=h)
    t.join()
    r = client.get("/products/9", headers=h)
    assert r.json()["price_cents"] == 5

def test_rate_limit_429(client):
    h = key(6)
    codes = [client.get("/products/1", headers=h).status_code for _ in range(30)]
    assert codes.count(200) <= 21 and codes.count(429) >= 9
    blocked = client.get("/products/1", headers=h)
    assert blocked.status_code == 429 and int(blocked.headers["retry-after"]) >= 1

def test_limit_is_per_client(client):
    a, b = key(7), key(8)
    for _ in range(25): client.get("/products/1", headers=a)
    assert client.get("/products/1", headers=b).status_code == 200

def test_remaining_header_counts_down(client):
    h = key(9)
    rem = [int(client.get("/products/2", headers=h).headers["ratelimit-remaining"]) for _ in range(3)]
    assert rem[0] > rem[1] > rem[2]
```

```text
test_api.py::test_miss_then_hit PASSED                                   [ 11%]
test_api.py::test_not_found_is_404 PASSED                                [ 22%]
test_api.py::test_update_invalidates PASSED                              [ 33%]
test_api.py::test_stampede_single_db_call PASSED                         [ 44%]
test_api.py::test_stale_while_revalidate PASSED                          [ 55%]
test_api.py::test_write_during_refresh_is_not_cached PASSED              [ 66%]
test_api.py::test_rate_limit_429 PASSED                                  [ 77%]
test_api.py::test_limit_is_per_client PASSED                             [ 88%]
test_api.py::test_remaining_header_counts_down PASSED                    [100%]

============================== 9 passed in 24.13s ==============================
```

Most of the 24 seconds is fixture setup, which scans the lab database (it still holds about 100,000 keys from earlier
lessons) to delete `api:*` keys, plus the deliberate sleeps in the SWR test.

## The bug the stampede test caught

The first version of `cached()` passed every test except one:

```text
>       assert appmod.DB_CALLS["product"] == 1
E       assert 2 == 1

FAILED test_api.py::test_stampede_single_db_call - assert 2 == 1
1 failed, 8 passed, 1 warning in 24.71s
```

Thirty concurrent requests produced **two** database calls. The waiter loop was:

```text
try SET lock NX  ->  fail
sleep 10 ms
GET key          ->  still empty (loader not finished)
try SET lock NX  ->  succeeds, because the loader has meanwhile written the key AND released the lock
load again       ->  second database call
```

A waiter could acquire the lock in the instant between the first loader finishing and the waiter's next check. The
fix is the classic **double-check**: after acquiring the lock, look at the cache once more before loading:

```python
if self.r.set(lock, 1, nx=True, px=lock_ms):
    raw = self.r.get(key)                         # re-check: the previous holder may have
    if raw is not None:                           # just filled it and released the lock
        self.r.delete(lock)
        return json.loads(raw)["value"], "WAIT"
    return self._fill(tag, ver, key, lock, loader, ttl, grace), "MISS"
```

After the fix the suite passed three runs in a row. Concurrency bugs like this are invisible in single-request
testing and only show up when you assert on the *count* of backend calls under real parallelism.

## Running it for real

Start the app under Uvicorn and send traffic with `curl`:

```bash
uvicorn app:app --port 8077
```

Twenty-five rapid requests with one API key against a limit of 20 per second:

```text
25 rapid requests, one API key:
200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 429 429 429 429
HTTP/1.1 429 Too Many Requests
retry-after: 1
ratelimit-remaining: 0
```

**21** requests got through, one more than the limit. That's the sliding-window counter's approximation (Level 3 · 04):
the requests straddled a window boundary, and the weighted estimate of the previous window let one extra through.
If the limit must be exact, use a sliding log.

Twenty concurrent first requests for the 300 ms report, with the twenty `X-Cache` response headers tallied:

```text
x-cache: WAIT    (×19)
x-cache: MISS    (×1)
```

One request loaded the report and nineteen waited for its result. That's one database query instead of twenty.

## How It Actually Works

Each request does a small, fixed number of Redis round trips: one `EVALSHA` for the rate limit, a `GET` for the
version, a `GET` for the cache entry, and on a miss, a `SET NX` for the lock, a conditional `SET` via script and a
`DEL`. Every decision that must be exclusive (who loads, whether the write-back is still valid, whether the client is
over its limit) is a single atomic Redis operation. The Python code only orchestrates.

Stale-while-revalidate splits freshness from existence: the entry carries `fresh_until` (soft TTL) and the key has
`ttl + grace` (hard TTL). Within the grace period, users never wait for the report. If the background refresh keeps
failing, the entry eventually expires and the next request loads synchronously, so a broken backend can't keep
serving stale data forever.

What this layer doesn't do: it doesn't make Redis durable (rate-limit counters and cache entries are disposable, so
that's fine), and it doesn't protect against Redis being down. A production version should catch connection errors
and decide, per endpoint, whether to fail open (serve from the database, skip rate limiting) or fail closed.

## Common mistakes

- **No double-check after acquiring a rebuild lock.** Two loads under load, as the test showed.
- **Invalidating before writing the database.** A concurrent read caches the old row.
- **Rate limits in process memory.** With N app servers, every client gets N× the limit. Shared state belongs in
  Redis.
- **One global limit key.** Hot-spots one Redis node, and one noisy client throttles everyone. Key by client.
- **Missing `Retry-After`.** Well-behaved clients can't back off properly without it.
- **Failing closed by accident** when Redis is unreachable. Decide explicitly.

## Exercise

1. Add fail-open behaviour: if Redis raises a connection error, skip rate limiting and call the loader directly, and
   expose a counter of degraded requests. Test it by pointing the app at a stopped Redis.
2. Give different API keys different limits (free 20/s, pro 200/s) stored in a Redis hash, cached in-process with
   client-side caching (Level 3 · 06) so the lookup costs nothing on the hot path.
3. Replace the sliding window counter with a token bucket for `/reports/top` only (bursty but expensive), with a cost
   of 5 tokens per call.
4. Load-test with `ab -n 5000 -c 50` against `/products/{random id}` and report the hit ratio, p99 latency and the
   number of database calls. Then double the product catalogue so that it no longer fits under a `maxmemory` limit,
   and repeat with `allkeys-lru` and `allkeys-lfu`.
