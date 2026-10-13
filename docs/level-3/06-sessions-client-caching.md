---
description: "Redis session stores done right (random IDs, idle and absolute expiry, rotation, logout everywhere) and client-side caching with CLIENT TRACKING."
---

# 06 · Sessions & Client-Side Caching

This lesson covers two patterns that both treat Redis as the shared memory of a fleet of app servers. **Sessions**
keep per-user state that any server can read, so a user's next request can land anywhere. **Client-side caching**
goes the other way: app servers keep copies of hot Redis keys in their own memory, and Redis tells them when a copy
goes stale.

## A session store

Sessions are small, read on almost every request, written occasionally, and supposed to vanish after inactivity.
Redis with TTLs fits that exactly. Here's a complete store:

```python
import json, secrets, time
import redis

r = redis.Redis(decode_responses=True)
IDLE_TTL = 30 * 60            # sliding: 30 minutes of inactivity
ABSOLUTE_TTL = 12 * 3600      # hard cap: re-login after 12 hours no matter what

def create_session(user_id, data):
    sid = secrets.token_urlsafe(32)                       # 256 bits from the OS CSPRNG
    now = int(time.time())
    p = r.pipeline(transaction=True)
    p.set(f"sess:{sid}", json.dumps({"uid": user_id, "created": now, **data}), ex=IDLE_TTL)
    p.sadd(f"user:{user_id}:sessions", sid)
    p.expire(f"user:{user_id}:sessions", ABSOLUTE_TTL)
    p.execute()
    return sid

def get_session(sid):
    raw = r.getex(f"sess:{sid}", ex=IDLE_TTL)             # read + slide the idle timeout atomically
    if raw is None:
        return None
    s = json.loads(raw)
    if time.time() - s["created"] > ABSOLUTE_TTL:
        destroy_session(sid); return None
    return s

def rotate_session(sid):
    """Issue a new id for the same session (call on login / privilege change) to defeat fixation."""
    s = get_session(sid)
    if s is None: return None
    new = secrets.token_urlsafe(32)
    p = r.pipeline(transaction=True)
    p.rename(f"sess:{sid}", f"sess:{new}")
    p.srem(f"user:{s['uid']}:sessions", sid); p.sadd(f"user:{s['uid']}:sessions", new)
    p.execute()
    return new

def destroy_session(sid):
    raw = r.getdel(f"sess:{sid}")
    if raw: r.srem(f"user:{json.loads(raw)['uid']}:sessions", sid)

def logout_everywhere(user_id):
    sids = r.smembers(f"user:{user_id}:sessions")
    if sids: r.unlink(*[f"sess:{s}" for s in sids])
    r.unlink(f"user:{user_id}:sessions")
    return len(sids)

if __name__ == "__main__":
    a = create_session(42, {"role": "user"}); b = create_session(42, {"role": "user"})
    print("session id length:", len(a), " ttl:", r.ttl(f"sess:{a}"))
    print("get:", get_session(a))
    a2 = rotate_session(a)
    print("old id after rotate:", get_session(a), "| new id works:", get_session(a2) is not None)
    print("logout everywhere removed", logout_everywhere(42), "sessions; b now:", get_session(b))
    print("memory of one session:", r.memory_usage(f"sess:{create_session(7, {'role': 'admin', 'cart': [1, 2, 3]})}"), "bytes")
```

```text
session id length: 43  ttl: 1800
get: {'uid': 42, 'created': 1791886636, 'role': 'user'}
old id after rotate: None | new id works: True
logout everywhere removed 2 sessions; b now: None
memory of one session: 157 bytes
```

The decisions that matter:

- **Session IDs come from `secrets`**, which draws from the OS's cryptographic generator. 32 bytes is 256 bits of
  randomness, 43 characters in URL-safe base64. Anything guessable, such as sequential IDs, timestamps, `uuid1`, or
  `random` instead of `secrets`, lets an attacker hijack sessions.
- **Two expiries.** The *idle* timeout slides forward on every request: `GETEX … EX` reads and extends in one
  atomic step. The *absolute* lifetime (12 hours here) is checked against the creation time stored in the
  session. Without it, an active stolen session lives forever.
- **Rotation on login and on privilege change.** If an attacker can plant a session ID in a victim's browser before
  login (session fixation), rotating the ID at login makes the planted one useless. `RENAME` keeps the value and the
  TTL, and the old ID stops working, as the output shows.
- **A per-user index** (`user:42:sessions`) makes "log out of all devices" and "show my active sessions" cheap. It's a
  small set per user with its own TTL. It can hold IDs of sessions that already expired, and that's harmless: the
  `UNLINK` of a missing key is a no-op.
- **Store little.** 157 bytes per session here. Keep the source of truth for carts and profiles in your database,
  and put only identifiers and small flags in the session.

Should the session be a hash or a JSON string? A hash lets you update one field (`HSET sess:… last_page /x`) without
rewriting the whole value, and since 7.4 fields can expire on their own (Level 1 · 05). A JSON string is simpler when
the session is read and written whole. Either works. Pick one per application.

## Client-side caching

Even a 0.1 ms round trip adds up when every request reads the same feature flags or config ten times.
**Client-side caching** keeps those values in the application's own memory. The classic problem with local caches
is invalidation: how does server 7 know that server 3 just changed a flag? Since Redis 6, the server tells it.

### What the server does: CLIENT TRACKING

A RESP3 connection turns on tracking. Redis then remembers which keys this connection has read, and when any client
modifies one of them, it sends an **`invalidate` push message**. This is a raw-socket session:

```python
send(b"HELLO 3\r\n")
print(send(b"CLIENT TRACKING ON\r\n"))
print(send(b"GET feature:flags\r\n"))
# another client: SET feature:flags '{"new_checkout": false}'
print(s.recv(65536))
```

```text
b'+OK\r\n'
b'$22\r\n{"new_checkout": true}\r\n'
b'>2\r\n$10\r\ninvalidate\r\n*1\r\n$13\r\nfeature:flags\r\n'
```

The `>` prefix is a RESP3 push frame: `["invalidate", ["feature:flags"]]`. When another client modified the key a
**second** time, before this connection had read it again, nothing more arrived:

```text
nothing received: tracking is one-shot until the key is read again
```

Tracking is **one-shot**. After an invalidation, the server forgets that this client cares about the key until the
client reads it again. That keeps server memory bounded: Redis only tracks keys that clients actually hold.

### What the library does

redis-py 8 implements the client side. Pass `protocol=3` and a `CacheConfig`, and `GET`s are served from a local cache
that is invalidated by those push messages:

```python
import time, redis
from redis.cache import CacheConfig
writer = redis.Redis(decode_responses=True)
writer.set("feature:flags", '{"new_checkout": false}')
cached = redis.Redis(protocol=3, decode_responses=True, cache_config=CacheConfig())
plain = redis.Redis(decode_responses=True)
N = 20000
for name, c in (("plain client", plain), ("client-side cache", cached)):
    c.get("feature:flags")
    t = time.perf_counter()
    for _ in range(N): c.get("feature:flags")
    print(f"{name:18}: {(time.perf_counter()-t)/N*1e6:6.1f} us per GET")
print("cached value     :", cached.get("feature:flags"))
writer.set("feature:flags", '{"new_checkout": true}')
time.sleep(0.05)
print("after another client's SET:", cached.get("feature:flags"))
print("tracking clients:", [c["flags"] for c in writer.client_list() if "t" in c["flags"]])
print("INFO tracking_total_keys:", writer.info("stats").get("tracking_total_keys"))
```

```text
plain client      :  113.7 us per GET
client-side cache :   45.9 us per GET
cached value     : {"new_checkout": false}
after another client's SET: {"new_checkout": true}
tracking clients: ['t']
INFO tracking_total_keys: 1
```

- Cached reads were about **2.5× faster** than round trips over loopback. That's less than you might expect from a
  local dictionary lookup. In redis-py 8.1, each cached read still goes through the connection's `send_command`, which
  first processes any pending invalidation messages from the socket (`_process_pending_invalidations` in the source). Over a real network, where a round trip
  is 0.5 ms or more, the gain is much larger.
- After another client changed the key, the next read returned the **new** value. The invalidation arrived and the
  local copy was dropped.
- The server shows the tracking flag `t` on the connection and 1 tracked key.

### Caveats

- **Staleness window.** Between the write and the invalidation reaching the client, the client can serve the old
  value. Usually that's microseconds to milliseconds, but it's not zero, and a lost connection loses invalidations
  (libraries flush the cache on reconnect for this reason).
- **Memory.** Every cached value lives in every app process. Cache small, hot, rarely changing keys (flags, config,
  rate-limit rules), not your whole dataset.
- **Server cost.** Tracking needs a table of key → tracking clients. **Broadcasting mode** (`CLIENT TRACKING ON BCAST
  PREFIX flags:`) avoids per-key memory on the server by notifying every client about any change under a prefix.
  That's useful when many clients cache the same small set of keys.

## How It Actually Works

Sessions rely on nothing more exotic than per-key expiry and atomic multi-command operations. `GETEX` updates the
expiry in the same command as the read, so a request can't see a session that expires between "read" and "extend".
`RENAME` is atomic and moves the TTL with the key, so rotation never leaves a moment where neither ID works.

For tracking, Redis keeps a radix tree (the *tracking table*) that maps each key to the set of client IDs that read it
while tracking was on. Every write goes through the same "key modified" hook that `WATCH` uses (Level 2 · 08). The
hook looks the key up in the table, sends an `invalidate` push to each client, and removes the entry. That's why
tracking is one-shot. `tracking-table-max-keys` caps the table, and when it's full, Redis evicts entries by sending
invalidations early, so clients refetch. In BCAST mode there's no per-key table, just a list of prefixes and the
clients subscribed to them, checked on every write.

In RESP2, invalidations can't share the data connection, so tracking uses `REDIRECT <client-id>` to send them to a
separate connection subscribed to the `__redis__:invalidate` channel. RESP3's push frames are what make the
single-connection design practical, and why redis-py requires `protocol=3` for its cache.

## Common mistakes

- **Guessable session IDs.** Use a CSPRNG and at least 128 bits.
- **Only a sliding expiry.** A stolen session that's used regularly never expires. Add an absolute lifetime.
- **Not rotating the ID at login.** It leaves the door open to session fixation.
- **Storing large objects in sessions.** Every request pays to fetch them.
- **Client-side caching large or fast-changing keys.** It wastes memory everywhere, and invalidation traffic can exceed
  the savings.
- **Assuming the local cache is never stale.** It's eventually consistent, with a small window.

## Exercise

1. Store sessions as hashes instead of JSON, with `last_seen` updated on each request (`HSET` + `EXPIRE` in one
   pipeline). Compare memory per session and the cost of a request.
2. Add an "active sessions" page: list each of a user's sessions with its creation time and remaining idle TTL, and let
   them revoke one. Make sure expired IDs in the index are cleaned up.
3. Run the client-side caching benchmark against a Redis server on another machine (or add 1 ms of latency with a
   proxy) and report the speedup.
4. Turn on BCAST tracking for a `flags:` prefix with two cached clients, update five flags, and count the invalidation
   messages each client receives.
