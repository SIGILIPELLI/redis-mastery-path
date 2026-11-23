---
title: "Redis Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Redis cheat sheet: 15 quick-reference tables from every level of the Redis Mastery Path, each linked to its full lesson."
---

# Redis Cheat Sheet

Quick-reference tables from **15 lessons** of the free Redis Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Redis Foundations

### [02 · redis-cli &amp; the RESP Protocol](level-1/02-redis-cli-and-resp.md)

*Speaking RESP by hand*

| Prefix | RESP2 type | Example above |
|--------|------------|---------------|
| `+` | simple string (status) | `+PONG`, `+OK` |
| `-` | error | `-ERR value is not an integer…` |
| `:` | integer | `:1` from `INCR` |
| `$` | bulk string (length-prefixed, binary-safe) | `$5\r\nPorto` |
| `$-1` | null bulk string | `GET missing` |
| `*` | array | replies to `LRANGE`, `HGETALL`… |

### [05 · Hashes &amp; Per-Field Expiry](level-1/05-hashes.md)

*Hash or separate keys or a JSON string?*

| | Separate keys | JSON string | Hash |
|---|---|---|---|
| Read one attribute | `GET` | fetch and parse all of it | `HGET` |
| Update one attribute atomically | `SET` | needs a script or `WATCH` | `HSET` / `HINCRBY` |
| TTL granularity | per attribute | whole object | whole key, or per field (7.4+) |
| Nested data | no | yes | no (one level, values are strings) |

### [10 · Project — URL Shortener with Click Analytics](level-1/10-project-url-shortener.md)

| Need | Structure | Why this one |
|------|-----------|--------------|
| Next numeric id | string + `INCR` | atomic, gap-tolerant counter |
| A link's data | hash `short:link:<code>` | one object, per-field updates, `HINCRBY` clicks |
| Claim a code or alias once | `HSETNX` on the link hash | first writer wins, atomically |
| Same URL → same code | string `short:by-target:<sha256>` + `SET NX` | dedupe without scanning |
| Last 20 clicks | list + `LPUSH`/`LTRIM` | capped, newest first |
| Unique visitors per day | set per code per day, with TTL | `SADD` dedupes, `SCARD` counts |
| Top links today | sorted set per day | `ZINCRBY` + `ZRANGE … REV` |

## Level 2 · Data Structures in Depth

### [01 · Encodings &amp; Memory](level-2/01-encodings-memory.md)

*A survey of encodings*

| Type | Compact encoding | General encoding | Switch when (defaults on 8.10) |
|------|------------------|------------------|--------------------------------|
| string | `int`, `embstr` (≤44 bytes) | `raw` | non-integer, longer, or modified in place |
| list | `listpack` | `quicklist` (linked listpacks) | list exceeds one listpack node (`list-max-listpack-size`, 8 KB) |
| set | `intset`, `listpack` | `hashtable` | >512 ints, >128 members, or a member >64 bytes |
| hash | `listpack` (`listpackex` with field TTLs) | `hashtable` | >512 fields or a value >64 bytes |
| zset | `listpack` | `skiplist` + hash table | >128 members or a member >64 bytes |
| stream | radix tree of listpacks | (always) | — |

### [03 · HyperLogLog, Bloom Filters &amp; Other Sketches](level-2/03-hyperloglog-probabilistic.md)

*Choosing*

| Question | Structure | Memory here | Error |
|---|---|---|---|
| How many distinct? | HyperLogLog | 12 KB at 10M items | ~0.81% std error, no listing |
| Seen before? (append-only) | Bloom filter | 1.4 MB per 1M items at 1% | false positives only |
| Seen before? (with deletes) | Cuckoo filter | similar | false positives only |
| How often has X occurred? | Count-Min Sketch | 56 KB here | overestimates only |
| What are the heaviest hitters? | Top-K | 2.7 KB for top 5 | may miss near-ties |
| Exact anything | set / hash / sorted set | 36–125 MB here | none |

### [08 · Transactions: MULTI, EXEC &amp; WATCH](level-2/08-transactions-watch.md)

*Transactions vs pipelines*

| | Pipeline (`transaction=False`) | `MULTI`/`EXEC` |
|---|---|---|
| Fewer round trips | yes | yes, if sent pipelined |
| Other clients can interleave | yes, between commands | no |
| All queued commands run | yes | yes (unless `WATCH` fails or queuing failed) |
| Rollback on error | no | no |

## Level 3 · Caching & Application Patterns

### [01 · Caching Patterns: Cache-Aside to Write-Behind](level-3/01-caching-patterns.md)

*Choosing*

| Pattern | Who fills the cache | Staleness risk | Good for |
|---|---|---|---|
| Cache-aside + TTL | readers, on miss | up to TTL after a write (race) | most read-heavy data |
| Cache-aside + versioned write-back | readers, conditionally | until next read | data where stale is harmful |
| Write-through | writers | race still possible | read-after-write heavy data |
| Write-behind | app writes Redis, job writes DB | DB lags; loss on Redis failure | counters, analytics |

### [02 · maxmemory, Eviction &amp; How Expiry Really Works](level-3/02-eviction-expiry-internals.md)

*The eviction policies*

| Policy | Evicts from | Chooses |
|---|---|---|
| `noeviction` | — | refuses writes |
| `allkeys-lru` | all keys | least recently used (approximate) |
| `allkeys-lfu` | all keys | least frequently used (approximate) |
| `allkeys-random` | all keys | random |
| `volatile-lru` / `volatile-lfu` / `volatile-random` | only keys with a TTL | as above |
| `volatile-ttl` | only keys with a TTL | shortest remaining TTL |

### [03 · Cache Stampedes &amp; Invalidation](level-3/03-stampedes-invalidation.md)

*Steady traffic: who waits?*

| Strategy | DB calls | Requests that waited > 100 ms |
|---|---|---|
| Lock | 5 | 100 |
| XFetch (probabilistic early expiry) | 35 | 35 |
| Stale-while-revalidate | 5 | 20 |

### [04 · Rate Limiting Algorithms](level-3/04-rate-limiting.md)

*Which to use*

| Algorithm | Exact? | Bursts | Memory per client | Use for |
|---|---|---|---|---|
| Fixed window | no, up to 2× at boundaries | uncontrolled | tiny | coarse quotas (per day), where 2× briefly is fine |
| Sliding log | yes | none above limit | O(limit) | low limits that must be exact (login attempts) |
| Sliding window counter | nearly | small overshoot | tiny | most API rate limits |
| Token bucket | to its definition | up to capacity | tiny | APIs that should allow short bursts |

### [07 · Reliable Job Queues](level-3/07-job-queues.md)

*Lists or streams?*

| | List + BLMOVE (this lesson) | Stream + consumer group (Level 2 · 06) |
|---|---|---|
| Crash recovery | your reaper + heartbeats | built in: PEL + `XAUTOCLAIM` |
| Delivery count | you track `attempts` in the payload | built in (`times_delivered`) |
| History / replay | gone after ack | kept until trimmed |
| Several independent consumers of the same jobs | no | yes, one group each |
| Ordering | FIFO | ID order |
| Simplicity | fewer concepts | more concepts, less code |

### [08 · JSON Documents](level-3/08-json-documents.md)

*Why not just store a JSON string?*

| | JSON string | Hash | JSON type |
|---|---|---|---|
| Nested data | yes | no | yes |
| Atomic update of one nested field | no (needs WATCH/Lua) | top-level fields only | yes |
| Read one field of a big doc | fetch everything | `HGET` | path |
| Memory | smallest | small (listpack) | about 2× the text here |
| Indexable by the query engine | no | yes | yes |

## Level 4 · Operations & Production

### [02 · Persistence II: AOF, fsync &amp; Hybrid Persistence](level-4/02-aof-durability.md)

*What does a crash actually lose?*

| Failure | `no` | `everysec` | `always` |
|---|---|---|---|
| Redis process crash / OOM kill / `kill -9` | nothing acknowledged lost | nothing lost | nothing lost |
| OS crash or power loss | up to ~30 s (OS-dependent) | up to ~1 s | nothing acknowledged |

### [08 · Monitoring &amp; Troubleshooting](level-4/08-monitoring-troubleshooting.md)

*What to watch continuously*

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

### [10 · Capstone — A Secured, Persistent, Failover-Tested Cluster](level-4/10-capstone-production-cluster.md)

*The design*

| Concern | Choice | From |
|---|---|---|
| Scale-out | 3 primaries × 1 replica, Redis Cluster | Level 4 · 05–06 |
| Partial availability | `cluster-require-full-coverage no` | Level 4 · 06 |
| Durability | AOF `everysec` + RDB `save 300 100` | Level 4 · 01–02 |
| Data-store memory policy | `maxmemory 256mb`, `noeviction` | Level 3 · 02 |
| Large-key safety | `lazyfree-lazy-user-del yes`, `lazyfree-lazy-expire yes` | Level 1 · 04 |
| Limit writes that can be lost | `min-replicas-to-write 1`, `min-replicas-max-lag 10` | Level 4 · 03 |
| Confirmed durability per write | application calls `WAIT 1` and checks the answer | Level 4 · 03 |
| Access control | `default` off; admin, replicator and app users in an ACL file | Level 4 · 07 |
| Network | `bind 127.0.0.1` (a private interface in real life) | Level 4 · 07 |
| Observability | `latency-monitor-threshold 25` | Level 4 · 08 |

---

Go deeper with the full [Redis Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
