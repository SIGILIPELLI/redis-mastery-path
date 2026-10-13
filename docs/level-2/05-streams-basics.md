---
description: "Redis Streams as a log: entry IDs, XADD, XRANGE and XREAD, why '$' loses messages between reads, bounded trimming, and idempotent XADD."
---

# 05 · Streams: An Append-Only Log

Lists can act as queues, but a popped item is gone: only one consumer sees it, and there's no history. A
**stream** is different. It's an **append-only log** of entries, each with a unique, time-ordered ID and a small set
of field/value pairs. Readers don't remove anything. They remember where they are and read onward, so many
independent readers can process the same log, a new reader can replay history, and you can query by time range.
If you've used Kafka, a stream is a single partition of a topic living in one Redis key. This lesson covers the log
itself. The next adds **consumer groups**, which turn it into a work queue with acknowledgements.

## Appending and reading ranges

```text
127.0.0.1:6379> XADD orders * user ana total 1999 status created
"1791856584765-0"
127.0.0.1:6379> XADD orders * user ben total 4500 status created
"1791856584826-0"
127.0.0.1:6379> XADD orders * user ana total 1200 status created
"1791856584882-0"
127.0.0.1:6379> XLEN orders
(integer) 3
```

`*` asks Redis to generate the ID. It has the form `<milliseconds>-<sequence>`: the server's Unix time in ms and a
counter for entries in the same millisecond. Reading:

```text
127.0.0.1:6379> XRANGE orders - + COUNT 1
1) 1) "1791856584765-0"
   2) 1) "user"
      2) "ana"
      3) "total"
      4) "1999"
      5) "status"
      6) "created"
127.0.0.1:6379> XREVRANGE orders + - COUNT 1
1) 1) "1791856584882-0"
   2) 1) "user"
      2) "ana"
      3) "total"
      4) "1200"
      5) "status"
      6) "created"
```

`-` and `+` are the lowest and highest possible IDs. Because IDs are times, you can also ask for "everything between
10:00 and 10:05" by passing millisecond timestamps as the bounds (`XRANGE orders 1791856500000 1791856800000`).

To **page** through a stream, take the last ID of one page and start the next page *after* it with `(`:

```python
first = r.xrange("events", count=2)
nxt = r.xrange("events", min="(" + first[-1][0], count=2)
print("page1", [e[0] for e in first], "page2", [e[0] for e in nxt])
```

```text
page1 ['1791856637080-0', '1791856637080-1'] page2 ['1791856637080-2', '1791856637080-3']
```

(These four entries were added within the same millisecond, so only the sequence part differs.)

## IDs only go forward

```text
127.0.0.1:6379> XADD orders 1-1 user old total 1
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
127.0.0.1:6379> XADD orders 9999999999999-0 user future total 1
"9999999999999-0"
127.0.0.1:6379> XADD orders * user after-future total 1
"9999999999999-1"
```

Every new ID must be greater than the last one. If you supply an ID far in the future (here, the year 2286), every
auto-generated ID after it becomes "that millisecond, next sequence number". The stream keeps working, but the
time component of its IDs is now meaningless. The same mechanism protects you when the server clock jumps
**backwards**: Redis keeps the last ID's millisecond and increments the sequence, so ordering is never violated.
Use explicit IDs only when you are importing data that already has its own monotonically increasing IDs.

`XDEL` removes individual entries, but streams are designed to be appended to and trimmed from the front. Most
applications never `XDEL`.

## Reading new entries: XREAD

`XREAD` returns entries **after** the ID you pass, and it can block until something arrives:

```text
XREAD COUNT 10 BLOCK 5000 STREAMS orders 1791856584882-0
```

The special ID `$` means "the current last entry", i.e. only things added from now on. It's tempting to write a
reader loop that always passes `$`. Here are two readers watching a stream that receives 10 entries, 50 ms apart.
Each reader spends 80 ms "working" between reads. One passes `$` every time, and the other passes the last ID it
saw:

```python
def reader_dollar():
    c = redis.Redis(decode_responses=True)
    end = time.time() + 1.2
    while time.time() < end:
        res = c.xread({"ticks": "$"}, block=200, count=10)
        for _, entries in res or []:
            seen_dollar.extend(int(e[1]["n"]) for e in entries)
        time.sleep(0.08)          # simulate work between reads

def reader_last_id():
    c = redis.Redis(decode_responses=True)
    last = "0-0"; end = time.time() + 1.2
    while time.time() < end:
        res = c.xread({"ticks": last}, block=200, count=10)
        for _, entries in res or []:
            seen_id.extend(int(e[1]["n"]) for e in entries); last = entries[-1][0]
        time.sleep(0.08)
```

```text
reader using $ each time  saw: [0, 2, 4, 6, 8]
reader passing last id    saw: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

The `$` reader **missed half the entries**, all the ones that arrived while it was busy. `$` is resolved when the
command runs, so anything added between two `XREAD` calls is skipped. Use `$` once to start "from now", and then
always pass the last ID you processed. Because that ID is just a value, a reader can save it (in Redis, or alongside
its own output) and resume exactly where it left off after a restart. That's what makes a stream a log rather than a
broadcast channel.

## Keeping the stream bounded

A stream grows forever unless you trim it. `MAXLEN` keeps the newest N entries, and `MINID` drops entries older
than an ID (a time). Both are available on `XTRIM` and inline on `XADD`:

```python
t = time.perf_counter(); n = r.xtrim("events", maxlen=500_000, approximate=False)
print(f"XTRIM MAXLEN = 500000 removed {n} in {(time.perf_counter()-t)*1000:.1f} ms, XLEN={r.xlen('events')}")
t = time.perf_counter(); n = r.xtrim("events", maxlen=100_000, approximate=True)
print(f"XTRIM MAXLEN ~ 100000 removed {n} in {(time.perf_counter()-t)*1000:.1f} ms, XLEN={r.xlen('events')}")
for _ in range(3):
    r.xadd("events", {"type": "click"}, maxlen=100_000, approximate=True)
print("after 3 more XADD MAXLEN ~ 100000: XLEN", r.xlen("events"))
```

```text
XTRIM MAXLEN = 500000 removed 500000 in 5.4 ms, XLEN=500000
XTRIM MAXLEN ~ 100000 removed 10000 in 0.3 ms, XLEN=490000
after 3 more XADD MAXLEN ~ 100000: XLEN 460003
```

Two behaviours are worth knowing:

- **Exact trimming (`=`)** removed half a million entries in one command, 5.4 ms with the server blocked.
- **Approximate trimming (`~`)** only removes whole internal nodes, and **at most 10,000 entries per call** by
  default (`LIMIT` defaults to 100 × `stream-node-max-entries`). It took 0.3 ms. A stream that's far over its limit
  shrinks gradually, 10,000 entries per `XADD`, instead of in one long pause.

In production, put `MAXLEN ~ N` (or `MINID ~ <cutoff>`) on every `XADD`. The cost is spread across writes and the
stream stays near its target. Use `=` only when you need an exact length and the trim is small.

## Idempotent appends (Redis 8.6+)

A producer that times out doesn't know whether its `XADD` happened (Level 1 · 09). If it retries, the log may get
the event twice. Redis 8.6 added **idempotent production** to `XADD`. The producer names itself and gives each
message an idempotency ID:

```text
127.0.0.1:6379> XADD payments IDMP checkout-svc req-001 * amount 1999
"1791856600024-0"
127.0.0.1:6379> XADD payments IDMP checkout-svc req-001 * amount 1999
"1791856600024-0"
127.0.0.1:6379> XADD payments IDMP checkout-svc req-002 * amount 500
"1791856600143-0"
127.0.0.1:6379> XLEN payments
(integer) 2
```

The retry with the same producer and ID `req-001` returned the **original entry's ID** and added nothing.
`IDMPAUTO` derives the idempotency ID from the message content instead:

```text
127.0.0.1:6379> XADD payments IDMPAUTO checkout-svc * amount 700
"1791856600261-0"
127.0.0.1:6379> XADD payments IDMPAUTO checkout-svc * amount 700
"1791856600261-0"
```

`XINFO STREAM` reports the bookkeeping (`pids-tracked 1`, `iids-tracked 3`, `iids-duplicates 2`), plus the window
settings `idmp-duration 100` and `idmp-maxsize 100`. Two behaviours, observed in experiments on 8.10.2:

- Deduplication lasts for **`IDMP-DURATION` seconds** (1–86400, set with `XCFGSET`). With the duration set to 2,
  a repeat 1 second later was deduplicated, and a repeat about 3.5 seconds later created a new entry.
- Running `XCFGSET` on a stream **reset the tracked IDs**: a repeat sent right after changing the settings was
  accepted as new. Configure streams before you depend on deduplication.

This is a very new feature (`HELP XADD` lists `[IDMPAUTO pid|IDMP pid iid]`). Older servers and other forks reject
these arguments, and client libraries may need raw commands for them. On older servers, the portable alternative
is a Lua script that checks a `SET … NX` marker before appending (Level 2 · 09).

## Memory: a stream vs a list of JSON

A million events with the same three fields, stored as a stream and as a list of JSON strings:

```text
1M entries -> stream 26557515 bytes; list of JSON 52764823 bytes
radix-tree-keys 10000 radix-tree-nodes 9550
```

The stream used **half** the memory of the list, even though it also stores an ID for every entry.

## How It Actually Works

A stream is a **radix tree** (rax) keyed by entry ID. Each leaf holds a **listpack** containing a run of
consecutive entries, at most `stream-node-max-entries` (100) entries or `stream-node-max-bytes` (4 KB) per node.
That's 10,000 radix keys for a million entries above. The first entry in each listpack is a **master entry** that
records the field names. Later entries with the same fields store a flag saying "same fields as the master" plus
their values, and their IDs are stored as deltas from the master's ID. That's why a stream of uniformly shaped
events is so compact: the field names `type`, `user`, `page` are stored once per 100 entries instead of a million
times.

`XADD *` takes the current time in ms. If it's not greater than the last ID's time, it reuses the last time and
increments the sequence. Appending goes to the last listpack, or starts a new node when that one is full. Range
reads seek in the radix tree in O(log N) and then walk listpacks. `XDEL` only marks entries deleted inside their
listpack, and the space is reclaimed when the whole node is trimmed. That's one reason deletes are discouraged.
Approximate trimming removes whole nodes from the front, which is why it's cheap and why it can't hit an exact
count.

A blocked `XREAD` is registered like a blocked `BLPOP` (Level 1 · 06), keyed by stream. When `XADD` appends, the
server wakes the blocked readers and serves each the entries after the ID it asked for. Every reader gets the
entries (they're not consumed). For `$`, the "ID it asked for" was fixed at the moment the command started.

The idempotency feature keeps, per stream, a bounded table of producer IDs and their recent idempotency IDs with
the stream ID each one produced. A repeat within the window returns the stored ID without appending. The table is
bounded by `IDMP-MAXSIZE` and `IDMP-DURATION`, which is why deduplication only covers recent retries.

## Common mistakes

- **Passing `$` on every `XREAD`.** It drops everything that arrives between calls. Track and pass the last ID.
- **No trimming.** A stream without `MAXLEN`/`MINID` grows until memory runs out.
- **Assuming `MAXLEN ~` trims everything at once.** It's bounded per call (10,000 entries by default).
- **Using `XDEL` as a work queue's "done" marker.** Use consumer groups and `XACK` (next lesson).
- **Explicit IDs from client clocks.** One bad clock can push IDs into the future for everyone.
- **Huge entries.** An entry is meant to be an event, not a document. Store large payloads elsewhere and put a
  reference in the stream.
- **Relying on `IDMP` without checking the server version**, or without knowing that it's time-bounded.

## Exercise

1. Write a reader that persists its last processed ID to a key after each batch, kill it mid-stream, restart it, and
   prove that it neither skips nor repeats entries (or explain precisely when it can repeat).
2. Add 2 million entries with `XADD … MAXLEN ~ 100000` on every write, starting from an empty stream. Record `XLEN`
   every 100,000 writes. How far above 100,000 does it ever get?
3. Query "all orders between 10:00:00 and 10:00:05 today" using millisecond IDs as range bounds.
4. On Redis 8.6+, write a producer that retries `XADD IDMP` on timeout, and simulate timeouts by blocking the server
   with `DEBUG SLEEP` (lab only). Confirm no duplicates within the window, then show a duplicate slipping through
   when the retry comes after `IDMP-DURATION` has passed.
