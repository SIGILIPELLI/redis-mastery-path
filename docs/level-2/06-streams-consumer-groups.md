---
description: "Stream consumer groups: XREADGROUP, the pending list, XACK, recovering a crashed worker with XAUTOCLAIM, dead letters, XACKDEL and XNACK."
---

# 06 · Stream Consumer Groups & Recovery

The last lesson's readers each saw every entry, which is right for fan-out ("every service hears about every
order"). Work distribution is different: you want each job handled by **one** of several workers, a job to be retried
if its worker crashes, and a way to find jobs that keep failing. **Consumer groups** add exactly that to a stream.
A group tracks which entries it has delivered, to which consumer, and which ones are still unacknowledged.

## Creating a group and reading from it

```text
127.0.0.1:6379> XGROUP CREATE jobs mailers $ MKSTREAM
OK
```

`$` means the group starts with entries added *after* now. Use `0` to have it process the stream's whole history.
`MKSTREAM` creates the stream if it doesn't exist yet. Add four jobs, with small explicit IDs to keep the output
readable:

```text
127.0.0.1:6379> XADD jobs 1-0 to ana@example.com tpl welcome
"1-0"
127.0.0.1:6379> XADD jobs 2-0 to ben@example.com tpl welcome
"2-0"
127.0.0.1:6379> XADD jobs 3-0 to cy@example.com tpl reset
"3-0"
127.0.0.1:6379> XADD jobs 4-0 to di@example.com tpl reset
"4-0"
```

Two consumers, `w1` and `w2`, read from the group. The special ID `>` means "entries never delivered to anyone in
this group":

```text
127.0.0.1:6379> XREADGROUP GROUP mailers w1 COUNT 2 STREAMS jobs >
1) 1) "jobs"
   2) 1) 1) "1-0"
         2) 1) "to"
            2) "ana@example.com"
            3) "tpl"
            4) "welcome"
      2) 1) "2-0"
         2) 1) "to"
            2) "ben@example.com"
            3) "tpl"
            4) "welcome"
127.0.0.1:6379> XREADGROUP GROUP mailers w2 COUNT 1 STREAMS jobs >
1) 1) "jobs"
   2) 1) 1) "3-0"
         2) 1) "to"
            2) "cy@example.com"
            3) "tpl"
            4) "reset"
```

Each entry went to exactly one consumer. Consumers are created automatically the first time they read. Name them
stably (hostname plus process index works) so a restarted worker can find its own unfinished work.

## Acknowledging and the pending entries list

Delivering an entry doesn't remove it. The group adds it to its **pending entries list** (PEL) until a consumer
acknowledges it with `XACK`:

```text
127.0.0.1:6379> XACK jobs mailers 1-0
(integer) 1
127.0.0.1:6379> XPENDING jobs mailers
1) (integer) 2
2) "2-0"
3) "3-0"
4) 1) 1) "w1"
      2) "1"
   2) 1) "w2"
      2) "1"
```

The summary says there are 2 pending entries, with IDs ranging from 2-0 to 3-0, one held by each consumer. The
extended form shows each pending entry's owner, how many milliseconds since it was last delivered, and **how many
times it has been delivered**:

```text
127.0.0.1:6379> XPENDING jobs mailers - + 10
1) 1) "2-0"
   2) "w1"
   3) (integer) 231
   4) (integer) 1
2) 1) "3-0"
   2) "w2"
   3) (integer) 169
   4) (integer) 1
```

This is **at-least-once delivery**. If a worker sends the email and crashes before `XACK`, the entry stays pending
and will be delivered again. Your handlers must tolerate seeing a job twice: check an idempotency key, use
`SET … NX`, or make the side effect naturally idempotent.

A consumer can re-read its own pending entries by passing an ID instead of `>`. `0` means "from the start of my
pending list". This is what a restarted worker should do first:

```text
127.0.0.1:6379> XREADGROUP GROUP mailers w1 STREAMS jobs 0
1) 1) "jobs"
   2) 1) 1) "2-0"
         2) 1) "to"
            2) "ben@example.com"
            3) "tpl"
            4) "welcome"
```

## Watching a group

```text
127.0.0.1:6379> XINFO GROUPS jobs
1)  1) "name"
    2) "mailers"
    3) "consumers"
    4) (integer) 2
    5) "pending"
    6) (integer) 2
    7) "last-delivered-id"
    8) "3-0"
    9) "entries-read"
   10) (integer) 3
   11) "lag"
   12) (integer) 1
```

- `pending` — delivered but not acknowledged. If it keeps growing, workers are failing or forgetting to `XACK`.
- `lag` — entries in the stream that haven't been delivered to this group yet (here only 4-0). If it keeps growing,
  you need more workers.

`XINFO CONSUMERS jobs mailers` shows each consumer's pending count and `idle` and `inactive` times. A consumer with
pending entries and a large idle time is probably dead.

## Recovering a dead consumer's work

If `w1` and `w2` crash, their entries sit in the PEL forever unless someone takes them over. `XAUTOCLAIM`
transfers entries that have been idle longer than a threshold to another consumer and returns them. Here it runs
after the two entries had been idle for more than a second:

```text
127.0.0.1:6379> XAUTOCLAIM jobs mailers w3 1000 0-0 COUNT 10
1) "0-0"
2) 1) 1) "2-0"
      2) 1) "to"
         2) "ben@example.com"
         3) "tpl"
         4) "welcome"
   2) 1) "3-0"
      2) 1) "to"
         2) "cy@example.com"
         3) "tpl"
         4) "reset"
3) (empty array)
127.0.0.1:6379> XPENDING jobs mailers - + 10
1) 1) "2-0"
   2) "w3"
   3) (integer) 61
   4) (integer) 3
2) 1) "3-0"
   2) "w3"
   3) (integer) 61
   4) (integer) 2
```

`w3` now owns both, and each claim **incremented the delivery count**: entry 2-0 has been delivered 3 times
(read by `w1`, re-read by `w1` with ID `0`, claimed by `w3`). The first element of the reply is a cursor for scanning
a large PEL in batches (`0-0` means "done"). The third lists IDs that were pending but have since been deleted from
the stream.

The idle threshold is a trade-off. Too short, and a slow but healthy worker has its jobs stolen and run twice. Too
long, and a crashed worker's jobs wait that long. Set it comfortably above your slowest legitimate job time.

## A worker loop with recovery and dead letters

This puts it together: three workers process 300 jobs. One worker "crashes" after 15 jobs, without acknowledging
what it had read. Job 123 is poison and always fails. Each worker first claims stale entries, then reads new ones,
and moves anything delivered more than three times to a dead-letter stream:

```python
import redis, threading, time, collections
r = redis.Redis(decode_responses=True)
r.delete("tasks", "tasks:dead")
r.xgroup_create("tasks", "workers", id="0", mkstream=True)
for i in range(300):
    r.xadd("tasks", {"n": i, "poison": int(i == 123)})

done = collections.Counter(); lock = threading.Lock()
MAX_DELIVERIES = 3

def handle(c, msg_id, fields):
    if fields["poison"] == "1":
        raise ValueError("cannot process")
    with lock: done[fields["n"]] += 1
    c.xack("tasks", "workers", msg_id)

def worker(consumer, crash_after=None):
    c = redis.Redis(decode_responses=True)
    handled = 0; deadline = time.time() + 4
    while time.time() < deadline:
        # 1) take over anything idle > 500 ms, including entries of dead consumers
        _, batch, _ = c.xautoclaim("tasks", "workers", consumer,
                                   min_idle_time=500, start_id="0-0", count=10)
        # 2) otherwise read new entries
        if not batch:
            res = c.xreadgroup("workers", consumer, {"tasks": ">"}, count=10, block=200)
            batch = res[0][1] if res else []
        for msg_id, fields in batch:
            if crash_after is not None and handled >= crash_after:
                return                                  # crash: stop without XACK
            info = c.xpending_range("tasks", "workers", min=msg_id, max=msg_id, count=1)
            if info and info[0]["times_delivered"] > MAX_DELIVERIES:
                c.xadd("tasks:dead", {"orig": msg_id, **fields})
                c.xack("tasks", "workers", msg_id)
                continue
            try:
                handle(c, msg_id, fields)
            except ValueError:
                pass                                    # stays pending; retried later
            handled += 1

ts = [threading.Thread(target=worker, args=("w1",)), threading.Thread(target=worker, args=("w2",)),
      threading.Thread(target=worker, args=("crashy", 15))]
for t in ts: t.start()
for t in ts: t.join()
print("distinct tasks done:", len(done), " processed more than once:", sum(1 for v in done.values() if v > 1))
print("pending now:", r.xpending("tasks", "workers")["pending"])
print("dead letters:", r.xrange("tasks:dead"))
for cinfo in r.xinfo_consumers("tasks", "workers"):
    print(cinfo["name"], "pending", cinfo["pending"])
```

```text
distinct tasks done: 299  processed more than once: 0
pending now: 0
dead letters: [('1791856809419-0', {'orig': '1791856807662-4', 'n': '123', 'poison': '1'})]
crashy pending 0
w1 pending 0
w2 pending 0
```

All 299 good jobs completed. The ones the crashed worker had taken were claimed and finished by the others, and the
poison message ended up in `tasks:dead` with its original ID, instead of being retried forever. Nothing ran twice
in this run, but that's luck of timing, not a guarantee. A slow `handle` crossing the 500 ms threshold would have
been claimed and run a second time.

## Newer helpers: XACKDEL and XNACK

**`XACKDEL`** (Redis 8.2) acknowledges and deletes in one command. With `ACKED`, an entry is only deleted once
every group has acknowledged it, which matters when several groups read the same stream:

```text
127.0.0.1:6379> XACKDEL jobs mailers ACKED IDS 1 2-0
1) (integer) 1
127.0.0.1:6379> XLEN jobs
(integer) 3
```

**`XNACK`** (Redis 8.8) lets a worker hand a message back **immediately** instead of letting it sit until the idle
threshold. What it did in experiments on 8.10.2:

```text
127.0.0.1:6379> XNACK nq g FAIL IDS 1 1-0
(integer) 1
127.0.0.1:6379> XPENDING nq g - + 10
1) 1) "1-0"
   2) ""
   3) (integer) -1
   4) (integer) 1
127.0.0.1:6379> XREADGROUP GROUP g c2 COUNT 1 STREAMS nq >
(nil)
127.0.0.1:6379> XAUTOCLAIM nq g c3 60000 0-0 COUNT 10
1) "0-0"
2) 1) 1) "1-0"
      2) 1) "job"
         2) "a"
3) (empty array)
```

- After `XNACK`, the entry stays in the PEL with **no owner** and idle `-1`.
- It is **not** redelivered by `XREADGROUP … >`, which only returns never-delivered entries. `XAUTOCLAIM` picked
  it up immediately, even with a 60-second idle threshold.
- The mode changes the delivery count: `FAIL` kept it at 1, `SILENT` decremented it (to 0 here, "don't count this
  attempt"), and `FATAL` set it to 9223372036854775807, so any `times_delivered > N` dead-letter check fires on the
  next claim.

These commands are new. Check `HELP XNACK` on your server and your client library's support before designing
around them. The `XAUTOCLAIM` loop above works on any Redis from 6.2.

## Several groups, one stream

Groups are independent. Adding an `audit` group that starts from `0` delivers the whole retained history to it,
regardless of what `mailers` has done:

```text
127.0.0.1:6379> XGROUP CREATE jobs audit 0
OK
127.0.0.1:6379> XREADGROUP GROUP audit a1 COUNT 10 STREAMS jobs >
1) 1) "jobs"
   2) 1) 1) "1-0"
   ...
      2) 1) "3-0"
   ...
      3) 1) "4-0"
```

(Entry 2-0 is missing because `XACKDEL` deleted it from the stream.) This is the fan-out-plus-work-queue
combination: each **group** gets every entry, and within a group each entry goes to **one** consumer.

## How It Actually Works

A consumer group is a small structure attached to the stream key: a `last-delivered-id`, an entries-read counter
(used to compute `lag`), a **PEL** (a radix tree from entry ID to a record of owner, last-delivery time and
delivery count), and a table of consumers, each with its own radix tree of the IDs it holds. `XREADGROUP … >`
walks the stream from `last-delivered-id`, advances it, and inserts each delivered ID into the group PEL and the
consumer's PEL. `XACK` removes IDs from both. Reading with an explicit ID iterates the consumer's PEL instead of the
stream, which is why it returns only *your* pending entries.

`XAUTOCLAIM` scans the group PEL from a cursor, and for each entry idle at least `min-idle-time` it reassigns
ownership, resets the idle time and increments the delivery count. All of these are single commands, so two
workers running `XAUTOCLAIM` at once can't both take the same entry. The second sees it as freshly delivered and
not idle.

The PEL entries refer to stream IDs, but the stream data is stored separately. Trimming the stream can delete
entries that are still pending. A claim then returns their IDs in the third ("deleted") part of the reply, with no
data. If you trim aggressively, make sure workers keep up, or check that list.

All group state is part of the key. It's saved in RDB and AOF and replicated, so after a failover the new primary
knows what each group has delivered and what is pending (subject to the replication lag of Level 4 · 03).

## Common mistakes

- **Never acknowledging.** The PEL grows forever, and memory and `XPENDING` with it.
- **Acknowledging before doing the work.** That's at-most-once: a crash loses the job.
- **No recovery loop.** Without `XAUTOCLAIM` (or `XCLAIM`), a crashed consumer's pending entries are stuck forever.
- **Idle threshold shorter than the slowest job.** Healthy work gets stolen and duplicated.
- **No dead-letter path.** One poison message gets retried forever by every worker.
- **Random consumer names per process start.** Old consumers pile up in `XINFO CONSUMERS` with orphaned entries.
  Use stable names, and delete retired consumers with `XGROUP DELCONSUMER` after reclaiming their entries.
- **Trimming entries that are still pending.**

## Exercise

1. Run the worker script with the idle threshold at 50 ms and make `handle` sleep a random 0–100 ms. Count how many
   jobs are processed more than once. Then make `handle` idempotent using `SET done:<n> 1 NX` and show the side
   effect happens exactly once.
2. Add a monitoring loop that prints `pending` and `lag` for the group every second while you start and stop workers.
   What does each number tell you?
3. Create two groups on one stream (`emails` and `analytics`) where `analytics` is slow. Trim with
   `XADD … MAXLEN ~ 1000`. What happens to analytics' pending entries when the stream is trimmed past them?
4. On Redis 8.8+, change the worker to call `XNACK … FAIL` instead of silently leaving failed jobs pending. Measure
   how much faster a failing job is retried, and use `FATAL` for errors you know are permanent.
