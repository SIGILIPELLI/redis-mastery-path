---
description: "Redis locks done carefully: SET NX PX with tokens, safe release, why a paused holder breaks any lease, fencing tokens, and the Redlock debate."
---

# 05 · Distributed Locks & Their Limits

A distributed lock makes sure only one process at a time does something: runs the nightly invoice job, rebuilds a
cache entry, or processes a particular user's payment. Redis makes a lock look trivial, one `SET … NX`. It *is*
easy to get the basic version right, and it's even easier to believe that version gives a guarantee it can't give.
This lesson builds a correct lock, then breaks it with a paused process, which is the failure every lease-based lock
shares, and fixes the damage with **fencing tokens**.

## A correct basic lock

```python
import threading, time, uuid
import redis
r = redis.Redis(decode_responses=True)

RELEASE = r.register_script("""
if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) end
return 0
""")
EXTEND = r.register_script("""
if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('PEXPIRE', KEYS[1], ARGV[2]) end
return 0
""")

def acquire(name, ttl_ms, wait_s=5):
    token = uuid.uuid4().hex
    deadline = time.time() + wait_s
    while time.time() < deadline:
        if r.set(f"lock:{name}", token, nx=True, px=ttl_ms):
            fence = r.incr(f"fence:{name}")          # monotonically increasing fencing token
            return token, fence
        time.sleep(0.01)
    return None, None

def release(name, token):
    return RELEASE(keys=[f"lock:{name}"], args=[token]) == 1
```

Three properties make this the correct basic form:

1. **`NX`** makes acquisition atomic: only one `SET` can create the key.
2. **`PX ttl`** makes it a **lease**. If the holder crashes, the lock frees itself. Without a TTL, one crashed process
   deadlocks the job forever.
3. **A random token**, checked on release. Releasing with a plain `DEL` is a classic bug:

```python
r.set("lock:report", "token-A", nx=True, px=300)    # A takes the lock
time.sleep(0.4)                                      # ...A stalls; its lease expires
print("B acquires:", r.set("lock:report", "token-B", nx=True, px=5000))
r.delete("lock:report")                              # A finishes and "releases" with DEL
print("C acquires while B still works:", r.set("lock:report", "token-C", nx=True, px=5000))
```

```text
B acquires: True
C acquires while B still works: True
```

A's `DEL` removed **B's** lock, so C walked in while B was still working. With the token check, A's release is a
no-op. Since **Redis 8.4**, `DELEX key IFEQ token` does the same check without a script:

```text
DELEX with wrong token: 0  with right token: 1
```

`EXTEND` (a token-checked `PEXPIRE`) lets a long job renew its lease periodically, a "watchdog". Renew well before
expiry, and stop working if a renewal fails, because that means you've lost the lock.

Under contention it does what it should. Twenty threads each do 50 read-modify-write increments of a counter,
protected by the lock:

```text
20 threads x 50 locked increments -> balance 1000 in 0.76s
```

That's exactly 1,000, with no lost updates. (For a counter you'd just use `INCR`. The lock is standing in for work
that can't be expressed as one Redis command.)

## How a correct lock still fails: the paused holder

A lease has a fundamental weakness. The lock expires on Redis's clock, but the holder decides it's safe to act on
**its own** view of time. If the holder pauses (a long garbage-collection pause, the VM being descheduled, a slow
network call, the laptop lid closing) for longer than the lease, then:

1. The lease expires while A is paused.
2. B acquires the lock and does its work.
3. A wakes up, still believing it holds the lock, and does its work too.

This script reproduces it. Client A takes a 500 ms lease and stalls for 1 s. B arrives at 0.6 s, after A's lease
expired. Both write to a "storage service":

```python
storage = {"value": None, "max_fence": 0, "writes": []}
def storage_write(value, fence=None):
    if fence is not None:
        if fence < storage["max_fence"]:
            storage["writes"].append(("REJECTED", value, fence)); return False
        storage["max_fence"] = fence
    storage["value"] = value; storage["writes"].append(("ok", value, fence)); return True

def scenario(use_fencing):
    r.delete("lock:invoice", "fence:invoice")
    storage.update(value=None, max_fence=0, writes=[])
    def client(name, pause):
        token, fence = acquire("invoice", ttl_ms=500)
        time.sleep(pause)                              # e.g. a GC pause or a slow network call
        storage_write(f"written by {name}", fence if use_fencing else None)
        print(f"  {name}: release -> {release('invoice', token)}")
    a = threading.Thread(target=client, args=("A", 1.0))   # A stalls longer than its 500 ms lease
    a.start(); time.sleep(0.6)
    b = threading.Thread(target=client, args=("B", 0.1))   # B gets the lock after A's lease expired
    b.start(); a.join(); b.join()
    for w in storage["writes"]: print("  storage:", w)
    print("  final value:", storage["value"])
```

```text
Without fencing:
  B: release -> True
  A: release -> False
  storage: ('ok', 'written by B', None)
  storage: ('ok', 'written by A', None)
  final value: written by A
```

**Both clients wrote**, and A, whose lock had expired, wrote **last**, overwriting B's newer result. A's release
returned `False`, so the token check stopped A from deleting B's lock, but by then the damage was done. No choice of
TTL removes this: a pause can always be longer than the lease.

## Fencing tokens

The fix has to live in the **resource being protected**, not in the lock. Every time the lock is granted, issue a
number that only ever increases (here `INCR fence:invoice`). The holder sends it with every write, and the storage
rejects any write carrying a token lower than one it has already seen:

```text
With fencing tokens:
  B: release -> True
  A: release -> False
  storage: ('ok', 'written by B', 2)
  storage: ('REJECTED', 'written by A', 1)
  final value: written by B
```

A's stale write (token 1) arrived after B's (token 2) and was refused. The resource can do this with a version column
(`UPDATE … WHERE fence < :token`), a conditional write in object storage, or the same check in a Lua script if the
protected data is in Redis itself.

When you can't add fencing because the protected system doesn't support conditional writes (sending an email,
charging a card through a third-party API), make the operation **idempotent** instead (an idempotency key on the
charge), and treat the lock as an efficiency measure that usually prevents duplicate work, not as a correctness
guarantee.

## Failover and Redlock

There's a second failure mode. Redis replication is asynchronous (Level 4 · 03). If the primary acknowledges your
`SET NX` and crashes before the write reaches a replica, the promoted replica has no lock, and another client can
acquire it.

**Redlock**, the algorithm described in the Redis documentation, addresses that by acquiring the lock on a majority
of N independent Redis primaries (typically 5) within a time budget, with validity reduced by the elapsed acquisition
time. It survives the loss of a minority of nodes. It's also the subject of a well-known disagreement. Martin
Kleppmann's 2016 essay "How to do distributed locking" argued that Redlock depends on timing assumptions
(bounded pauses, network delays and clock drift) that real systems violate, and that it provides no fencing tokens,
so it's unsafe when correctness depends on the lock. Salvatore Sanfilippo (antirez) replied defending it under its
stated assumptions. Both essays are worth reading in full. A fair summary for practice:

- For **efficiency** (avoid doing the same work twice; occasional duplicates are harmless), a single-instance lock
  like the one above is enough, and Redlock adds little.
- For **correctness** (duplicates corrupt data), no lease-based lock is sufficient by itself, including Redlock. Use
  fencing tokens or idempotency at the resource, or a coordination system built on consensus (etcd, ZooKeeper),
  which provides fencing via revision numbers.

Redlock wasn't run for this course. It needs several independent primaries and client-library support, and
the point above holds regardless of the implementation.

## How It Actually Works

`SET key token NX PX ttl` is a single command, so the check "does the key exist?" and the creation can't be split by
another client. The expiry is stored as an absolute time and enforced by Redis's own clock (Level 1 · 03), and the
holder's clock isn't involved at all. That's the root of the paused-holder problem: Redis's notion of "lock held"
and the client's notion diverge whenever the client stops observing time.

The token release is a compare-and-delete, which must be atomic. Hence the script, or `DELEX IFEQ` from 8.4. A
`GET`, compare in the client, then `DEL` would have its own race, with the lease expiring between the `GET` and the
`DEL`.

Fencing works because it moves the decision to the only place that sees writes in their actual order: the resource.
`INCR` on the fence key gives a total order of lock grants. Even if two holders overlap in real time, the resource
accepts their writes only in grant order, and it rejects the older holder once a newer one has written.

## Common mistakes

- **No TTL on the lock.** One crash deadlocks forever.
- **Releasing with plain `DEL`.** You can free someone else's lock.
- **A TTL shorter than the work**, without a watchdog. The lock silently expires mid-job.
- **Believing the lock guarantees mutual exclusion.** Pauses, failovers and clock jumps break it. Fence or make
  operations idempotent.
- **Locks for things Redis can do atomically.** `INCR`, `SET NX`, `ZADD GT` or a script are simpler and faster.
- **Spinning acquisition without backoff or a deadline.** Add jitter and give up eventually.

## Exercise

1. Add a watchdog thread that extends the lease every `ttl/3` while work runs, and stops the work if an extension
   returns 0. Rerun the paused-holder scenario with A's pause inside the work loop. Does the watchdog help when the
   *whole process* is paused? Why not?
2. Protect a row in SQLite with fencing: add a `fence` column and update with
   `UPDATE … SET value=?, fence=? WHERE id=? AND fence < ?`. Reproduce the stale-writer scenario against it.
3. Compare a lock-protected read-modify-write with a single Lua script doing the same update, for 20 threads × 500
   updates. Report the time for each.
4. Read both sides of the Redlock debate and write a one-paragraph decision for a job scheduler in your own system:
   which lock, what TTL, and what protects correctness.
