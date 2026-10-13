---
description: "What Redis really is — an in-memory data structure server with one command thread — plus a throwaway local lab you can rebuild in seconds."
---

# 01 · What Redis Is: One Thread, Memory & a Local Lab

Redis is usually introduced as "a cache" or "a key-value store". Both undersell it and both mislead. Redis is
a **data structure server**: a process that keeps strings, hashes, lists, sets, sorted sets, streams and a few
more exotic types in RAM, and lets many clients run small, atomic operations on them over a network socket.
A cache is one thing you can build with that. So are leaderboards, rate limiters, job queues, session stores,
presence indicators, distributed locks and event logs.

Two facts shape everything else in this course:

1. **The data lives in memory.** Reads and writes don't touch a disk on the request path. That's why Redis is
   fast, and it's also why the dataset must fit in RAM and why durability is something you configure rather
   than something you get.
2. **Commands execute one at a time, on one thread.** Each command runs to completion before the next one
   starts. That's why `INCR` never loses an update. It's also why one slow command makes *every* client wait.

This lesson sets up a lab, then measures both facts on a real server.

## Installing Redis

Everything in this course was run on **Redis 8.10.2**, installed with Homebrew on macOS (Apple Silicon):

```bash
brew install redis
redis-server --version
```

```text
Redis server v=8.10.2 sha=00000000:1 malloc=libc bits=64 build=764b3e45065a66fb
```

The formula installs six programs:

```text
redis-benchmark  redis-check-aof  redis-check-rdb  redis-cli  redis-sentinel  redis-server
```

On Linux, the Redis project publishes its own APT/RPM repositories and an official `redis` container image;
your distribution's package may be several major versions behind, so check `redis-server --version` before
assuming a feature exists. (Those install paths weren't used for this course. Commands are the same once the
server is running.)

!!! note "Valkey"
    If your platform ships **Valkey** instead of Redis, most of Levels 1–3 still applies. Valkey is a fork of
    Redis 7.2 and speaks the same protocol and core commands. The two projects have diverged since then.
    Level 4 · 09 compares them directly on the same machine.

## A throwaway lab

You want a server that you can wipe and restart freely, separate from any Redis that something else on the
machine depends on. Command-line flags override config file settings, so you don't need a config file to
start one:

```bash
mkdir -p ~/redis-lab && cd ~/redis-lab
redis-server --port 6400 --dir ~/redis-lab --save '' --enable-debug-command local
```

- `--port 6400` keeps it off the default port 6379.
- `--dir` is where any snapshot or append-only file would be written.
- `--save ''` turns off automatic snapshots, so nothing lands on disk unless you ask.
- `--enable-debug-command local` allows `DEBUG` from localhost. It's off by default since Redis 7, and we
  use it here to generate test data. Never enable it on a shared server.

The server logs to the terminal:

```text
97838:C 13 Oct 2026 06:54:46.616 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
97838:C 13 Oct 2026 06:54:46.616 * Redis version=8.10.2, bits=64, commit=00000000, modified=1, pid=97838, just started
97838:C 13 Oct 2026 06:54:46.616 * Configuration loaded
97838:M 13 Oct 2026 06:54:46.616 * monotonic clock: ARM CNTVCT @ 24 ticks/us
97838:M 13 Oct 2026 06:54:46.616 * Running mode=standalone, port=6400.
97838:M 13 Oct 2026 06:54:46.616 # WARNING: The TCP backlog setting of 511 cannot be enforced because kern.ipc.somaxconn is set to the lower value of 128.
97838:M 13 Oct 2026 06:54:46.617 * Server initialized
97838:M 13 Oct 2026 06:54:46.617 * Ready to accept connections tcp
97838:M 13 Oct 2026 06:54:46.617 # WARNING: Redis does not require authentication. Redis will accept connections from any local client.
```

Each line starts with the PID and a role letter: `C` is a child or startup phase, `M` is the main process.
The two warnings are worth reading rather than ignoring. The backlog warning means the OS caps how many
not-yet-accepted connections can queue up, which matters on busy Linux servers (raise `net.core.somaxconn`).
The authentication warning says plainly that anyone who can reach this port can run any command. Level 4 ·
07 deals with that.

In a second terminal:

```bash
redis-cli -p 6400 ping
```

```text
PONG
```

The **lab convention** for the rest of the course: a lab server on the default port **6379**, started the
same way with its own `--dir`. Lessons that need several servers (replication, Sentinel, Cluster) start them
on other ports and say so. Shut a lab server down with `redis-cli -p <port> shutdown nosave`. The `nosave`
means "don't write a snapshot on the way out".

### Loading the bundled modules

Redis 8 ships JSON, the query engine (search and vector indexes), probabilistic types (Bloom, Cuckoo, Count-Min,
Top-K, t-digest) and time series as modules that are part of the Redis Open Source distribution. Homebrew's
`/opt/homebrew/etc/redis.conf` loads them with `loadmodule` lines, but a server started only with flags, like
the one above, doesn't:

```bash
redis-cli -p 6400 MODULE LIST
```

```text
1) 1) "name"
   2) "vectorset"
   3) "ver"
   4) (integer) 1
   ...
```

Only the built-in `vectorset` type is there. For the lessons that use JSON, search or Bloom filters (Level 2 ·
03, Level 3 · 08–10), start the lab from a small config file instead:

```text
# ~/redis-lab/redis.conf
port 6379
dir /Users/you/redis-lab
save ""
appendonly no
loadmodule /opt/homebrew/opt/redis/lib/redis/modules/redisbloom.so
loadmodule /opt/homebrew/opt/redis/lib/redis/modules/redisearch.so
loadmodule /opt/homebrew/opt/redis/lib/redis/modules/rejson.so
loadmodule /opt/homebrew/opt/redis/lib/redis/modules/redistimeseries.so
```

```bash
redis-server ~/redis-lab/redis.conf
```

On Linux the module paths depend on how Redis was packaged. Look for the `loadmodule` lines in the
`redis.conf` your package installed.

## Fact 1: the data lives in memory

Ask the empty server how much memory it's using:

```bash
redis-cli -p 6400 INFO memory | grep -E '^used_memory(_human|_rss_human)?:'
```

```text
used_memory:930608
used_memory_human:908.80K
used_memory_rss_human:4.72M
```

`used_memory` is what Redis's allocator has handed out. `used_memory_rss` is what the operating system says the
process occupies, including allocator overhead and fragmentation. Now generate a million keys, each a
64-byte string. `DEBUG POPULATE` exists for exactly this kind of test:

```bash
redis-cli -p 6400 DEBUG POPULATE 1000000 user 64
redis-cli -p 6400 DBSIZE
redis-cli -p 6400 INFO memory | grep -E '^used_memory_human|^used_memory_rss_human'
```

```text
OK
1000000
used_memory_human:121.20M
used_memory_rss_human:126.12M
```

About 120 MB for 64 MB of values. That's roughly **120 bytes per key** for keys like `user:42` holding 64
bytes. The difference is overhead: the hash table entry, the key object, the value object and the allocator's
size classes. Ask about one key:

```bash
redis-cli -p 6400 GET user:42
redis-cli -p 6400 MEMORY USAGE user:42
```

```text
value:42
106
```

`DEBUG POPULATE` pads the value to 64 bytes, which is why `value:42` has trailing spaces. `MEMORY USAGE`
estimates 106 bytes for this key, its value and its share of the table. When you size a Redis server, count the
overhead per key and not just the payload. With many small keys, the overhead can be bigger than the data.
Level 2 · 01 shows how to shrink it.

Memory is volatile, so what happens to those million keys if the process stops? It depends entirely on
configuration. Write a snapshot by hand, kill the server without saving, and start it again:

```bash
redis-cli -p 6400 SAVE
ls -la dump.rdb
redis-cli -p 6400 SHUTDOWN NOSAVE
redis-server --port 6400 --dir ~/redis-lab --save ''
```

```text
OK
-rw-r--r--@ 1 bhanuja  wheel  36777608 Oct 13 06:55 dump.rdb
```

```text
97881:M 13 Oct 2026 06:55:06.833 * Loading RDB produced by version 8.10.2
97881:M 13 Oct 2026 06:55:07.524 * Done loading RDB, keys loaded: 1000000, keys expired: 0.
97881:M 13 Oct 2026 06:55:07.524 * DB loaded from disk: 0.692 seconds
```

The 121 MB in memory became a 37 MB file. RDB is a compact serialization and compresses repetitive strings.
On startup Redis found `dump.rdb` in its `dir` and loaded it before accepting connections. Without that `SAVE`,
the restart would have come up empty, because this lab disabled automatic snapshots.

A server started with **no** configuration at all is *not* in that state:

```bash
redis-server --port 6401 &
redis-cli -p 6401 CONFIG GET save
redis-cli -p 6401 CONFIG GET appendonly
```

```text
save
3600 1 300 100 60 10000
appendonly
no
```

The built-in default snapshots after 3600 s if at least 1 key changed, after 300 s if 100 changed, and after
60 s if 10,000 changed. So by default you can lose up to an hour of writes in a crash, and the append-only file
that narrows that window is off. Whether that's acceptable depends on what you store. Level 4 · 01–02 cover
persistence properly.

## Fact 2: one command at a time

Look at the threads of the running server:

```bash
ps -M -p $(pgrep -f 'redis-server.*6400')
```

```text
USER      PID   TT   %CPU STAT PRI     STIME     UTIME COMMAND
bhanuja 97838   ??    0.2 S    31T   0:00.01   0:00.01 redis-server *:6400
        97838         0.0 S    31T   0:00.00   0:00.00
        97838         0.0 S    31T   0:00.00   0:00.00
        97838         0.0 S    31T   0:00.00   0:00.00
```

There are four threads. The main thread runs the event loop and **executes every command**. The other three are
background ("bio") threads that close files, fsync the append-only file and free large values lazily, so that
none of those slow system calls stall the main thread. None of them runs client commands.

The consequence is easy to measure. This script pings the server every millisecond from one connection while
another connection runs `KEYS *`, a command that walks the entire keyspace:

```python
import threading, time, redis

r = redis.Redis(port=6400)

def slow():
    t = time.perf_counter()
    n = len(r.keys('*'))
    print(f"KEYS * returned {n} keys in {time.perf_counter()-t:.3f}s")

worst = 0.0
def pinger(stop):
    global worst
    c = redis.Redis(port=6400); c.ping()
    while not stop.is_set():
        t = time.perf_counter(); c.ping()
        worst = max(worst, time.perf_counter() - t)
        time.sleep(0.001)

stop = threading.Event()
th = threading.Thread(target=pinger, args=(stop,)); th.start()
time.sleep(0.3); print(f"worst PING before: {worst*1000:.2f} ms"); worst = 0.0
slow()
time.sleep(0.2); stop.set(); th.join()
print(f"worst PING while KEYS ran: {worst*1000:.1f} ms")
```

```text
worst PING before: 0.34 ms
KEYS * returned 1000000 keys in 0.192s
worst PING while KEYS ran: 149.1 ms
```

A `PING` normally takes a third of a millisecond. While `KEYS *` ran, one `PING` waited 149 ms. Nothing was
wrong with that client; it was just queued behind someone else's command. On a laptop with a million keys that's
a hiccup. With 50 million keys in production, it's an outage. That's why Level 1 · 04 replaces `KEYS` with
`SCAN`, and why "what's the time complexity of this command?" is a question you'll keep asking all course.

The same property is a gift. Because nothing else runs while `INCR counter` executes, the read-add-write
inside it can't interleave with another client's. You get atomicity for every single command without writing
a lock.

## When Redis is the wrong tool

- **The data doesn't fit in RAM**, or RAM costs too much at your size. Redis is priced per gigabyte of memory.
- **You need ad-hoc queries.** There's no general query language over values. You design keys and data
  structures around the questions you'll ask. The query engine (Level 3 · 09) adds secondary indexes, but you
  still declare them up front.
- **Losing the last second of writes is unacceptable** and you can't prove your persistence and replication
  setup meets that. A relational database with synchronous commit is the safer default for records of truth.
- **Large values.** A single 50 MB string has to be serialized on the one thread that serves everyone.

Most production systems use Redis *alongside* a primary database: the database holds the truth, and Redis
holds fast, derived or short-lived state.

## How It Actually Works

When `redis-server` starts, it parses flags and the config file, creates a listening socket and enters an
**event loop** (Redis's small `ae` library). On macOS the loop waits on `kqueue` (the `multiplexing_api:kqueue`
line in `INFO server`); on Linux it uses `epoll`. Each iteration asks the kernel which sockets are readable,
reads whatever bytes have arrived into each client's query buffer, parses complete commands out of it, and runs
them one after another. Replies go into per-client output buffers and are written back when the sockets become
writable. Timers also run in that loop. The `serverCron` function, at `hz` times per second (10 by default),
expires keys, checks whether a snapshot is due and updates statistics.

Because one thread owns all the data, the data structures need no locks. A command reads and modifies the
in-memory structures directly and finishes before the loop moves on. This is the source of both the atomicity
and the head-of-line blocking you measured. Redis can be configured with `io-threads` (it was `1` on the lab
server) to move reading and writing of sockets onto helper threads. That helps when a large server spends most
of its time in network I/O. Command execution itself stays on the main thread.

The keyspace is a hash table from key to value object. Each key is a compact string type (SDS, "simple dynamic
string") that stores its length up front, so strings are binary-safe and `STRLEN` is O(1). Each value carries a
type tag and an *encoding*: the same logical type may be stored in different ways depending on its size. Level
2 · 01 explores this. The per-key overhead you measured is the table entry plus those headers, rounded up to the
allocator's size classes.

`SAVE` serialized the whole dataset into the RDB format on the main thread, blocking clients while it wrote.
On startup Redis reads the file back before opening for business, which is why loading a million keys took
0.69 s with the server unavailable. That gap is one reason restarting a large Redis isn't free.

## Common mistakes

- **Treating Redis as durable by default.** The default snapshot schedule can lose an hour of writes, and many
  managed or containerized setups disable persistence entirely. Know your configuration.
- **Running O(N) commands on big keyspaces.** `KEYS *`, `FLUSHALL` (synchronous), `SMEMBERS` on a huge set, or
  `HGETALL` on a huge hash all block every other client for the whole time they run.
- **Sizing by payload.** A million 64-byte values took 121 MB, not 64 MB. Measure `used_memory` with a realistic
  sample before you buy RAM.
- **Exposing the port.** With no password, anyone who reaches the port can read, delete or reconfigure
  everything. Protected mode only stops connections from outside localhost when no password *and* no explicit
  bind address are set.
- **Testing against the shared server.** Use a lab you can wipe. Several lessons in this course deliberately
  break things.

## Exercise

1. Start a lab on port 6400 with `--save ''` and `--enable-debug-command local`. Populate 200,000 keys with
   `DEBUG POPULATE 200000 k 16`, then with 200,000 keys of 1,000 bytes (flush in between with `FLUSHALL`).
   Record `used_memory` each time and work out the overhead per key in both cases. Which case has the larger
   overhead *as a percentage*?
2. Run the `KEYS *` blocking script with 100k, 1M and 3M keys. Does the worst `PING` latency grow roughly
   linearly with the key count?
3. Run `INFO server` and find `multiplexing_api`, `process_id` and `config_file`. Why is `config_file` empty for
   your flag-started lab?
4. Without `SAVE`, shut the `--save ''` server down with plain `SHUTDOWN` (no `NOSAVE`) and restart it. Were
   the keys kept? Then repeat with a server started with `--save '60 1'` and compare the shutdown lines in the
   two logs. What does `SHUTDOWN` decide based on the `save` setting?
