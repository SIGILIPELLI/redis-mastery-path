---
description: "The Redis AOF measured: fsync policies, what kill -9 really loses versus a power cut, multi-part AOF, rewrites, and truncated vs corrupt files."
---

# 02 · Persistence II: AOF, fsync & Hybrid Persistence

RDB loses everything since the last snapshot. The **append-only file (AOF)** narrows that window by logging every
write command as it happens. On restart, Redis replays the log. How narrow the window gets depends on one setting,
`appendfsync`, and on understanding which failure you're protecting against. This lesson measures the policies, runs
crash tests, and breaks the file on purpose.

## Turning it on and looking inside

```bash
redis-server --port 6451 --dir ./rw --save '' --appendonly yes
```

Since Redis 7, the AOF is a **directory** (`appendonlydir/`) with several parts and a manifest. After three `SET`s and
100,000 `INCR`s:

```text
89 appendonly.aof.1.base.rdb
3600137 appendonly.aof.1.incr.aof
102 appendonly.aof.manifest
```

The incremental file is plain RESP, exactly what clients send (`^M` is `\r`):

```text
*2^M
$6^M
SELECT^M
$1^M
0^M
*3^M
$3^M
SET^M
$8^M
greeting^M
$5^M
hello^M
...
```

It's 3.6 MB to describe a dataset of two keys, because the log records every one of the 100,000 increments.

## Rewriting: compacting the log

`BGREWRITEAOF` (or the automatic trigger) forks, writes the *current* dataset as a new base file, and starts a fresh
incremental file:

```text
Background append only file rewriting started

124 appendonly.aof.2.base.rdb
0 appendonly.aof.2.incr.aof
107 appendonly.aof.manifest
```

```text
file appendonly.aof.2.base.rdb seq 2 type b
file appendonly.aof.2.incr.aof seq 2 type i startoffset 100003
```

3.6 MB became 124 bytes. The base is in RDB format because `aof-use-rdb-preamble` is `yes` by default ("hybrid"
persistence: compact snapshot plus a log of what came after). The manifest says which files make up the current AOF.
Rewrites happen automatically when the AOF has grown by `auto-aof-rewrite-percentage` (100%) since the last rewrite
and is at least `auto-aof-rewrite-min-size` (64 MB).

## The fsync policies

Writing to a file puts the data in the operating system's page cache. Only `fsync` forces it onto the storage device.
`appendfsync` decides when Redis calls it:

- **`always`** — after every write batch, before replying. Nothing acknowledged is lost even if the machine loses
  power.
- **`everysec`** (default) — once per second in a background thread. A power loss or kernel crash can lose up to
  about a second of writes.
- **`no`** — never explicitly. The OS flushes when it chooses (often every 30 s on Linux).

Throughput of `SET` with `redis-benchmark`, on this MacBook's internal SSD:

```text
aof=off       SET 50 clients: 160513.64 req/s   1 client: 12763.24 req/s
aof=no        SET 50 clients: 143266.47 req/s   1 client: 11634.67 req/s
aof=everysec  SET 50 clients: 146198.83 req/s   1 client: 11876.48 req/s
aof=always    SET 50 clients:   5719.84 req/s   1 client:    29.67 req/s
```

- `everysec` cost about 9% of throughput against no AOF, which is why it's the default.
- **`always` collapsed throughput**: 5,720 writes/s with 50 clients (Redis batches all writes from one event-loop
  iteration into one fsync) and **under 30 writes per second** for a single client waiting for each reply.

Take that last number as a property of this machine, not of Redis. On macOS, Redis flushes with `F_FULLFSYNC`, which
forces the drive to write its own cache to the medium and takes tens of milliseconds. Linux servers with
datacenter NVMe drives (with power-loss protection) complete `fsync` much faster. Measure `always` on your own
production hardware before deciding.

## What does a crash actually lose?

This test increments a counter as fast as one client can, counts the increments the server **acknowledged**, kills
the server with `SIGKILL` after 3 seconds, restarts it, and compares:

```python
r = redis.Redis(port=6450, retry=redis.retry.Retry(redis.backoff.NoBackoff(), 0))
pid = r.info("server")["process_id"]
acked = 0
def killer():
    time.sleep(3); os.kill(pid, signal.SIGKILL)
threading.Thread(target=killer).start()
try:
    while True:
        acked = r.incr("acked")             # only count what the server acknowledged
except redis.ConnectionError:
    pass
# restart the server on the same directory, then:
after = int(redis.Redis(port=6450).get("acked") or 0)
```

```text
appendfsync everysec: acknowledged  24481, recovered  24481, lost 0
appendfsync always  : acknowledged    630, recovered    630, lost 0
```

**Neither lost anything**, and that's not a contradiction of the "up to one second" warning. `kill -9` kills the
*process*, not the machine. Everything Redis had `write()`n was already in the kernel's page cache, and the kernel
finished writing it to disk after Redis died. `everysec` only loses data when the **operating system or the power**
fails before the next fsync: a kernel panic, a hypervisor crash, a pulled plug. That couldn't be simulated safely on
this laptop, so it isn't measured here. The distinction matters:

| Failure | `no` | `everysec` | `always` |
|---|---|---|---|
| Redis process crash / OOM kill / `kill -9` | nothing acknowledged lost | nothing lost | nothing lost |
| OS crash or power loss | up to ~30 s (OS-dependent) | up to ~1 s | nothing acknowledged |

(One subtlety: with `everysec`, if the disk can't keep up, Redis may delay the next write until the previous fsync
completes, so the loss stays bounded at about 2 seconds rather than growing without limit.)

## Truncated vs corrupt files

A crash during a write can leave a **half-written command** at the end of the file. Here one is simulated by
appending a fragment, `*3\r\n$3\r\nSET\r\n$1\r\nc\r\n$1`, and restarting:

```text
* DB loaded from base file appendonly.aof.2.base.rdb: 0.001 seconds
# !!! Truncating the AOF appendonly.aof.2.incr.aof at offset 77 !!!
# AOF appendonlydir/appendonly.aof.2.incr.aof loaded anyway because aof-load-truncated is enabled
```

The server dropped the incomplete tail and started. `a` and `b` were intact, and the half-written `c` simply didn't
exist. That's the right behaviour, because an incomplete command was never acknowledged.

Corruption **in the middle** is different. With one command name changed to `SXT`:

```text
* DB loaded from base file appendonly.aof.2.base.rdb: 0.001 seconds
# Unknown command 'SXT' reading the append only file appendonly.aof.2.incr.aof
```

The server **refused to start**. Silently skipping a command in the middle of history could produce a state that
never existed. Note what `redis-check-aof` said about that same file:

```text
AOF analyzed: filename=…/appendonly.aof.2.incr.aof, size=77, ok_up_to=77, ok_up_to_line=20, diff=0
AOF …/appendonly.aof.2.incr.aof is valid
```

"Valid", because the tool checks the RESP **framing**, which was intact. It doesn't know whether `SXT` is a real
command. `redis-check-aof --fix` repairs framing damage by truncating at the first bad position, which can throw
away everything after it. Repair a **copy**, compare key counts before and after, and prefer restoring from a
known-good backup when the damage is in the middle.

## How It Actually Works

Every write command, after it executes, is appended to an in-memory **AOF buffer** in RESP form, with
`SELECT` emitted when the database changes. Commands are logged with deterministic effects: an `EXPIRE` with a
relative time is written as `PEXPIREAT` with an absolute one, so replaying it later gives the same result.
Before the event loop goes back to waiting, Redis `write()`s the buffer to the incremental file. Then, depending on
`appendfsync`, it either fsyncs right away (`always`, before replies are sent, which is why throughput collapses),
asks a background thread to fsync if a second has passed (`everysec`), or does nothing (`no`).

A rewrite forks a child, which writes the dataset as the new base (RDB format with the preamble). Meanwhile the
parent opens a **new** incremental file and records new writes there. When the child finishes, the parent atomically
updates the manifest to point at the new base and new incremental file, then deletes the old ones. Since the
multi-part design (Redis 7.0), there's no need to buffer writes in memory during a rewrite and merge them afterwards,
which removed a source of memory spikes and latency in older versions.

On startup, Redis loads the base (fast, RDB) and replays the incremental files in manifest order. A transaction is
logged wrapped in `MULTI`/`EXEC`, and an incomplete transaction at the end of the file is treated like any truncated
tail.

## Common mistakes

- **Choosing `always` without measuring** on production hardware. It can turn a 100,000-writes/s server into a
  hundreds-per-second one.
- **Testing durability with `kill -9`** and concluding `everysec` loses nothing. Only OS/power failures expose the
  window.
- **Running `redis-check-aof --fix` on the only copy.** It truncates.
- **Trusting "AOF is valid"** to mean the contents are sound. It checks framing only.
- **AOF and RDB in different directories you back up separately.** Back up the whole `dir`, including
  `appendonlydir/` and its manifest.
- **Disks too slow for the write rate.** `everysec` then stalls writers. Watch `aof_delayed_fsync` in `INFO
  persistence`.

## Exercise

1. Repeat the throughput benchmark with pipelining (`redis-benchmark -P 16`) for each policy. How much of `always`'s
   cost does batching recover?
2. On a Linux VM you can crash (`echo c > /proc/sysrq-trigger`, only on a disposable VM), run the acknowledged-counter
   test with `everysec` and `always`, and measure the actual loss after the VM reboots.
3. Generate a 500 MB AOF with `auto-aof-rewrite-percentage 0` (disabled), restart and time loading. Then rewrite and
   restart again. Compare load times.
4. Corrupt a byte in the middle of a copy of an incremental file so that the RESP framing breaks (change a `$5` to
   `$9`). Run `redis-check-aof` and `--fix` on the copy, and count how many commands were lost.
