---
description: "Four rate limiters as atomic Lua scripts: fixed window, sliding log, sliding counter and token bucket, with boundary bursts and memory measured."
---

# 04 · Rate Limiting Algorithms

"At most 10 requests per second per client" sounds like one requirement, but there are several reasonable
definitions of it, and they differ exactly where it matters: at bursts and window boundaries. Redis is the
standard place to keep rate-limit state, because every app server sees the same counters, and because one script
can check and update a limit atomically. This lesson implements four algorithms and runs the same traffic through
all of them.

## The implementations

Each limiter is a Lua script, so "read the state, decide, update" can't interleave with another request. All four
take the current time as an argument, which makes the tests deterministic. In production, either pass a time from
the app servers (and keep their clocks in sync) or read `redis.call('TIME')` inside the script.

`limiters.py`:

```python
"""Four rate limiters, each one atomic in Redis. All take an explicit now_ms so tests are deterministic."""
import redis

r = redis.Redis(decode_responses=True)

FIXED = r.register_script("""
-- KEYS[1] = prefix ; ARGV = limit, window_ms, now_ms
local window = math.floor(ARGV[3] / ARGV[2])
local key = KEYS[1] .. ':' .. window
local n = redis.call('INCR', key)
if n == 1 then redis.call('PEXPIRE', key, ARGV[2]) end
return n <= tonumber(ARGV[1]) and 1 or 0
""")

SLIDING_LOG = r.register_script("""
-- KEYS[1] = zset ; ARGV = limit, window_ms, now_ms, request_id
redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[3] - ARGV[2])
if redis.call('ZCARD', KEYS[1]) < tonumber(ARGV[1]) then
  redis.call('ZADD', KEYS[1], ARGV[3], ARGV[4])
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
end
return 0
""")

SLIDING_COUNTER = r.register_script("""
-- KEYS[1] = prefix ; ARGV = limit, window_ms, now_ms
local w = tonumber(ARGV[2]); local now = tonumber(ARGV[3])
local cur = math.floor(now / w)
local cur_key, prev_key = KEYS[1] .. ':' .. cur, KEYS[1] .. ':' .. (cur - 1)
local prev = tonumber(redis.call('GET', prev_key) or '0')
local count = tonumber(redis.call('GET', cur_key) or '0')
local weight = 1 - (now % w) / w                -- share of the previous window still inside the sliding window
if prev * weight + count >= tonumber(ARGV[1]) then return 0 end
redis.call('INCR', cur_key); redis.call('PEXPIRE', cur_key, w * 2)
return 1
""")

TOKEN_BUCKET = r.register_script("""
-- KEYS[1] = hash ; ARGV = capacity, refill_per_sec, now_ms, cost
local cap, rate = tonumber(ARGV[1]), tonumber(ARGV[2])
local now, cost = tonumber(ARGV[3]), tonumber(ARGV[4])
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or cap
local ts = tonumber(b[2]) or now
tokens = math.min(cap, tokens + (now - ts) / 1000 * rate)
local ok = 0
if tokens >= cost then tokens = tokens - cost; ok = 1 end
redis.call('HSET', KEYS[1], 'tokens', tostring(tokens), 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(cap / rate * 1000) + 1000)
return ok
""")

def fixed(client, now_ms, limit=10, window_ms=1000):
    return FIXED(keys=[f"rl:fixed:{client}"], args=[limit, window_ms, now_ms]) == 1

def sliding_log(client, now_ms, req_id, limit=10, window_ms=1000):
    return SLIDING_LOG(keys=[f"rl:log:{client}"], args=[limit, window_ms, now_ms, req_id]) == 1

def sliding_counter(client, now_ms, limit=10, window_ms=1000):
    return SLIDING_COUNTER(keys=[f"rl:swc:{client}"], args=[limit, window_ms, now_ms]) == 1

def token_bucket(client, now_ms, capacity=10, per_sec=10, cost=1):
    return TOKEN_BUCKET(keys=[f"rl:tb:{client}"], args=[capacity, per_sec, now_ms, cost]) == 1
```

How each one decides:

- **Fixed window.** One counter per client per window (`rl:fixed:ana:1791860000`). `INCR`, and allow if the result is
  ≤ limit. The cheapest and simplest.
- **Sliding log.** A sorted set of request timestamps. Drop entries older than the window, count, and allow if under
  the limit. Exact, but it stores one entry per request.
- **Sliding window counter.** Two fixed-window counters, the current and the previous one. Estimate the count over
  the last second as `previous × (share of previous window still in range) + current`. Nearly exact, with constant
  memory.
- **Token bucket.** A bucket holds up to `capacity` tokens and refills at `rate` per second. Each request takes a token
  if one is available. It allows **bursts** up to the capacity, then a steady rate. Stored as two hash fields, and
  refill is computed lazily from the elapsed time, so there's no background job.

## The boundary burst

The limit is 10 requests per second. A client sends 15 requests at t = 0.990 s and 15 more at t = 1.010 s, which is
30 requests within 20 ms, straddling a one-second boundary:

```python
from limiters import r, fixed, sliding_log, sliding_counter, token_bucket
T0 = 1_791_860_000_000                         # a window boundary (divisible by 1000)
for k in r.scan_iter("rl:*"): r.delete(k)
# limit: 10 requests per 1 s. Burst of 15 at t=0.990 s, then 15 more at t=1.010 s
times = [T0 + 990] * 15 + [T0 + 1010] * 15
def run(name, fn):
    allowed = [fn(t, i) for i, t in enumerate(times)]
    print(f"{name:16} allowed {sum(allowed[:15]):2} at 0.990s + {sum(allowed[15:]):2} at 1.010s = {sum(allowed):2} within 20 ms")
run("fixed window",    lambda t, i: fixed("ana", t))
run("sliding log",     lambda t, i: sliding_log("ana", t, f"req-{i}"))
run("sliding counter", lambda t, i: sliding_counter("ana", t))
run("token bucket",    lambda t, i: token_bucket("ana", t))
```

```text
fixed window     allowed 10 at 0.990s + 10 at 1.010s = 20 within 20 ms
sliding log      allowed 10 at 0.990s +  0 at 1.010s = 10 within 20 ms
sliding counter  allowed 10 at 0.990s +  1 at 1.010s = 11 within 20 ms
token bucket     allowed 10 at 0.990s +  0 at 1.010s = 10 within 20 ms
```

The **fixed window allowed 20 requests in 20 milliseconds**, twice the limit, because the first burst counted
against window 0 and the second against window 1, each of which was "under 10". A client that times bursts to
window boundaries can always get double the limit. The sliding log held exactly at 10. The sliding counter allowed
one extra: at 1.010 s it estimated 10 × 0.99 + 0 = 9.9, which is under 10. That's the approximation, and it's small.
The token bucket allowed 10, because the bucket was empty after the first burst and only 20 ms worth of tokens
(0.2) had dripped back in.

## Steady overload

A client sends 200 requests at random times over 10 seconds (about 20/s, double the limit). "Most in any 1-s span"
slides a one-second window across the accepted requests in 5 ms steps:

```python
import random
from limiters import r, fixed, sliding_log, sliding_counter, token_bucket
T0 = 1_791_860_000_000
for k in r.scan_iter("rl:*"): r.delete(k)
random.seed(4)
# one client sending ~20 req/s at random times for 10 s against a 10 req/s limit
times = sorted(T0 + random.randint(0, 9999) for _ in range(200))
def worst_1s(accepted):
    return max(sum(1 for t in accepted if s <= t < s + 1000) for s in range(T0, T0 + 9000, 5))
for name, fn in [("fixed window", lambda t, i: fixed("bob", t)),
                 ("sliding log", lambda t, i: sliding_log("bob", t, f"r{i}")),
                 ("sliding counter", lambda t, i: sliding_counter("bob", t)),
                 ("token bucket", lambda t, i: token_bucket("bob", t))]:
    acc = [t for i, t in enumerate(times) if fn(t, i)]
    print(f"{name:16} accepted {len(acc):3} of 200 in 10 s; most in any 1-s span: {worst_1s(acc)}")
# memory per client at the limit
for name, key in [("fixed window", "rl:fixed:bob:*"), ("sliding log", "rl:log:bob"), ("sliding counter", "rl:swc:bob:*"), ("token bucket", "rl:tb:bob")]:
    keys = list(r.scan_iter(key))
    print(f"{name:16} keys per client {len(keys)}, bytes {sum(r.memory_usage(k) or 0 for k in keys)}")
```

```text
fixed window     accepted 100 of 200 in 10 s; most in any 1-s span: 17
sliding log      accepted  93 of 200 in 10 s; most in any 1-s span: 10
sliding counter  accepted  99 of 200 in 10 s; most in any 1-s span: 12
token bucket     accepted 108 of 200 in 10 s; most in any 1-s span: 19
fixed window     keys per client 10, bytes 500
sliding log      keys per client 1, bytes 204
sliding counter  keys per client 10, bytes 480
token bucket     keys per client 1, bytes 71
```

- All four let through roughly the long-run rate of about 10/s (93–108 of 200).
- The **sliding log never exceeded 10** in any one-second span. It's the only exact one.
- The **sliding counter peaked at 12**, a small overshoot from its weighting.
- The **fixed window peaked at 17** by straddling boundaries.
- The **token bucket peaked at 19, by design**: a full bucket of 10 plus 10 more refilled during that second. If you
  choose a token bucket, choose its capacity as the burst you're willing to accept.

Memory per client (in this simulation the window keys hadn't expired yet, because the TTLs run on the real clock
while the test uses simulated times; in real time a fixed-window or sliding-counter client holds one or two keys):

- The token bucket was the smallest (one hash, 71 bytes), and its size is independent of the limit.
- The sliding log costs one sorted-set member per allowed request in the window: 204 bytes at a limit of 10, but at a
  limit of 10,000 per hour it holds 10,000 members per client. That's the price of exactness.

## Which to use

| Algorithm | Exact? | Bursts | Memory per client | Use for |
|---|---|---|---|---|
| Fixed window | no, up to 2× at boundaries | uncontrolled | tiny | coarse quotas (per day), where 2× briefly is fine |
| Sliding log | yes | none above limit | O(limit) | low limits that must be exact (login attempts) |
| Sliding window counter | nearly | small overshoot | tiny | most API rate limits |
| Token bucket | to its definition | up to capacity | tiny | APIs that should allow short bursts |

The client-facing part matters too. Return HTTP **429** with a `Retry-After` header, and ideally
`RateLimit-Limit`/`RateLimit-Remaining`. The scripts can return those numbers instead of a 0/1 flag (see the
exercises).

## How It Actually Works

Correctness comes from atomicity. Without a script, a sliding log needs `ZREMRANGEBYSCORE`, `ZCARD`, then a
conditional `ZADD`, and two requests running those concurrently can both see 9 and both be admitted. Inside a script
on Redis's single command thread, the check and the update are one step, so the limit holds even with hundreds of
app servers.

Each script touches only the client's own keys, so in Redis Cluster each client's limiter runs entirely on one node
(the multi-key fixed and sliding-counter scripts build their key names from the `KEYS[1]` prefix; for strict Cluster
compliance, pass both window keys in `KEYS` with a hash tag like `rl:{ana}:...`). Load spreads naturally across nodes
by client.

Expiry keeps the state from accumulating. Fixed-window keys expire with their window, sliding-counter keys after two
windows, the log after one window of inactivity, and the bucket after the time it takes to refill completely, after
which a missing bucket is equivalent to a full one. A client that stops sending eventually costs nothing.

## Common mistakes

- **`INCR` and `EXPIRE` as two separate client calls.** A crash between them leaves a counter with no TTL that blocks
  the client forever. Do both in one script (or use `SET … NX EX` then `INCR`).
- **Check-then-act in the client** (`ZCARD`, then `ZADD`). Concurrent requests overshoot. Use a script.
- **Fixed windows for security limits.** An attacker gets double the attempts by timing bursts.
- **Sliding logs with large limits.** Memory grows with the limit, per client.
- **Mixing clocks.** Times from many app servers with clock skew make windows inconsistent. Use one time source.
- **Limiting by IP behind a proxy** without trusting the right header, which limits everyone as one client, or none.

## Exercise

1. Change the sliding-counter script to return `{allowed, remaining, retry_after_ms}` and use it to set HTTP headers
   in a small Flask or FastAPI endpoint.
2. Make the token bucket support a request `cost` (an expensive endpoint costs 5 tokens) and show a client spending its
   budget on 2 expensive calls.
3. Use `redis.call('TIME')` instead of a passed-in `now_ms`. How do you test it deterministically now?
4. Load-test the sliding log and the token bucket at 5,000 requests/s across 1,000 clients. Compare server CPU
   (`INFO cpu`) and memory. Which would you choose for a public API, and why?
