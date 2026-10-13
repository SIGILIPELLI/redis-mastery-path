---
description: "redis-py 8 and node-redis 6 in practice: shared clients and pools, RESP3 defaults, pipelines vs transactions, and retries that can double-apply."
---

# 09 · Clients: redis-py & node-redis

Everything so far ran through `redis-cli`. Applications use a client library, and most production incidents
blamed on "Redis" are really about how the client was configured: a new connection per request, a pool that's too
small, a timeout that's too long, or an automatic retry that ran a write twice. This lesson uses the two most
common libraries and measures each of those behaviours:

- **redis-py 8.1.0** (Python, with the optional `hiredis` parser 3.4.2): `pip install redis hiredis`
- **node-redis 6.3.0** (Node.js 26): `npm install redis`

(**ioredis**, at 6.0.0 on npm when this was written, is the other popular Node client. It wasn't used here.)

## redis-py: the basics

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)
print(r.hgetall("user:1"))
print(r.zrange("lb", 0, -1, withscores=True))

raw = redis.Redis()
print(raw.get("greeting"))
```

```text
{'name': 'Ana', 'plan': 'pro', 'logins': '4'}
[('ana', 10.0), ('ben', 25.0)]
b'hello'
```

Without `decode_responses=True` you get `bytes`, which is correct for binary values but noisy for text. Decide
per client; you can keep one decoding client and one raw client. Hash values come back as strings (`'4'`) because
Redis stores strings, while sorted-set scores come back as floats because the library knows that command returns
numbers.

Errors from the server raise exceptions:

```python
try:
    r.incr("user:1")
except redis.ResponseError as e:
    print("ResponseError:", e)
```

```text
ResponseError: WRONGTYPE Operation against a key holding the wrong kind of value
```

### What redis-py 8 sends when it connects

`MONITOR` captured the session above:

```text
"HELLO" "3"
"CLIENT" "SETINFO" "LIB-NAME" "redis-py"
"CLIENT" "SETINFO" "LIB-VER" "8.1.0"
"HGETALL" "user:1"
"ZRANGE" "lb" "0" "-1" "WITHSCORES"
```

**redis-py 8 negotiates RESP3 by default.** In the library source, `DEFAULT_RESP_VERSION = 3`, together with a
`legacy_responses=True` default that converts results back to the shapes older code expects. That's why
`hgetall` still gives a dict and `zrange` gives a list of tuples. If you upgrade from redis-py 5 or 6 and see
different result types, or if you run against a proxy or an old server that doesn't support `HELLO`, the
`protocol=2` and `legacy_responses` arguments are where to look. Check your installed version's changelog.
These defaults are recent and may change again.

## One client, many requests

Every `redis.Redis()` object owns a **connection pool**. The cost of getting that wrong:

```python
import redis, time
N = 2000

t = time.perf_counter()
for i in range(N):
    c = redis.Redis(); c.get("greeting"); c.close()
print(f"new client per request : {(time.perf_counter()-t)/N*1e6:7.1f} us/request")

shared = redis.Redis()
t = time.perf_counter()
for i in range(N):
    shared.get("greeting")
print(f"one shared client      : {(time.perf_counter()-t)/N*1e6:7.1f} us/request")
```

```text
new client per request :  1940.6 us/request
one shared client      :   122.9 us/request
```

A new client per request was **16 times slower** on loopback. Each one opened a TCP connection, sent `HELLO` and
`CLIENT SETINFO`, ran one `GET` and then tore everything down. Over a real network, and with TLS or `AUTH` added,
the difference grows. It also churns connections on the server and in the OS. Create the client **once** per
process (at module level or in your app's startup) and share it. The redis-py client is thread-safe: each command
borrows a connection from the pool and returns it.

The pool's defaults, printed from that client (trimmed):

```text
socket_timeout=5, socket_connect_timeout=5, retry=<redis.retry.Retry ...>, socket_keepalive=True,
protocol=None, legacy_responses=True, host=localhost, port=6379
```

## Timeouts and retries: what actually happens

With nothing listening on port 6411:

```python
dead = redis.Redis(port=6411)
t = time.perf_counter()
try:
    dead.ping()
except redis.ConnectionError as e:
    print(f"ConnectionError after {time.perf_counter()-t:.2f}s: {str(e)[:70]}")
```

```text
ConnectionError after 4.43s: Error 61 connecting to localhost:6411. Connection refused.
```

"Connection refused" is instant at the TCP level, yet the call took **4.4 seconds** to fail. The client's default
retry policy is behind that:

```python
r0 = redis.Redis()
retry = r0.get_retry()
print(retry._retries, type(retry._backoff).__name__, retry._supported_errors)
```

```text
10 ExponentialWithJitterBackoff (<class 'redis.exceptions.ConnectionError'>, <class 'redis.exceptions.TimeoutError'>, <class 'TimeoutError'>)
```

redis-py 8.1 retries **10 times with exponential backoff** on connection errors **and on timeouts**. For reads
that's convenient. For writes it's a hazard. This test blocks the server for 1 second from another connection (a
stand-in for any slow command or a brief network stall), while a client with a 0.2 s socket timeout runs one
`INCR`:

```python
import redis, threading, time
r0 = redis.Redis(); r0.delete("k")
fast = redis.Redis(socket_timeout=0.2); fast.ping()

def block():
    redis.Redis().execute_command("DEBUG", "SLEEP", "1")   # lab server only

th = threading.Thread(target=block); th.start(); time.sleep(0.05)
print("INCR returned", fast.incr("k"))
th.join(); time.sleep(0.3)
print("k is now", r0.get("k"))
```

```text
INCR returned 2
k is now b'2'
```

**One `INCR` call, applied twice.** The first `INCR` reached the server and sat in the queue behind the sleep. The
client timed out, reconnected and sent `INCR` again. When the server woke up it executed both. Repeat with
retries disabled:

```python
noretry = redis.Redis(socket_timeout=0.2,
                      retry=redis.retry.Retry(redis.backoff.NoBackoff(), 0))
```

```text
no-retry client: TimeoutError Timeout reading from socket
k is now b'1'
```

Now the caller gets an error, **and the increment still happened**. That's the fundamental ambiguity of a timeout:
you don't know whether the command ran. Practical consequences:

- Choose the retry policy deliberately. Retries are safe for idempotent commands (`GET`, `SET` with a fixed value,
  `SADD`, `ZADD GT`) and unsafe for `INCR`, `LPUSH`, `RPOPLPUSH` and the like.
- Make important writes idempotent: `SET payment:<id> … NX`, `SADD processed <id>`, or a Lua script that checks a
  request ID before applying a change (Level 2 · 09).
- Set `socket_timeout` from your latency budget. A 5-second default on a request path means a stalled Redis turns
  into 5-second page loads, before any retries.

## Pipelines and transactions in redis-py

```python
pipe = r.pipeline()                       # transaction=True by default
pipe.incr("visits").expire("visits", 60).get("visits")
print(pipe.execute())

pipe = r.pipeline(transaction=False)
pipe.incr("visits").get("visits")
print(pipe.execute())
```

```text
[1, True, '1']
[2, '2']
```

What went over the wire:

```text
"MULTI"
"INCRBY" "visits" "1"
"EXPIRE" "visits" "60"
"GET" "visits"
"EXEC"
"INCRBY" "visits" "1"
"GET" "visits"
```

`r.pipeline()` **wraps the batch in `MULTI`/`EXEC` by default**, so it's a transaction as well as a pipeline.
That's usually harmless, but it isn't free, and in Redis Cluster a transaction requires all keys to be in one
slot. Use `transaction=False` when you just want fewer round trips. Level 2 · 08 explains what `MULTI` does and
doesn't guarantee.

## asyncio

`redis.asyncio` has the same API with `await`:

```python
import asyncio, time
import redis.asyncio as aredis

async def main():
    r = aredis.Redis(decode_responses=True)
    await r.set("greeting", "hello")
    t = time.perf_counter()
    for i in range(1000):
        await r.get("greeting")
    print(f"1000 awaited one by one : {(time.perf_counter()-t)*1000:5.0f} ms")
    try:
        await asyncio.gather(*(r.get("greeting") for _ in range(1000)))
    except Exception as e:
        print("gather of 1000 on default pool:", type(e).__name__, e)
    await r.aclose()

    pool = aredis.BlockingConnectionPool(max_connections=20, timeout=5)
    r = aredis.Redis(connection_pool=pool)
    t = time.perf_counter()
    await asyncio.gather(*(r.get("greeting") for _ in range(1000)))
    print(f"1000 gathered, 20 conns : {(time.perf_counter()-t)*1000:5.0f} ms")
    t = time.perf_counter()
    async with r.pipeline(transaction=False) as p:
        for _ in range(1000): p.get("greeting")
        await p.execute()
    print(f"1000 in one pipeline    : {(time.perf_counter()-t)*1000:5.0f} ms")
    await r.aclose()

asyncio.run(main())
```

```text
1000 awaited one by one :   139 ms
gather of 1000 on default pool: MaxConnectionsError Too many connections
1000 gathered, 20 conns :    75 ms
1000 in one pipeline    :     6 ms
```

Firing 1,000 commands at once on the default pool **failed**. Each concurrent command needs its own connection,
and the default pool caps out at 100 (`max_connections = max_connections or 100` in the source) and raises rather
than waits. A `BlockingConnectionPool` makes callers queue for a connection instead, and 20 connections handled
the burst in 75 ms. The pipeline did the same work in 6 ms over one connection. Under load, pick a pool size on
purpose and prefer pipelines for batches.

## node-redis 6

```javascript
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
client.on('error', (err) => console.error('redis error:', err.message));
await client.connect();

await client.set('greeting', 'hello', { EX: 60 });
console.log(await client.get('greeting'));
await client.hSet('user:1', { name: 'Ana', plan: 'pro', logins: 3 });
console.log(await client.hGetAll('user:1'));
console.log(await client.hIncrBy('user:1', 'logins', 1));
await client.zAdd('lb', [{ score: 10, value: 'ana' }, { score: 25, value: 'ben' }]);
console.log(await client.zRangeWithScores('lb', 0, -1, { REV: true }));
console.log(await client.get('nope'));

const replies = await client.multi()
  .incr('counter')
  .expire('counter', 60)
  .get('counter')
  .exec();
console.log('multi:', replies);

const [a, b, c] = await Promise.all([client.incr('x'), client.incr('x'), client.incr('x')]);
console.log('parallel incr:', a, b, c);

try { await client.incr('greeting'); }
catch (e) { console.log('error class:', e.constructor.name, '-', e.message); }

console.log(await client.sendCommand(['OBJECT', 'ENCODING', 'user:1']));
await client.close();
```

```text
hello
{ name: 'Ana', plan: 'pro', logins: '3' }
4
[ { value: 'ben', score: 25 }, { value: 'ana', score: 10 } ]
null
multi: [ 1, 1, '1' ]
parallel incr: 1 2 3
error class: SimpleError - ERR value is not an integer or out of range
listpack
```

Things to notice:

- Method names are camelCase versions of commands (`hSet`, `zRangeWithScores`), and options are objects
  (`{ EX: 60 }`). `sendCommand` runs anything the typed API doesn't cover.
- **Always attach an `'error'` listener.** In Node, an `EventEmitter` that emits `'error'` with no listener
  crashes the process. A Redis restart would take your app down with it.
- One client is one connection, and commands issued in the same tick are **automatically pipelined**.
  `MONITOR` showed the two `INCR`s of a `Promise.all` arriving 5 µs apart:

```text
"HELLO" "3"
"CLIENT" "SETINFO" "LIB-VER" "6.3.0"
"CLIENT" "SETINFO" "LIB-NAME" "node-redis"
"INCR" "x"
"INCR" "y"
```

node-redis 6 also opened with `HELLO 3`. Both libraries now speak RESP3 by default and normalize the results for
you. The `RESP: 3` client option exposes the RESP3 types more directly; with it, `zScore` returned a JavaScript
`number`.

Because a node-redis client is a single multiplexed connection, a **blocking** command (`BLPOP`, `XREAD BLOCK`)
blocks every other command on that client. Use a separate client (`client.duplicate()`) for blocking work and for
Pub/Sub subscribers.

## How It Actually Works

A client library is three layers. A **connection** layer opens a socket, performs the handshake (`HELLO` with
protocol version and optional `AUTH`, `CLIENT SETINFO`, `SELECT` if you chose a database) and reconnects when the
socket breaks. A **protocol** layer encodes commands into RESP and parses replies. That's pure-Python in
redis-py unless `hiredis` is installed, in which case a C parser does it much faster for large replies. A
**command** layer maps methods to commands and converts reply types into language types: bytes or str, dicts for
maps, floats for scores.

Pools differ in where concurrency comes from. redis-py's sync client takes a connection from the pool for each
command (or pipeline) and returns it afterwards, so N threads need up to N connections. The asyncio client does
the same per concurrent coroutine, which is why `gather` over 1,000 coroutines asked for 1,000 connections.
node-redis instead writes every command from the whole process into one socket and matches replies by order.
That works because RESP replies always come back in request order on a connection, and it's also why a blocking
command stalls everything behind it.

Retries live in the connection layer. When a read times out, the library can't know whether the server received
the command, executed it, or never saw it. All it can do is close the socket (the stream may hold a half-read
reply) and try again on a fresh connection. The server, meanwhile, still has the original command in its
query buffer and executes it. Exactly-once execution over an unreliable network is not something a client
library can give you. Idempotent operations are how you get it.

## Common mistakes

- **A client per request** (or per function call). Create one per process and share it.
- **Unbounded concurrency on a bounded pool.** Use a blocking pool, a semaphore, or pipelines.
- **Default retries on non-idempotent writes.** They can double-apply. Configure retries per client, or make the
  writes idempotent.
- **Assuming a timeout means "didn't happen".** It means "don't know".
- **Forgetting the `'error'` listener in node-redis.** An unhandled error event crashes Node.
- **Running `BLPOP`/`SUBSCRIBE` on the shared client.** Use a dedicated connection.
- **Mixing `decode_responses=True` with binary values.** Decoding a JPEG as UTF-8 fails. Keep a raw client for
  binary data.

## Exercise

1. Write a small Flask or FastAPI endpoint that increments a page counter. Load-test it (`ab -n 2000 -c 20`) once
   with a module-level client and once creating a client inside the handler. Compare throughput and
   `connected_clients` in `INFO clients` during the test.
2. Reproduce the double-`INCR` with your own retry settings, then make the operation idempotent: the caller sends a
   request ID, and the increment applies only if `SET seen:<request-id> 1 NX EX 600` succeeds. Do both commands in
   one `MULTI`. Does that fully solve it? Why or why not?
3. In node-redis, start a `BLPOP` with a 5-second timeout on a client and immediately `GET` a key on the same
   client. How long does the `GET` take? Fix it with `client.duplicate()`.
4. Find out which protocol your production client library negotiates (check `CLIENT LIST` for `resp=`), and what
   its default socket timeout and retry policy are.
