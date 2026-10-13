---
description: "Redis Level 2: memory encodings, bitmaps, HyperLogLog and Bloom filters, geo, streams and consumer groups, Pub/Sub, transactions, Lua."
---

# Level 2 · Data Structures in Depth

Level 1 used Redis's core types correctly. Level 2 goes underneath them and beyond them. You'll see why one extra
sorted-set member made a key 12× larger, why a million small values fit in 7.9 MB instead of 46 MB, and how 10
million unique visitors fit in 12 KB. You'll also see why a stream reader that passes `$` loses half its messages,
why a slow subscriber gets cut off at 32 MB, and why a Lua script ran the same transfers 7× faster than `WATCH`.

Some results surprised us, and they're reported as found. On the Homebrew build of Redis 8.10.2 used here,
`BITCOUNT` returned wrong totals above 2^21. Lesson 02 shows the evidence, the comparison with Valkey, and a
workaround. Several commands used here are very recent (8.2's `XACKDEL`, 8.6's idempotent `XADD`, 8.8's `XNACK`),
and each lesson says which version introduced them.

**Prerequisites:** Level 1, or equivalent experience with Redis's strings, hashes, lists, sets and sorted sets and a
client library.

## Modules

1. **[Encodings & Memory](01-encodings-memory.md)** — listpack, intset, skiplist and quicklist, the thresholds
   between them, hash bucketing, and reading fragmentation in `INFO memory`.
2. **[Bitmaps & Bitfields](02-bitmaps-bitfields.md)** — daily-active-user bitmaps, `BITOP` set algebra, packed
   counters with overflow control, the sparse-offset trap, and a `BITCOUNT` bug found.
3. **[HyperLogLog, Bloom Filters & Other Sketches](03-hyperloglog-probabilistic.md)** — approximate counting and
   membership with measured error rates: HyperLogLog, Bloom, Cuckoo, Count-Min and Top-K.
4. **[Geospatial Indexes](04-geospatial.md)** — `GEOADD` and `GEOSEARCH`, nearest-N on a million points, why `ANY`
   matters, and the spherical model's 0.2% error.
5. **[Streams: An Append-Only Log](05-streams-basics.md)** — entry IDs, ranges, `XREAD` without losing messages,
   bounded trimming, and idempotent `XADD`.
6. **[Stream Consumer Groups & Recovery](06-streams-consumer-groups.md)** — the pending entries list, `XACK`,
   recovering a crashed worker with `XAUTOCLAIM`, dead letters, `XACKDEL` and `XNACK`.
7. **[Pub/Sub, Sharded Pub/Sub & Keyspace Notifications](07-pubsub-notifications.md)** — broadcast messaging,
   slow-subscriber disconnects, sharded channels, and how late expiry events really are.
8. **[Transactions: MULTI, EXEC & WATCH](08-transactions-watch.md)** — what a Redis transaction guarantees (no
   rollback), and optimistic locking that took overdrafts from 2,815 to zero.
9. **[Lua Scripts & Redis Functions](09-lua-functions.md)** — atomic server-side logic, conversion traps, `BUSY`
   scripts that can't be killed, and persistent function libraries.
10. **[Project — Game Leaderboard & Activity Feed](10-project-leaderboard-feed.md)** — a Functions-based weekly
    leaderboard with first-come tie-breaking, a trimmed activity stream and an idempotent badge worker.

## After Level 2

You know what each structure costs and how to compose them atomically. Level 3 applies that to the jobs Redis is
most often hired for: caching (and the stampedes, invalidation bugs and eviction surprises that come with it), rate
limiting, distributed locks and their limits, sessions, reliable job queues, JSON documents, and the query engine's
secondary and vector indexes.
