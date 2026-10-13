---
description: "Redis Level 3: caching and stampedes, eviction and expiry, rate limiting, locks and fencing, sessions, job queues, JSON, search and vectors."
---

# Level 3 · Caching & Application Patterns

Level 3 is about the jobs Redis is hired for: caching, rate limiting, locking, sessions, job queues, documents and
search. Each one looks like a few lines of code, and each has a failure mode that only shows up under concurrency or
at the worst possible moment. That's why every lesson here reproduces the failure before fixing it:

- a stale price cached for five minutes, whether the writer deletes the key or overwrites it;
- 50 database calls for one expired key;
- a fixed-window limiter that let through 20 requests in 20 ms against a limit of 10;
- a correctly implemented lock that still let two clients write, until fencing tokens rejected the stale one;
- `kill -9` losing jobs from a `BLPOP` queue but none from a `BLMOVE` queue;
- an HNSW vector index that found only 68% of the true nearest neighbours with default settings.

Everything was run on **Redis 8.10.2** with redis-py 8.1.0, and the JSON, search and probabilistic modules loaded.
The project uses FastAPI 0.143.0 under Uvicorn 0.54.0.

**Prerequisites:** Levels 1 and 2. In particular, Level 2 · 08–09 on transactions and Lua scripts, which almost
every pattern here relies on.

## Modules

1. **[Caching Patterns: Cache-Aside to Write-Behind](01-caching-patterns.md)** — a cache at 97.7% hits in front of
   SQLite, the stale-write race that defeats delete-on-write, a versioned fix, write-through and write-behind.
2. **[maxmemory, Eviction & How Expiry Really Works](02-eviction-expiry-internals.md)** — OOM errors under
   `noeviction`, the `volatile-*` trap, LFU vs LRU measured, and how expired keys are reclaimed.
3. **[Cache Stampedes & Invalidation](03-stampedes-invalidation.md)** — rebuild locks, XFetch and
   stale-while-revalidate compared, TTL jitter, and invalidating groups of keys.
4. **[Rate Limiting Algorithms](04-rate-limiting.md)** — fixed window, sliding log, sliding window counter and token
   bucket as atomic scripts, with boundary bursts and memory measured.
5. **[Distributed Locks & Their Limits](05-distributed-locks.md)** — `SET NX PX` with tokens, safe release, the
   paused-holder failure, fencing tokens and the Redlock debate.
6. **[Sessions & Client-Side Caching](06-sessions-client-caching.md)** — secure session IDs, sliding and absolute
   expiry, rotation, logout-everywhere, and `CLIENT TRACKING` with redis-py's local cache.
7. **[Reliable Job Queues](07-job-queues.md)** — a `kill -9` crash test, processing lists, heartbeats, reapers,
   retries, dead letters, delays, and lists vs streams.
8. **[JSON Documents](08-json-documents.md)** — JSONPath reads, atomic nested updates, merge semantics, and when the
   JSON type is worth twice the memory.
9. **[Query Engine: Secondary Indexes & Vector Search](09-search-vector.md)** — TEXT/TAG/NUMERIC/GEO indexes,
   aggregations, a negation trap, FLAT vs HNSW recall, and vector sets.
10. **[Project — Caching & Rate-Limiting Layer for an API](10-project-api-cache-limiter.md)** — a tested FastAPI
    service with per-client limits, stampede-proof caching and stale-while-revalidate, plus the race its tests caught.

## After Level 3

Your application logic is correct. Level 4 makes the server it depends on survive: RDB snapshots and the AOF,
replication and failover with Sentinel, Redis Cluster and resharding, ACLs and TLS, monitoring and troubleshooting,
versions and licensing (including Valkey), and a capstone that builds a secured, persistent cluster, fails it over under
load, and measures exactly what was lost.
