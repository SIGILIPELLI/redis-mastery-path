---
description: "Redis lists as queues, stacks and capped logs: push/pop, LTRIM, blocking pops with BLPOP, LMOVE for safer handoff, and which list operations are O(N)."
---

# 06 · Lists: Queues, Stacks & Blocking Pops

A Redis **list** is an ordered sequence of strings that you can push onto and pop from at either end. Those
end operations are cheap no matter how long the list is, which makes lists a natural fit for queues, stacks and
"most recent N" logs. Lists also have a feature that plain data structures don't: a client can **block** on an
empty list and wake up the instant something is pushed. That's what makes a list a working message queue.

## Push, range, pop

```text
127.0.0.1:6379> RPUSH jobs j1 j2 j3
(integer) 3
127.0.0.1:6379> LPUSH jobs j0
(integer) 4
127.0.0.1:6379> LRANGE jobs 0 -1
1) "j0"
2) "j1"
3) "j2"
4) "j3"
127.0.0.1:6379> LLEN jobs
(integer) 4
```

`L` and `R` mean left (head) and right (tail). Pushes return the new length. `LRANGE` takes inclusive indexes,
and negative indexes count from the end, so `0 -1` means "everything". Popping:

```text
127.0.0.1:6379> LPOP jobs
"j0"
127.0.0.1:6379> RPOP jobs
"j3"
127.0.0.1:6379> LPOP jobs 5
1) "j1"
2) "j2"
127.0.0.1:6379> LPOP jobs
(nil)
127.0.0.1:6379> EXISTS jobs
(integer) 0
```

`LPOP key count` pops up to `count` items at once (Redis 6.2+). Once the last item is popped, the key no longer
exists.

The two ends give you two disciplines:

- **Queue (FIFO):** `RPUSH` to enqueue, `LPOP` to dequeue (or `LPUSH` + `RPOP`).
- **Stack (LIFO):** `LPUSH` and `LPOP` on the same end.

## Capped lists: "the last N things"

Activity feeds, recent searches and recent log lines all want the newest N items and nothing older. Push, then
trim:

```text
127.0.0.1:6379> LPUSH recent:ana a1 a2 a3 a4 a5 a6
(integer) 6
127.0.0.1:6379> LTRIM recent:ana 0 2
OK
127.0.0.1:6379> LRANGE recent:ana 0 -1
1) "a6"
2) "a5"
3) "a4"
```

`LPUSH` with several values pushes them one at a time from left to right, so the *last* argument ends up at the
head. `LTRIM 0 2` keeps only indexes 0 to 2. In an application you'd send `LPUSH` and `LTRIM` together in a
pipeline or transaction on every write. The list then never grows beyond N, and trimming one extra element each
time is cheap.

## Working in the middle (carefully)

```text
127.0.0.1:6379> LINDEX recent:ana -1
"a4"
127.0.0.1:6379> LPOS recent:ana a5
(integer) 1
127.0.0.1:6379> LINSERT recent:ana BEFORE a5 x
(integer) 4
127.0.0.1:6379> LSET recent:ana 0 a6!
OK
127.0.0.1:6379> LRANGE recent:ana 0 -1
1) "a6!"
2) "x"
3) "a5"
4) "a4"
127.0.0.1:6379> LREM recent:ana 0 x
(integer) 1
```

These commands work, but they all have to *find* a position. `LREM key count value` removes up to `count`
occurrences (0 means all of them), and it has to scan the list to find them. How much does that matter? This
script builds a 2-million-item list and times reads at different positions:

```python
import redis, time
r = redis.Redis()
p = r.pipeline(transaction=False)
for i in range(0, 2_000_000, 1000):
    p.rpush("big:list", *[f"item-{j}" for j in range(i, i + 1000)])
p.execute()
print("LLEN", r.llen("big:list"), "encoding", r.object("encoding", "big:list"))
for idx in (0, -1, 1_000_000):
    t = time.perf_counter()
    for _ in range(200): r.lindex("big:list", idx)
    print(f"LINDEX {idx:>9}: {(time.perf_counter()-t)/200*1e6:7.1f} us per call")
t = time.perf_counter()
for _ in range(200): r.lpos("big:list", "item-1999999")
print(f"LPOS last item:   {(time.perf_counter()-t)/200*1e6:7.1f} us per call")
```

```text
LLEN 2000000 encoding b'quicklist'
LINDEX         0:   115.1 us per call
LINDEX        -1:   111.3 us per call
LINDEX   1000000:   114.8 us per call
LPOS last item:   23403.8 us per call
```

Two lessons are in those numbers. First, `LINDEX` in the middle of 2 million items was no slower than at the ends.
Each call was dominated by the client round trip, because the list is stored in chunks that know their own size
and the server skips whole chunks to reach an index (see below). The documentation calls `LINDEX` O(N), and it is,
but N here is effectively the number of chunks. Second, **`LPOS` had to compare every element**. It took 23 ms,
and the server could do nothing else during that time. Searching a list by value is a linear scan. If you need
"is X in here?" or "remove X" on large collections, you want a set or a sorted set.

## Blocking pops: a real work queue

A worker that polls an empty queue in a loop with `LPOP` either burns CPU and network or adds latency by
sleeping. `BLPOP` and `BRPOP` solve that by blocking until there's something to pop, or until a timeout runs out:

```text
127.0.0.1:6379> BLPOP q:nothing 1
(nil)
(1.06s)
```

With a timeout of 1 second and nothing pushed, it returned `(nil)` after a second. Here are three workers blocked
on the same queue while a producer pushes six tasks, 100 ms apart:

```python
import redis, threading, time
r = redis.Redis(decode_responses=True)
got = []

def worker(name):
    c = redis.Redis(decode_responses=True)       # one connection per blocked worker
    while True:
        item = c.blpop(["tasks"], timeout=2)
        if item is None: return
        _, payload = item
        task, sent = payload.split("|")
        got.append((name, task, (time.time() - float(sent)) * 1000))

ws = [threading.Thread(target=worker, args=(f"w{i}",)) for i in range(3)]
for w in ws: w.start()
time.sleep(0.3)
for i in range(6):
    r.rpush("tasks", f"task{i}|{time.time()}"); time.sleep(0.1)
for w in ws: w.join()
for name, task, ms in got: print(f"{task} -> {name}  waited {ms:.2f} ms")
```

```text
task0 -> w2  waited 2.04 ms
task1 -> w1  waited 2.08 ms
task2 -> w0  waited 1.00 ms
task3 -> w2  waited 1.31 ms
task4 -> w1  waited 1.27 ms
task5 -> w0  waited 1.40 ms
```

Each task went to exactly one worker, about a millisecond after it was pushed (most of that is Python thread
scheduling in this test). The workers took turns: Redis serves blocked clients **in the order they started
waiting**, and a worker that just finished a task rejoins the back of the line.

A few rules for blocking commands:

- A blocked connection can't be used for anything else. Give each blocking worker its own connection, not one
  shared from a pool that other code also uses.
- `BLPOP` accepts several keys and pops from the first non-empty one in the order listed, which gives you
  simple priority queues: `BLPOP q:high q:normal q:low 5`.
- Use a finite timeout and loop, so a worker notices shutdown signals and dead connections.
- `LMPOP` / `BLMPOP` (Redis 7.0+) pop several items from the first non-empty of several lists:

```text
127.0.0.1:6379> LMPOP 2 q:empty q:pending LEFT COUNT 2
1) "q:pending"
2) 1) "t2"
   2) "t3"
```

## The problem with pop: what if the worker dies?

`BLPOP` removes the task from Redis the moment it's delivered. If the worker crashes before finishing, the task
is gone. Nobody will ever retry it. `LMOVE` (and its blocking twin `BLMOVE`) pops from one list and pushes to
another **atomically**, so a task is never "in flight" without being stored somewhere:

```text
127.0.0.1:6379> RPUSH q:pending t1 t2 t3
(integer) 3
127.0.0.1:6379> LMOVE q:pending q:processing LEFT RIGHT
"t1"
127.0.0.1:6379> LRANGE q:processing 0 -1
1) "t1"
127.0.0.1:6379> LREM q:processing 1 t1
(integer) 1
```

The worker moves a task into `q:processing`, does the work, then removes it with `LREM`. If the worker dies, the
task is still sitting in `q:processing`, where a reaper process can find it and push it back. Level 3 · 07 builds
that reliable queue properly (one processing list per worker, visibility timeouts, retries, dead letters) and
compares it with Redis Streams, which have acknowledgement built in.

## How It Actually Works

Lists use an encoding called **quicklist** for anything but the smallest lists: a doubly linked list of nodes, each
holding a **listpack**, which is a compact block of several elements. Small lists are a single listpack with no
linked list at all. (`q:pending` printed `(nil)` for its encoding above only because the key had already been
emptied and deleted.)

```bash
redis-cli CONFIG GET 'list-*'
```

```text
list-max-ziplist-size
-2
list-max-listpack-size
-2
list-compress-depth
0
```

(`list-max-ziplist-size` is the older name of the same setting, kept as an alias.) A negative
`list-max-listpack-size` limits each node by bytes: `-2` means 8 KB per node. `list-compress-depth` can
LZF-compress interior nodes and leave `depth` nodes uncompressed at each end, which helps for long lists where you
only touch the ends. It's off (0) by default.

This layout is why push and pop at the ends are O(1): they touch only the head or tail node. It's also why
`LINDEX` in the middle was fast. Each node records how many elements it holds, so the server walks node by node,
skipping hundreds of short elements at a time, and then scans within one node. `LPOS` and `LREM` have to compare
*values*, though, so they visit every element.

Blocking works without any thread waiting. When `BLPOP` finds all its keys empty, Redis marks the client as
blocked, records which keys it's waiting for in a dictionary (key → queue of blocked clients), and simply stops
reading its commands. When a later command pushes to one of those keys, Redis notes that the key is "ready". After
that command finishes, it serves the waiting clients in FIFO order, popping on their behalf and writing the
replies. The timeout is checked by the periodic server cron. All of it happens on the same single command thread.

## Common mistakes

- **Treating a list as a set.** `LPOS`, `LREM` and "check before push" are O(N) scans and race-prone. Use a set
  or sorted set for membership.
- **Unbounded lists.** A log or feed list with no `LTRIM` grows until memory runs out. Pair every push with a trim,
  or use a stream with `MAXLEN` (Level 2 · 05).
- **`BLPOP` on a shared pooled connection.** It blocks that connection, and other code waiting for it stalls.
- **Timeout 0 (block forever)** in workers you need to stop gracefully. Use a few seconds and loop.
- **Losing tasks on crash** with plain `BLPOP`. Use `BLMOVE` into a processing list, or streams with consumer
  groups.
- **Reading a whole huge list with `LRANGE 0 -1`.** Page through it with bounded ranges.

## Exercise

1. Build a "recent searches" feature: keep the last 10 distinct searches per user, newest first. Searching for
   something already in the list should move it to the front rather than duplicate it. Which commands do you
   need, and why should they run in one `MULTI` block?
2. Write a priority worker that serves `q:high` before `q:low` with one `BLPOP`. Push 5 low and 5 high tasks
   *before* starting it, and show that all the high ones come out first.
3. Kill a worker (Ctrl-C) between `BLMOVE` and `LREM`. Write a reaper that finds stranded tasks in the processing
   list and moves them back to pending. What goes wrong if the reaper runs while a healthy worker is still
   working on a task?
4. Repeat the `LPOS` timing with lists of 100k, 1M and 4M items and plot the time against the length.
