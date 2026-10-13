---
description: "Finding what slows Redis down: planted bad commands caught with SLOWLOG, LATENCY, commandstats, latency percentiles and hot/big key scans."
---

# 08 · Monitoring & Troubleshooting

When an application team says "Redis is slow", Redis itself can usually tell you exactly why: which command, from which
client, how long it took, how often it happens, and which key. This lesson plants four classic problems in an otherwise
healthy workload, then finds each of them with Redis's built-in instruments, the way you would during an incident.

## Planting the problems

A scratch server (port 6495, `allkeys-lfu`, `latency-monitor-threshold 50`) gets 300,000 normal keys and one
400,000-field hash. After a burst of ordinary traffic, four mistakes are run:

```python
import random, time, threading, redis
r = redis.Redis(port=6495, decode_responses=True)
r.execute_command("DEBUG", "POPULATE", "300000", "item", "200")
p = r.pipeline(transaction=False)
for i in range(400_000):
    p.hset("big:catalog", f"sku{i}", "x" * 50)
    if i % 20000 == 0: p.execute()
p.execute()
r.config_resetstat(); r.slowlog_reset(); r.execute_command("LATENCY", "RESET")
# normal traffic plus three bad habits
for i in range(20000):
    r.get(f"item:{random.randrange(300000)}")
    if i % 1000 == 0: r.get("hot:config")
for _ in range(500): r.get("hot:config")
r.hgetall("big:catalog")                             # 1. reading a huge hash in one go
r.keys("item:1*")                                    # 2. KEYS on 300k keys
r.eval("local n=0 for i=1,3000000 do n=n+i end return n", 0)   # 3. a CPU-heavy script
r.delete("big:catalog")                              # 4. synchronous delete of a big key
print("done")
```

## SLOWLOG: which commands were slow

The slow log records every command whose **execution** took longer than `slowlog-log-slower-than` microseconds
(default 10,000, so 10 ms), keeping the last `slowlog-max-len` entries:

```text
127.0.0.1:6495> SLOWLOG GET 5
1) 1) (integer) 4
   2) (integer) 1791908690
   3) (integer) 43839
   4) 1) "DEL"
      2) "big:catalog"
   5) "[::1]:57218"
   6) ""
   7) (integer) 2
2) 1) (integer) 3
   2) (integer) 1791908690
   3) (integer) 14929
   4) 1) "EVAL"
      2) "local n=0 for i=1,3000000 do n=n+i end return n"
      3) "0"
   5) "[::1]:57218"
   6) ""
   7) (integer) 3
3) 1) (integer) 2
   2) (integer) 1791908690
   3) (integer) 24196
   4) 1) "KEYS"
      2) "item:1*"
   ...
4) 1) (integer) 1
   2) (integer) 1791908690
   3) (integer) 40213
   4) 1) "HGETALL"
      2) "big:catalog"
   ...
```

All four planted problems, newest first. Each entry has an ID, a Unix timestamp, the **duration in microseconds**
(the `DEL` blocked the server for 43.8 ms, `HGETALL` for 40.2 ms, `KEYS` for 24.2 ms, the script for 14.9 ms), the
command and arguments, the client address and name, and, new in 8.10 per `COMMAND DOCS`, the original argument count.
Name your connections (`CLIENT SETNAME`, or the `client_name` option in libraries) so the sixth field tells you which
service sent the command.

What the slow log **doesn't** include is time spent waiting in the queue, network time or time writing the reply. A
client can see 200 ms while the slow log shows nothing, if the server was busy with *someone else's* slow command. That
is the head-of-line blocking from Level 1 · 01.

## LATENCY: when the server stalled

The latency monitor records **events** (command execution, fork, AOF fsync, expiry cycles, eviction) that exceed
`latency-monitor-threshold` milliseconds, as a time series. At 50 ms nothing was recorded, since the worst command was
43.8 ms. With the threshold at 10 ms and the problems run again:

```text
127.0.0.1:6495> LATENCY LATEST
1) 1) "command"
   2) (integer) 1791908706
   3) (integer) 48
   4) (integer) 48
```

The `command` event peaked at 48 ms. There's one sample per event type per second, holding the worst value, so all the
slow commands that second collapsed into one point. `LATENCY DOCTOR` turns the history into advice:

```text
1. command: 1 latency spikes (average 48ms, mean deviation 0ms, period 1.00 sec). Worst all time event 48ms.

I have a few advices for you:

- Check your Slow Log to understand what are the commands you are running which are too slow to execute. …
- Deleting, expiring or evicting (because of maxmemory policy) large objects is a blocking operation. If you have very
  large objects that are often deleted, expired, or evicted, try to fragment those objects into multiple smaller objects.
```

Where the slow log tells you which *commands*, the latency monitor also covers stalls no command is responsible for:
`fork` (Level 4 · 01), `aof-fsync-always`, `expire-cycle`, `eviction-del`. Turn it on in production with a threshold a
little above your normal p99. It's cheap.

## commandstats and latency percentiles: the aggregate view

`INFO commandstats` accumulates calls and time per command since the last `CONFIG RESETSTAT`:

```text
cmdstat_hgetall:calls=2,usec=76755,usec_per_call=38377.50,rejected_calls=0,failed_calls=0,slowlog_count=2,slowlog_time_ms_sum=76.75,slowlog_time_ms_max=40.21
cmdstat_del:calls=2,usec=92676,usec_per_call=46338.00,rejected_calls=0,failed_calls=0,slowlog_count=2,slowlog_time_ms_sum=92.68,slowlog_time_ms_max=48.84
cmdstat_keys:calls=2,usec=47579,usec_per_call=23789.50,rejected_calls=0,failed_calls=0,slowlog_count=2,slowlog_time_ms_sum=47.58,slowlog_time_ms_max=24.20
cmdstat_eval:calls=1,usec=14929,usec_per_call=14929.00,rejected_calls=0,failed_calls=0,slowlog_count=1,slowlog_time_ms_sum=14.93,slowlog_time_ms_max=14.93
cmdstat_get:calls=20520,usec=30521,usec_per_call=1.49,rejected_calls=0,failed_calls=0
cmdstat_hset:calls=400000,usec=155547,usec_per_call=0.39,rejected_calls=0,failed_calls=0
```

This is the most useful single view for "where does Redis spend its time?". The 20,520 `GET`s together cost 30 ms. The
**two** `DEL`s cost 93 ms. Sort by `usec`, not by `calls`. On this version the slow-log counters are included per
command, so you can graph "slow calls of command X" directly. `rejected_calls` (refused before running: ACL, OOM,
wrong arity) and `failed_calls` (errored while running) are worth alerting on.

`INFO latencystats` gives per-command percentiles:

```text
latency_percentiles_usec_get:p50=1.003,p99=8.031,p99.9=15.039
latency_percentiles_usec_hgetall:p50=36700.159,p99=40370.175,p99.9=40370.175
latency_percentiles_usec_keys:p50=23461.887,p99=24248.319,p99.9=24248.319
```

A `GET`'s p99 of 8 µs says the server isn't the problem for normal reads. Anything slower that the clients see is
queueing behind the big commands, or the network.

## Finding the keys

```text
$ redis-cli -p 6495 --hotkeys
[00.00%] Hot key "item:185595" found so far with counter 5
...
$ redis-cli -p 6495 --memkeys
Biggest string found "item:185595" has 254 bytes
```

`--hotkeys` reads each key's LFU counter, so it needs an LFU eviction policy, which is why this server uses
`allkeys-lfu`. It found no key standing out (counters of 4–5 are random traffic), and the reason is instructive:
`hot:config` was requested 520 times but **never existed**, so it has no LFU counter. Such requests show up only in
`keyspace_misses`:

```text
keyspace_hits:620002
keyspace_misses:520
```

A hot *missing* key is a classic bug: a cache that's never populated, or a typo in a key name. Neither `--hotkeys` nor
`--bigkeys` will find it. `MONITOR` (briefly) or client-side logging will. `--memkeys` and `--bigkeys` find big keys
like `big:catalog` before they cause trouble.

## What to watch continuously

| Metric (`INFO`) | Why |
|---|---|
| `used_memory` vs `maxmemory`, `mem_fragmentation_ratio` | headroom, swapping (ratio < 1 on Linux), fragmentation |
| `evicted_keys`, `expired_keys` | eviction means the cache is too small, or wrongly configured |
| `keyspace_hits` / `keyspace_misses` | cache effectiveness, and missing hot keys |
| `instantaneous_ops_per_sec`, `cmdstat_*` usec | load and where time goes |
| `connected_clients`, `blocked_clients`, `rejected_connections` | connection leaks, `maxclients` (10,000 here) |
| `client_recent_max_output_buffer` | slow consumers, big replies |
| `rdb_last_bgsave_status`, `aof_last_write_status`, `latest_fork_usec` | persistence health and fork cost |
| `master_link_status`, replica offsets, `repl_backlog_*` | replication health and lag |
| `slowlog_count` per command, `LATENCY LATEST` | stalls |

Exporters for Prometheus and similar systems scrape these fields. Whatever you use, alert on trends and thresholds, not
just "is it up?".

## A troubleshooting checklist

1. **Is the server slow, or the client waiting?** Compare client-side latency with `latencystats`. If the server's p99
   is microseconds, look at queueing (slow log), the network, or the client (pool exhaustion, Level 1 · 09).
2. **Slow log** for O(N) commands on big keys, `KEYS`, long scripts and big deletes.
3. **Latency monitor** for forks, fsyncs and expiry or eviction stalls.
4. **Memory**: near `maxmemory`? Evicting? Swapping (`mem_fragmentation_ratio` < 1 on Linux)?
5. **Clients**: `CLIENT LIST` sorted by `omem` (output buffers) or `idle`. One subscriber or `MONITOR` can be the culprit.
6. **Persistence**: a fork every few seconds on a big instance? AOF `appendfsync always` on slow disks?
7. **The host**: CPU steal on VMs, transparent huge pages, swap. Redis can't see these directly, but
   `redis-cli --intrinsic-latency 30` *on the server host* measures how much latency the machine itself adds.

## How It Actually Works

Every command runs inside a wrapper that takes a monotonic timestamp before and after execution. The difference feeds
`commandstats` (sums), a per-command HDR histogram (`latencystats`, with percentiles chosen by
`latency-tracking-info-percentiles`), the slow log (if above the threshold) and the latency monitor's `command` event (if
above its threshold). That's why all four views agreed on 40–48 ms for the planted commands: they're the same
measurement aggregated differently. Other latency events (fork, fsync, expire cycle) are timed around those operations
in the same way.

The slow log is a fixed-length in-memory list. Arguments are truncated (32 arguments, 128 bytes each), so it's safe to
leave on, but entries scroll away quickly on busy servers, so ship them to your log system. The LFU counter that
`--hotkeys` reads is the 8-bit logarithmic counter from Level 3 · 02, exposed by `OBJECT FREQ`, which `--hotkeys` calls
for each key while scanning.

## Common mistakes

- **Raising `slowlog-log-slower-than` until the log is quiet.** Keep it at 10 ms or lower and investigate.
- **Reading only the slow log** when the issue is forks, fsyncs or eviction. Use the latency monitor too.
- **Sorting command stats by calls.** Time (`usec`) is what blocks the server.
- **Unnamed client connections.** Incidents take longer when you can't tell which service sent `KEYS *`.
- **Leaving `MONITOR` running**, which is itself a performance problem (Level 1 · 02).
- **Monitoring only availability.** Memory, evictions, replication and persistence fail silently.

## Exercise

1. Set `slowlog-log-slower-than 0` for 10 seconds on a busy lab server (it logs everything), then summarize which
   commands and clients dominate. Set it back afterwards.
2. Reproduce a fork latency event: populate 2 GB and run `BGSAVE` with `latency-monitor-threshold 10`. What does
   `LATENCY DOCTOR` advise?
3. Write a small exporter that prints, every 5 seconds, the hit ratio, ops/s, used memory, evicted keys and the top 3
   commands by `usec` (deltas, not totals).
4. Plant a hot missing key in an application and find it two ways: with `keyspace_misses` trends and with a 2-second
   `MONITOR` capture piped through `sort | uniq -c | sort -rn | head`.
