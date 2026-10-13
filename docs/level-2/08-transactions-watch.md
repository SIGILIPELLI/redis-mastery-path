---
description: "What MULTI/EXEC guarantees and what it doesn't (no rollback), and optimistic locking with WATCH: 2,815 overdrafts without it, none with it."
---

# 08 · Transactions: MULTI, EXEC & WATCH

Every single Redis command is atomic. Real operations often span several commands, though. Moving money is
"decrement one balance *and* increment another", and reserving stock is "check the quantity, *then* decrement it".
Redis has two tools for making multi-command operations safe. This lesson covers the first, **transactions**
(`MULTI`/`EXEC`) with **optimistic locking** (`WATCH`). The next lesson covers the second, server-side scripts.
Both are easy to misunderstand. A Redis transaction isn't a SQL transaction.

## MULTI and EXEC

```text
127.0.0.1:6379> SET acct:ana 100
OK
127.0.0.1:6379> SET acct:ben 50
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> DECRBY acct:ana 30
QUEUED
127.0.0.1:6379(TX)> INCRBY acct:ben 30
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (integer) 70
2) (integer) 80
```

After `MULTI`, commands aren't executed; they're **queued**. `EXEC` runs the whole queue in one go, and no other
client's command can run in between. Nobody ever sees Ana debited but Ben not yet credited. `DISCARD` throws the
queue away:

```text
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET acct:ana 0
QUEUED
127.0.0.1:6379(TX)> DISCARD
OK
127.0.0.1:6379> GET acct:ana
"75"
```

Because commands are only queued, **you can't read a value inside a transaction and use it later in the same
transaction**. `GET` inside `MULTI` returns `QUEUED`, not the value. The reply comes back with `EXEC`, when it's too
late to make decisions. That's the gap `WATCH` fills.

## Errors: abort before, no rollback after

There are two kinds of error, and they behave differently. A command that's **wrong when queued** (unknown command,
wrong number of arguments) makes `EXEC` refuse the whole transaction:

```text
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCRBY acct:ana 10
QUEUED
127.0.0.1:6379(TX)> INCRBYY acct:ben 10
(error) ERR unknown command 'INCRBYY', with args beginning with: 'acct:ben' '10'
127.0.0.1:6379(TX)> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> MGET acct:ana acct:ben
1) "70"
2) "80"
```

Nothing ran. But a command that's well-formed and **fails when it executes**, such as `INCR` on a non-number or a
command on the wrong type, fails **alone**, and the rest of the transaction still runs:

```text
127.0.0.1:6379> SET note hello
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCRBY acct:ana 5
QUEUED
127.0.0.1:6379(TX)> INCR note
QUEUED
127.0.0.1:6379(TX)> INCRBY acct:ben 5
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (integer) 75
2) (error) ERR value is not an integer or out of range
3) (integer) 85
127.0.0.1:6379> MGET acct:ana acct:ben
1) "75"
2) "85"
```

**There is no rollback.** The first and third commands took effect even though the second failed. Redis's
position is that runtime errors in a transaction are programming errors (using the wrong type) that should be
caught in development, and rollback would cost complexity and speed. In practice you need to check every element
of `EXEC`'s reply for errors, and design transactions so a partial failure is impossible or harmless.

So a Redis transaction gives you:

- **Isolation and all-or-nothing scheduling.** The queued commands run back to back with nothing in between.
- **Not** rollback on runtime errors.
- **Not** conditional logic, because you can't branch on values read inside the transaction.

## The check-then-act problem

A transfer must not overdraw an account. Here's the obvious code: check the balance, then do the transfer in a
transaction:

```python
def transfer_naive(c, src, dst, amt):
    if int(c.get(src)) >= amt:                      # check...
        p = c.pipeline(transaction=True)            # ...then act (atomically, but too late)
        p.decrby(src, amt); p.incrby(dst, amt)
        if p.execute()[0] < 0: bump("overdrafts")
        return True
    return False
```

The transfer itself is atomic, but the **check** isn't part of it. Between `GET` and `EXEC`, another client can spend
the same money. With 16 threads making 2,000 random transfers each among 5 accounts that start at 100:

```text
naive  total=500 transfers=23759 overdrafts=2815 watch-retries=0
```

The total stayed 500 (money was never created or destroyed, because each transfer was atomic), but **2,815
transfers drove an account negative**. Every one of them passed the balance check.

## WATCH: optimistic locking

`WATCH key` asks the server to remember that key. If **any** client modifies it before this connection's `EXEC`,
the `EXEC` does nothing and returns null. You read, decide, queue, and if someone interfered you start over:

```python
def transfer_watch(c, src, dst, amt):
    with c.pipeline() as p:
        while True:
            try:
                p.watch(src)                          # 1. watch
                if int(p.get(src)) < amt:             # 2. read and decide (runs immediately)
                    p.unwatch(); return False
                p.multi()                             # 3. queue
                p.decrby(src, amt); p.incrby(dst, amt)
                if p.execute()[0] < 0: bump("overdrafts")   # 4. EXEC: fails if src changed
                return True
            except redis.WatchError:
                bump("retries")                       # someone changed src: try again
```

```text
WATCH  total=500 transfers=22143 overdrafts=0 watch-retries=57341
```

**Zero overdrafts.** The cost shows in the last column: 57,341 retries for 22,143 successful transfers. With only 5
hot accounts and 16 threads, most attempts found that someone had touched their account in the meantime and had to
start again. Optimistic locking is excellent when conflicts are rare and wasteful when they're common. The next
lesson runs the same benchmark with a Lua script, which never retries.

Here is the abort happening by hand, using two connections:

```python
a = redis.Redis(decode_responses=True); b = redis.Redis(decode_responses=True)
a.set("acct:ana", 100)
p = a.pipeline(transaction=True)
p.watch("acct:ana")
bal = int(p.get("acct:ana"))
b.incrby("acct:ana", 1)               # someone else changes it
p.multi(); p.set("acct:ana", bal - 10)
try:
    p.execute()
except redis.WatchError as e:
    print("WatchError:", e)
print("balance:", a.get("acct:ana"))
```

```text
WatchError: Watched variable changed.
balance: 101
```

The stale write (`100 - 10 = 90`) was rejected, and the other client's increment survived.

Rules for `WATCH`:

- It's **per connection**. In redis-py, `pipeline.watch()` puts that pipeline's connection into "immediate mode" so
  reads run right away. Keep the whole read-decide-`EXEC` sequence on that one pipeline object.
- Any modification counts, including by the same value, by expiry or eviction of the key, and by `FLUSHALL`. A
  `WATCH`ed key that expires mid-transaction aborts the `EXEC`.
- `EXEC`, `DISCARD` and `UNWATCH` all clear the watches. So does the connection closing.
- Bound your retry loop in production (with a maximum number of attempts plus jitter), or a hot key can make a
  client spin.

## Transactions vs pipelines

The two get confused because redis-py's `pipeline()` does both by default (Level 1 · 09):

| | Pipeline (`transaction=False`) | `MULTI`/`EXEC` |
|---|---|---|
| Fewer round trips | yes | yes, if sent pipelined |
| Other clients can interleave | yes, between commands | no |
| All queued commands run | yes | yes (unless `WATCH` fails or queuing failed) |
| Rollback on error | no | no |

In **Redis Cluster**, all keys in a transaction must be in the same hash slot, which is why multi-key designs use
hash tags like `acct:{ana}` (Level 4 · 05).

## How It Actually Works

Each connection has a flag for "in `MULTI`" and a queue. While the flag is set, the command dispatcher checks each
command for arity and existence (the source of `EXECABORT`) and appends it to the queue instead of executing it.
`EXEC` then calls each queued command in a loop, inside one iteration of the event loop, which is why nothing else
interleaves. Since there's no undo log, a command that fails just records its error in the reply array, and the
loop carries on.

`WATCH` adds the connection to a per-database dictionary of watched keys → watching clients. Every command that
modifies a key calls an internal "signal modified key" hook, which marks all clients watching that key as dirty
(`CLIENT_DIRTY_CAS`). Expiry, eviction and flushes go through the same hook. `EXEC` checks the flag first, and if it's
set, it discards the queue and returns a null reply, which redis-py turns into `WatchError`. That's optimistic
concurrency control: no lock is held, and a conflict is detected at commit time.

When the server persists with AOF or replicates, `EXEC` wraps the propagated commands in `MULTI`/`EXEC`, so replicas
and a reloaded AOF apply the transaction as a unit too. (Level 4 · 02 shows what happens if a crash leaves half a
transaction at the end of the AOF.)

## Common mistakes

- **Expecting rollback.** Runtime errors don't undo earlier commands. Check every reply.
- **Reading inside `MULTI`.** You get `QUEUED`, not a value. Read before `MULTI`, under `WATCH`.
- **Check-then-act without `WATCH`.** The check isn't protected. That produced 2,815 overdrafts here.
- **`WATCH` on one connection, `EXEC` on another** (easy to do with a connection pool and separate client calls).
  The watch does nothing.
- **Unbounded retry loops on hot keys.** Cap attempts, add backoff, or switch to a Lua script.
- **Using `WATCH` under heavy contention.** It's correct but wasteful. The benchmark spent 2.6 failed attempts per
  success.
- **Cross-slot transactions in Cluster.** Co-locate the keys with hash tags.

## Exercise

1. Implement "reserve N units of product P for order O": check `stock:P ≥ N`, then decrement stock and add `O` to
   `reservations:P` atomically. Write it with `WATCH`, run 20 concurrent buyers against stock 50 with random N from
   1 to 5, and verify stock never goes negative and reservations add up.
2. Rerun the transfer benchmark with 50 accounts instead of 5. How do the retry count and throughput change? Explain
   why.
3. Show that a `WATCH`ed key expiring (set a 1-second TTL, `WATCH`, sleep 1.5 s, `MULTI`/`EXEC`) aborts the
   transaction.
4. Write a transaction where a runtime error in the middle would leave data inconsistent, then redesign it so that
   can't happen (hint: validate types first, or make each command independently safe).
