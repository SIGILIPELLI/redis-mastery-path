---
description: "Redis Pub/Sub, sharded channels and keyspace notifications, with limits measured: lost messages, a 32 MB subscriber cutoff, late expiry events."
---

# 07 · Pub/Sub, Sharded Pub/Sub & Keyspace Notifications

**Pub/Sub** is Redis's broadcast channel. Publishers send a message to a named channel, and every client
subscribed to that channel at that moment receives it. There's no storage, no history, no acknowledgement and no
retry. That sounds like a list of missing features, and for many jobs it is. For others, such as live dashboards,
chat typing indicators, cache invalidation signals and "config changed, reload" broadcasts, it's exactly right:
cheap, fast and forgettable. The trick is knowing which kind of job you have. This lesson measures the limits.

## Publish and subscribe

```python
import redis, threading, time
r = redis.Redis(decode_responses=True)
print("PUBLISH with nobody listening ->", r.publish("news", "hello?"))

got = []
def sub():
    c = redis.Redis(decode_responses=True)
    p = c.pubsub()
    p.subscribe("news"); p.psubscribe("orders:*")
    end = time.time() + 1.0
    while time.time() < end:
        m = p.get_message(timeout=0.1)
        if m: got.append((m["type"], m.get("pattern"), m["channel"], m["data"]))

th = threading.Thread(target=sub); th.start(); time.sleep(0.2)
print("PUBLISH news ->", r.publish("news", "redis 8.10 is out"))
print("PUBLISH orders:eu ->", r.publish("orders:eu", "order 17"))
print("PUBSUB CHANNELS:", r.pubsub_channels(), "NUMSUB news:", r.pubsub_numsub("news"),
      "NUMPAT:", r.pubsub_numpat())
th.join()
for g in got: print(g)
```

```text
PUBLISH with nobody listening -> 0
PUBLISH news -> 1
PUBLISH orders:eu -> 1
PUBSUB CHANNELS: ['news'] NUMSUB news: [('news', 1)] NUMPAT: 1
('subscribe', None, 'news', 1)
('psubscribe', None, 'orders:*', 2)
('message', None, 'news', 'redis 8.10 is out')
('pmessage', 'orders:*', 'orders:eu', 'order 17')
```

- **`PUBLISH` returns the number of clients that received the message.** The first message went to nobody, and
  it's gone. There's no way to get it back.
- `PSUBSCRIBE` subscribes to a glob pattern. Pattern messages arrive as `pmessage` and carry both the pattern and
  the actual channel.
- Subscription confirmations are messages too (`subscribe`, `psubscribe`). Their data is the connection's total
  subscription count.
- `PUBSUB CHANNELS` and `NUMSUB` show who is listening. They're useful for "is anyone watching this dashboard?".

In RESP2, a connection that has subscribed can't run normal commands (only more `SUBSCRIBE`/`UNSUBSCRIBE`/`PING`).
That's why the subscriber uses its own client here. In RESP3, messages arrive as out-of-band **push** frames, so
one connection can do both, but most libraries still give subscribers their own connection.

## What "fire and forget" really means

Pub/Sub delivers to whoever is connected **right now**:

- A subscriber that's restarting, or reconnecting after a network blip, misses everything published during the
  gap.
- A subscriber that subscribes a moment after the publisher sends misses that message. Startup ordering matters.
- There's no acknowledgement, so the publisher can't know whether a receiver processed the message, only that it
  was handed to a socket buffer.

If a missed message would be a bug (orders, payments, jobs), use a stream (Level 2 · 05–06). If it would merely be
stale for a moment (a "user is typing" indicator, a cache-invalidation hint backed by a TTL), Pub/Sub is fine.

## A slow subscriber gets disconnected

Redis pushes messages into each subscriber's **output buffer**. If a subscriber can't read as fast as messages are
published, that buffer grows on the server. To protect itself, Redis disconnects the subscriber at a configurable
limit:

```bash
redis-cli CONFIG GET client-output-buffer-limit
```

```text
normal 0 0 0 slave 268435456 67108864 60 pubsub 33554432 8388608 60
```

For `pubsub` clients: disconnect immediately at **32 MB**, or after **8 MB for 60 seconds**. To see it happen, this
script subscribes with a raw socket that never reads, then publishes 1 KB messages:

```python
import redis, socket, time
r = redis.Redis(decode_responses=True)
s = socket.create_connection(("127.0.0.1", 6379))          # a subscriber that never reads
s.sendall(b"*2\r\n$9\r\nSUBSCRIBE\r\n$5\r\nticks\r\n"); time.sleep(0.1)
payload = "x" * 1000
sent = 0
for i in range(200):
    for _ in range(500):
        n = r.publish("ticks", payload); sent += 1
    if n == 0:
        print(f"after {sent} messages PUBLISH reports 0 receivers: subscriber was disconnected")
        break
    if i % 20 == 0:
        info = [c for c in r.client_list() if c.get("sub") == "1"]
        if info: print(f"sent {sent:6}: subscriber omem={int(info[0]['omem'])/1e6:.1f} MB")
```

```text
sent    500: subscriber omem=0.0 MB
sent  10500: subscriber omem=10.2 MB
sent  20500: subscriber omem=20.6 MB
sent  30500: subscriber omem=31.0 MB
after 33000 messages PUBLISH reports 0 receivers: subscriber was disconnected
```

The server log explains why (trimmed):

```text
# Client id=2579 addr=127.0.0.1:63432 ... flags=P ... sub=1 ... omem=33565048 ... cmd=subscribe ...
  scheduled to be closed ASAP for overcoming of output buffer limits.
```

The subscriber was dropped once it had 33.5 MB queued. Its client library will reconnect and resubscribe, but
everything published in between is lost, and nothing tells it how much it missed. This is the right behaviour for
the server (otherwise one stuck client could consume all its memory), and it's one more reason why
must-not-lose data doesn't belong in Pub/Sub.

## Sharded Pub/Sub

In Redis Cluster, a classic `PUBLISH` is broadcast to **every node** so that subscribers anywhere receive it. That
cluster-wide traffic limits how far Pub/Sub can scale. **Sharded Pub/Sub** (Redis 7.0+) assigns each channel to a
hash slot like a key, so a message only travels to the node that owns the channel:

```python
p.ssubscribe("room:{42}:chat")                      # subscriber
r.spublish("room:{42}:chat", "hi")                  # publisher
```

```text
SPUBLISH -> 1
PUBLISH (non-sharded, same name) -> 0
PUBSUB SHARDCHANNELS: ['room:{42}:chat']
[('ssubscribe', 'room:{42}:chat', 1), ('smessage', 'room:{42}:chat', 'hi')]
```

Sharded and classic channels are separate namespaces. A plain `PUBLISH` to the same name reached nobody. On a
standalone server sharded Pub/Sub works the same way and just isn't distributed. Use it from the start if you
might move to Cluster. The `{42}` hash tag keeps related channels (and keys) on the same node, as Level 4 · 05
explains.

## Keyspace notifications

Redis can publish an event whenever a key changes. It's disabled by default because it costs CPU on every
write, and you enable the classes you need with `notify-keyspace-events`. `K` and `E` choose the channel style,
and the other letters choose event classes (`g` generic, `$` string, `h` hash, `x` expired, `e` evicted, `A` all
data types…).

With `KEA`, a subscriber to `__keyspace@0__:user:42` sees every operation on that key:

```python
r.config_set("notify-keyspace-events", "KEA")
# subscriber: p.psubscribe("__keyspace@0__:user:42")
r.hset("user:42", "name", "Ana"); r.hincrby("user:42", "logins", 1)
r.expire("user:42", 60); r.rename("user:42", "user:43")
```

```text
('__keyspace@0__:user:42', 'hset')
('__keyspace@0__:user:42', 'hincrby')
('__keyspace@0__:user:42', 'expire')
('__keyspace@0__:user:42', 'rename_from')
```

The `__keyevent@0__:<event>` style is the other way round: the channel names the event and the message is the key.
The most requested use is "run something when a key expires":

```python
r.config_set("notify-keyspace-events", "Ex")
# subscriber: p.psubscribe("__keyevent@0__:expired"), recording time.time() per key
for i in range(2000):
    p.set(f"tmp:{i}", 1, px=500)           # all due at t0 + 0.5 s, and nobody reads them
```

```text
expired events received: 2000 of 2000
delay after TTL deadline: median 130 ms, max 324 ms
```

All 2,000 events arrived, but **not at the deadline**. The median was 130 ms late and the worst 324 ms. The
`expired` event fires when Redis actually *deletes* the key, which happens either when someone accesses it or when
the background expiry cycle finds it (Level 3 · 02). Nobody touched these keys, so they waited for the sampler.
With millions of keys and a lower `hz`, the delay can be much longer. Never use expiry events as a precise timer,
and remember they're Pub/Sub: if your listener is down when a key expires, the event is lost. A delayed-job
sorted set (Level 1 · 08) is the reliable way to schedule work.

## How It Actually Works

The server keeps a dictionary from channel name to the list of subscribed clients, plus a list of pattern
subscriptions. `PUBLISH` looks up the channel, appends the message to each subscriber's output buffer, then checks
every pattern against the channel name, so the cost is O(subscribers + patterns). Many active patterns make every
publish more expensive. Nothing is stored. Once the message is copied into the output buffers, `PUBLISH` returns.

Output buffers are per client and drained by the event loop as the socket becomes writable. The
`client-output-buffer-limit` check runs whenever a reply is added. The hard limit closes the client "as soon as
possible", and the soft limit starts a timer that closes it if the buffer stays above the soft size for the given
number of seconds.

In Cluster, classic Pub/Sub messages are forwarded to all nodes over the cluster bus, which costs bandwidth
proportional to nodes × messages. Sharded channels are hashed to slots (`CRC16(channel) mod 16384`, the same as
keys), so `SPUBLISH` goes only to the node that owns the slot, and subscribers must connect to that node. Cluster
clients handle the routing.

Keyspace notifications are generated inside the command implementations through an internal notify call. If the
event class is enabled, it formats and publishes a message on the corresponding channel. That's why enabling
everything (`KEA`) costs a little on every write, even if no one subscribes.

## Common mistakes

- **Using Pub/Sub for work that must not be lost.** Offline, slow or late subscribers miss messages silently. Use
  streams.
- **Subscribing on a shared or pooled connection.** In RESP2 it can no longer run normal commands. Give
  subscribers a dedicated connection.
- **Ignoring `PUBLISH`'s return value.** 0 means nobody heard you.
- **Slow subscribers.** Do the work outside the receive loop (hand messages to a queue or thread), or you'll hit
  the output buffer limit and be disconnected.
- **Relying on `expired` events for timing.** They fired 130–324 ms late here and can be much later. They're also
  lost if no one is listening.
- **Enabling `KEA` in production "just in case".** Enable only the classes you consume.
- **Classic Pub/Sub at high volume in Cluster.** Use sharded Pub/Sub.

## Exercise

1. Build a tiny chat room: each client subscribes to `chat:<room>` and publishes typed lines. Kill and restart one
   client while others talk, and record what it missed. Then redesign it so a reconnecting client can catch up on
   the last 50 messages. What structure do you add?
2. Repeat the slow-subscriber experiment with the soft limit only (`CONFIG SET client-output-buffer-limit "pubsub
   0 1mb 5"`, lab only) and publish slowly enough to stay between 1 MB and 32 MB. How long until the disconnect?
3. Measure expired-event lateness for 100,000 keys with `hz 10` and then `hz 100`. Explain the difference.
4. Implement cache invalidation across three app processes: each keeps an in-process dict cache and subscribes to
   `invalidate`. A writer updates Redis and publishes the key. What happens if one process misses an invalidation
   message, and how does a TTL on the local cache limit the damage?
