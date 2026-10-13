---
description: "Reliable Redis job queues: a kill -9 test where BLPOP lost jobs and BLMOVE with a reaper lost none, plus retries, dead letters and delays."
---

# 07 · Reliable Job Queues

"Push a job, have a worker pop it" is ten lines of code and works perfectly until a worker dies mid-job. Then the
question is whether that job is **lost**, **retried**, or **retried forever**. This lesson kills workers with
`SIGKILL` while they process 2,000 jobs, and compares a naive list queue with a reliable one. Then it covers retries,
dead letters, delayed jobs, and when to use streams or an existing library instead.

## Two queues

`rq.py` (no relation to the RQ library) has both designs:

```python
"""Two list-based queues: naive (BLPOP) and reliable (BLMOVE + per-worker processing list + reaper)."""
import json, os, sys, time
import redis

r = redis.Redis(decode_responses=True)

def enqueue(q, job_id, payload):
    r.lpush(f"{q}:pending", json.dumps({"id": job_id, "payload": payload, "attempts": 0}))

def do_work(job):
    time.sleep(0.01)                                   # 10 ms of "work"
    r.sadd("done", job["id"])                          # the side effect (idempotent: a set)

def naive_worker(q):
    while True:
        item = r.brpop(f"{q}:pending", timeout=1)
        if item is None: return
        do_work(json.loads(item[1]))

def reliable_worker(q, name):
    proc = f"{q}:processing:{name}"
    r.hset(f"{q}:heartbeats", name, int(time.time()))
    while True:
        raw = r.blmove(f"{q}:pending", proc, timeout=1, src="RIGHT", dest="LEFT")
        if raw is None: return
        r.hset(f"{q}:heartbeats", name, int(time.time()))
        do_work(json.loads(raw))
        r.lrem(proc, 1, raw)                           # ack: remove exactly this item

REQUEUE = """
-- move everything from a dead worker's processing list back to pending (or to dead letters)
local moved, dead = 0, 0
while true do
  local raw = redis.call('RPOP', KEYS[1])
  if not raw then break end
  local job = cjson.decode(raw)
  job.attempts = job.attempts + 1
  if job.attempts > tonumber(ARGV[1]) then
    redis.call('LPUSH', KEYS[3], cjson.encode(job)); dead = dead + 1
  else
    redis.call('RPUSH', KEYS[2], cjson.encode(job)); moved = moved + 1   -- RPUSH: retried first
  end
end
return {moved, dead}
"""

def reap(q, stale_after=3, max_attempts=3):
    """Run periodically: requeue jobs held by workers whose heartbeat is older than stale_after seconds."""
    total = [0, 0]
    for name, beat in r.hgetall(f"{q}:heartbeats").items():
        if time.time() - int(beat) > stale_after:
            moved, dead = r.eval(REQUEUE, 3, f"{q}:processing:{name}", f"{q}:pending", f"{q}:dead", max_attempts)
            total[0] += moved; total[1] += dead
            r.hdel(f"{q}:heartbeats", name)
    return total

if __name__ == "__main__":
    kind, q = sys.argv[1], sys.argv[2]
    naive_worker(q) if kind == "naive" else reliable_worker(q, sys.argv[3])
```

**Naive:** `BRPOP` hands the job to the worker and removes it from Redis in the same step. If the worker dies before
finishing, nothing records that the job existed.

**Reliable:** `BLMOVE` atomically moves the job from `pending` into **this worker's own processing list**
(`q:processing:w3`), so at every moment the job is stored somewhere. After the work, `LREM` removes exactly that item
(the ack). Each worker updates a **heartbeat** in a hash. A **reaper** finds workers whose heartbeat is stale and moves
their processing list back to `pending` with a Lua script that also counts attempts and diverts jobs that exceeded the
limit to a **dead-letter list**.

## The crash test

Four workers process 2,000 jobs of 10 ms each. Every half second, one worker is killed with `SIGKILL` (no cleanup, no
`finally`, like an OOM kill or a lost machine) and a new one is started. That's five kills in all:

```python
import subprocess, sys, time, signal
import redis
from rq import enqueue, reap
r = redis.Redis(decode_responses=True)
PY = sys.executable
def run(kind):
    q = f"q:{kind}"
    for k in r.scan_iter(f"{q}*"): r.delete(k)
    r.delete("done")
    for i in range(2000): enqueue(q, i, {"n": i})
    procs = [subprocess.Popen([PY, "rq.py", kind, q, f"w{i}"]) for i in range(4)]
    for round_ in range(5):                       # every 0.5 s, kill -9 one worker and start a new one
        time.sleep(0.5)
        victim = procs.pop(0); victim.send_signal(signal.SIGKILL); victim.wait()
        procs.append(subprocess.Popen([PY, "rq.py", kind, q, f"w{4 + round_}"]))
    for p in procs: p.wait()
    requeued = [0, 0]
    if kind == "reliable":
        time.sleep(3.2)
        requeued = reap(q, stale_after=3)
        p = subprocess.Popen([PY, "rq.py", kind, q, "cleanup"]); p.wait()
    print(f"{kind:8}: 2000 jobs, 5 workers SIGKILLed -> done {r.scard('done')}, "
          f"lost {2000 - r.scard('done')}, requeued by reaper {requeued[0]}, dead-lettered {requeued[1]}")
run("naive")
run("reliable")
```

```text
naive   : 2000 jobs, 5 workers SIGKILLed -> done 1996, lost 4, requeued by reaper 0, dead-lettered 0
reliable: 2000 jobs, 5 workers SIGKILLed -> done 2000, lost 0, requeued by reaper 5, dead-lettered 0
```

- **Naive: 4 jobs lost** (an earlier run of the same test lost 5). Almost every kill caught a worker holding a job
  that existed nowhere else. Nothing reported an error. The jobs just never happened.
- **Reliable: 0 lost.** The reaper found 5 jobs stranded in dead workers' processing lists and requeued them. (In the
  earlier run it was 3, and the other kills landed between jobs, when the worker held nothing.)

There's a subtlety in the requeued count. A worker killed **after** `do_work` but **before** `LREM` had already applied the
side effect, and its job ran again after requeueing. This is at-least-once delivery, exactly like stream consumer
groups (Level 2 · 06). The handler here is idempotent (`SADD done <id>`), so the duplicate was harmless. Yours must be
too: use an idempotency key, a unique constraint, or a check-before-act on the side effect.

## Retries, backoff and dead letters

The reaper's script increments `attempts` each time a job is recovered, and after `max_attempts` it moves the job to
`q:dead` instead of back to `pending`. The same applies to jobs that fail with an exception rather than a crash. The
worker should catch the error, remove the job from its processing list, and either re-enqueue it with `attempts + 1`
or dead-letter it. Without that limit, a **poison job** (one that always crashes the worker) gets reaped, retried and
crashes the next worker, forever.

For **backoff**, don't push a failed job straight back to `pending`. Schedule it for later: put it in a delayed-jobs
sorted set scored by the retry time, and move due jobs to `pending` with a script (Level 1 · 08, Level 2 · 09
exercise):

```lua
-- KEYS[1] = delayed zset, KEYS[2] = pending list, ARGV[1] = now, ARGV[2] = batch size
local due = redis.call('ZRANGE', KEYS[1], '-inf', ARGV[1], 'BYSCORE', 'LIMIT', 0, ARGV[2])
for _, job in ipairs(due) do
  redis.call('ZREM', KEYS[1], job)
  redis.call('LPUSH', KEYS[2], job)
end
return #due
```

Run that every second. A typical policy is exponential backoff with jitter: retry after 1 s, 4 s, 16 s… plus a random
offset, so a burst of failures doesn't retry in lockstep.

## Lists or streams?

| | List + BLMOVE (this lesson) | Stream + consumer group (Level 2 · 06) |
|---|---|---|
| Crash recovery | your reaper + heartbeats | built in: PEL + `XAUTOCLAIM` |
| Delivery count | you track `attempts` in the payload | built in (`times_delivered`) |
| History / replay | gone after ack | kept until trimmed |
| Several independent consumers of the same jobs | no | yes, one group each |
| Ordering | FIFO | ID order |
| Simplicity | fewer concepts | more concepts, less code |

For new code, streams usually win: recovery and delivery counts come for free. List queues are still everywhere, and
the reliable pattern above is how the mature list-based libraries work underneath.

## Don't build it if a library fits

Mature libraries implement retries, scheduling, rate limits, priorities, dashboards and graceful shutdown on top of
Redis. Versions published at the time of writing: **RQ 2.12.0** and **Celery 5.6.3** (Python; Celery can also use
other brokers) and **BullMQ 6.3.14** (Node.js). None of them was run for this course, so check their documentation
for the delivery guarantees they actually give. Understanding the patterns in this lesson is what lets you read that
documentation critically and debug them when jobs go missing.

## How It Actually Works

The reliability comes from **never having a moment when a job exists only in a process's memory**. `BLMOVE` is one
command: the pop from `pending` and the push to `processing` happen together on Redis's single thread, so there's no
point at which a crash could drop the job between them. The ack (`LREM`) is the only way a job leaves the processing
list. A crash at any point leaves the job in Redis, either in `pending` or in a processing list.

What Redis can't know is whether a worker is dead or just slow. That's why the reaper needs **heartbeats** and a
staleness threshold. Like the lock lease in Level 3 · 05, the threshold is a guess. Set it too low, and a slow,
healthy worker has its job requeued and run twice. Set it too high, and jobs from crashed workers wait that long.
Workers should update their heartbeat during long jobs, not just between them.

`LREM key 1 raw` removes by **value**, so it scans the processing list. That's why each worker gets its own processing
list (usually holding one job) instead of sharing a big one. The requeue script does `RPOP`/`RPUSH` in a loop inside
one atomic call, so a reaper and a recovering worker can't fight over the same items.

All of this is only as durable as Redis itself. A job acknowledged as "enqueued" can still be lost if Redis crashes
before persisting it, or fails over before replicating it. Level 4 covers what `appendfsync` and replication give
you.

## Common mistakes

- **`BLPOP`/`BRPOP` for jobs that must not be lost.** The test lost roughly one per crash.
- **Non-idempotent handlers.** Every reliable queue redelivers sometimes.
- **No attempt limit.** Poison jobs loop forever.
- **A shared processing list** with `LREM` scanning thousands of items.
- **Heartbeats only between jobs.** Long jobs look dead and get duplicated.
- **Immediate retries.** Use delayed retries with backoff and jitter.
- **Reinventing a library** you could have used. Know the patterns, then use the tool.

## Exercise

1. Make one job in 500 raise an exception in `do_work`. Extend the worker to catch it, track attempts, and dead-letter
   after 3 failures, using delayed retries with exponential backoff. Verify the counts at the end.
2. Port the crash test to a stream with a consumer group and `XAUTOCLAIM`. Compare the amount of code and the number
   of duplicate executions.
3. Add priorities: `q:high:pending` and `q:low:pending`. How do you get priority with `BLMOVE`, which takes a single
   source? (Hint: `BLMPOP` can't move atomically. Consider a non-blocking `LMOVE` on high, then a blocking one on low
   with a short timeout.)
4. Measure throughput of the reliable queue with 8 workers and no-op jobs. Then batch: move up to 10 jobs per round
   trip with a script. What's the speedup?
