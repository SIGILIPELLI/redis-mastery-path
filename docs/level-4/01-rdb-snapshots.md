---
description: "RDB snapshots in practice: BGSAVE and fork, copy-on-write headroom, what kill -9 loses, load times, redis-check-rdb and save points."
---

# 01 · Persistence I: RDB Snapshots & fork()

Redis keeps data in memory, so persistence is about one question: after a crash or restart, **what comes back?**
Redis has two mechanisms. **RDB** writes a compact point-in-time snapshot of the whole dataset. **AOF** (next lesson)
logs every write. This lesson measures RDB on a scratch server (port 6440) holding 3 million keys, about 458 MB.

## SAVE vs BGSAVE

`SAVE` writes the snapshot in the server's main thread:

```text
SAVE (foreground) took 1.78 s; the server answered nobody meanwhile
```

For 1.78 seconds every client was blocked. Never run `SAVE` on a live server. `BGSAVE` forks a child process to do
the writing, and the parent keeps serving:

```text
BGSAVE -> True
BGSAVE returned in 37.6 ms
rdb_last_bgsave_status: ok
rdb_last_bgsave_time_sec: 3
latest_fork_usec: 23099
```

The command returned in 38 ms, and the snapshot took 3 seconds in the background while a writer thread kept
modifying keys (398,000 writes during the save). The one cost the parent *does* pay is the `fork()` itself:
**23 ms** (`latest_fork_usec`), during which the server is frozen. Fork time grows with the size of the process's
page tables, roughly with memory used, and on large instances (tens of GB) it can reach hundreds of milliseconds.
Watch `latest_fork_usec` on big servers.

The log shows both processes, the parent `M` and the child `C`:

```text
5489:M 13 Oct 2026 15:58:06.888 * Background saving started by pid 5495
5495:C 13 Oct 2026 15:58:09.204 * DB saved on disk
5495:C 13 Oct 2026 15:58:09.204 * Fork CoW for RDB: current 0 MB, peak 0 MB, average 0 MB
5489:M 13 Oct 2026 15:58:09.249 * Background saving terminated with success
```

## Copy-on-write: the memory you need to reserve

After `fork()`, parent and child share every memory page. When the parent modifies a page (any write to a key on
it), the OS copies that page first, so the child keeps seeing the snapshot moment. Under heavy writes during a
save, the copies add up. In the worst case, every page is written and memory use **doubles**.

Here the log reports CoW as 0 MB, and `rdb_last_cow_size` was 0, but that is a **measurement limitation of this
platform**, not a fact about the save. Redis computes CoW from Linux's `/proc/<pid>/smaps`, which macOS doesn't
have. Sampling the processes with `ps` doesn't help either:

```text
parent RSS before: 462 MB
child pid ['5518'] peak child RSS: 457 MB | peak parent RSS: 462 MB
```

RSS counts shared pages in **both** processes, so these numbers say nothing about how much was actually copied. On
Linux, the `Fork CoW` log line and `rdb_last_cow_size` give a real figure. Use them to size headroom: `maxmemory`
plus expected CoW plus buffers must fit in RAM. A common rule of thumb is to keep `maxmemory` well below the machine's
memory on write-heavy instances that snapshot. Also, disable **transparent huge pages** on Linux, which Redis warns
about at startup: with 2 MB pages, writing one byte copies 2 MB instead of 4 KB.

## What a crash loses

Count 5,000 orders after a snapshot, then kill the server with `SIGKILL` (no chance to save):

```python
r.set("orders:count", 0)
r.bgsave(); time.sleep(4)
for i in range(5000): r.incr("orders:count")
print("orders:count before crash:", r.get("orders:count"),
      "| changes since last save:", r.info("persistence")["rdb_changes_since_last_save"])
```

```text
orders:count before crash: b'5000' | changes since last save: 5000
```

```bash
kill -9 <pid>
redis-server --port 6440 --dir … --save ''
redis-cli -p 6440 GET orders:count
```

```text
* Loading RDB produced by version 8.10.2
* Done loading RDB, keys loaded: 3000001, keys expired: 0.
* DB loaded from disk: 2.299 seconds
0
```

**All 5,000 increments were lost**. The counter came back as 0, its value at the last snapshot. With RDB alone, you
lose everything since the last successful snapshot. That's the right trade-off for caches and for data you can
rebuild, and the wrong one for orders.

Two more facts from that restart:

- **Loading took 2.3 seconds**, and the server doesn't accept normal commands until it's done (clients get
  `LOADING`). Restart time grows with dataset size, so plan for it in failover and deploy procedures.
- `rdb_changes_since_last_save` is the number of writes at risk right now. It's a useful metric to alert on.

## Save points

When to snapshot automatically is set with `save <seconds> <changes>` rules, any of which triggers a `BGSAVE`. The
built-in default (Level 1 · 01) is `3600 1 300 100 60 10000`. For a cache you can rebuild, `save ""` (no snapshots)
is common. For data you care about, combine RDB with AOF rather than tightening save points to every few seconds,
because each snapshot rewrites the *whole* dataset.

`stop-writes-on-bgsave-error yes` (the default) makes Redis refuse writes when the last background save failed, for
example because the disk is full or the directory isn't writable. It's a loud failure instead of a silent loss of
durability. (An attempt to trigger it here by making the directory read-only didn't produce a failed save on this
macOS setup, so it isn't demonstrated.) If you turn it off, monitor `rdb_last_bgsave_status` yourself.

## Checking a snapshot file

`redis-check-rdb` validates a file offline:

```text
[offset 101043498] Checksum OK
[offset 101043498] \o/ RDB looks OK! \o/
[info] 3000001 keys read
```

A file truncated at 50 MB (as a copy interrupted mid-transfer might be):

```text
[offset 50000000] Unexpected EOF reading RDB file
[additional info] While doing: read-object-value
[additional info] Reading key 'k:305343'
```

Run it on backups before you trust them. A backup that has never been restored is a hope, not a backup.

## How It Actually Works

`BGSAVE` calls `fork()`. The kernel duplicates the process's page tables (not its memory) and marks all pages
read-only and shared. That table copy is the 23 ms freeze. The child walks every database and key and serializes
them into a temporary file (`temp-<pid>.rdb`), then `fsync`s it and `rename`s it over `dump.rdb`. Rename is atomic, so
`dump.rdb` is always either the old complete snapshot or the new complete one, never half-written. The parent sees the
child exit through `waitpid` in the event loop and records the result.

The RDB format is compact: a header with the version, then for each key an optional expiry, a type byte, the key and
the value in a type-specific encoding (small hashes and sets are stored as their listpack or intset blobs directly,
which also makes loading fast), and an 8-byte CRC64 checksum at the end. Strings may be LZF-compressed, which is why
458 MB in memory became a 105 MB file. Loading reverses it and re-encodes each value compactly (Level 2 · 01's
exercise 3 relies on this).

Because the snapshot is the state at the moment of `fork()`, writes that arrive during a long save are in memory but
not in the file. They count towards `rdb_changes_since_last_save` for the *next* save.

## Common mistakes

- **Running `SAVE`.** It blocks everything for seconds.
- **No memory headroom for copy-on-write.** A save under heavy writes can push a full server into swap or the OOM
  killer.
- **Transparent huge pages enabled** on Linux. CoW copies get 512× bigger.
- **Assuming RDB is enough for important data.** Everything since the last snapshot is lost in a crash.
- **Ignoring `rdb_last_bgsave_status`** with `stop-writes-on-bgsave-error no`.
- **Untested backups.** Run `redis-check-rdb` and do a real restore.
- **Forgetting restart time.** 2.3 s for 3 M keys here. Minutes for very large datasets.

## Exercise

1. Measure `latest_fork_usec` for datasets of 0.5, 1, 2 and 4 GB (`DEBUG POPULATE`). Is the relationship roughly
   linear?
2. On Linux, run `BGSAVE` with and without a heavy writer and compare `rdb_last_cow_size`. Then estimate the RAM
   headroom you'd need for your largest instance.
3. Corrupt one byte in the middle of a copy of `dump.rdb` and see what `redis-check-rdb` reports. Then try starting a
   server on it. What does the server do?
4. Configure `save 60 1000` and write a script that tells you, at any moment, how many seconds of writes you'd lose
   in a crash, using `rdb_last_save_time` and `rdb_changes_since_last_save`.
