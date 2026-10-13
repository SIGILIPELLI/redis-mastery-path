---
description: "Redis Level 1: a local lab, redis-cli and RESP, strings, expiry, safe keyspace iteration, hashes, lists, sets, sorted sets, client libraries and a project."
---

# Level 1 · Redis Foundations

Level 1 takes you from "Redis is a cache" to writing correct application code against it. That's more than
learning `GET` and `SET`. You need to know why one `KEYS *` made an unrelated `PING` wait 149 ms, why a plain
`SET` quietly removed a session's expiry, why read-then-write counters lost 89% of their increments, and why
a client library's helpful automatic retry applied one `INCR` twice.

Every lesson is built from commands run against a real server, **Redis 8.10.2**, and every output shown is what
that server (or the client library) actually returned, errors included.

**Prerequisites:** basic programming in Python or JavaScript, and comfort at a terminal. No Redis experience is
needed. If you know a relational database, you'll see where Redis's model differs.

## Modules

1. **[What Redis Is: One Thread, Memory & a Local Lab](01-what-redis-is-and-a-lab.md)** — install Redis, start a
   throwaway server, measure memory per key, and watch one slow command block every client.
2. **[redis-cli & the RESP Protocol](02-redis-cli-and-resp.md)** — speak RESP2 and RESP3 by hand, raw vs
   formatted output, `--pipe` loading 100,000 keys in 79 ms, `MONITOR` and `--bigkeys`.
3. **[Strings, Counters & Expiry](03-strings-counters-expiry.md)** — `SET`'s options including 8.4's `IFEQ`,
   atomic counters vs lost updates, float pitfalls, TTLs and the overwrite trap.
4. **[The Keyspace: Naming, SCAN & Safe Deletes](04-keyspace-scan-delete.md)** — key naming, cursor iteration
   and its guarantees, `UNLINK` vs `DEL` on a 2-million-field hash, pattern deletes that don't block.
5. **[Hashes & Per-Field Expiry](05-hashes.md)** — objects as hashes, `HEXPIRE`/`HSETEX`/`HGETDEL`, and why a
   small hash beat both JSON and separate keys on memory.
6. **[Lists: Queues, Stacks & Blocking Pops](06-lists-queues.md)** — capped lists, which list commands are really
   O(N), `BLPOP` workers and `LMOVE` for safer handoff.
7. **[Sets: Membership, Set Algebra & Random Picks](07-sets.md)** — dedupe with `SADD`, intersections and
   `SINTERCARD`, random draws, and the cost of algebra on million-member sets.
8. **[Sorted Sets: Rankings, Ranges & Time Windows](08-sorted-sets.md)** — leaderboards, `ZADD` flags, score and
   lex ranges, delayed jobs, sliding windows and O(log N) measured.
9. **[Clients: redis-py & node-redis](09-clients-python-node.md)** — shared clients and pools, RESP3 defaults,
   pipelines vs transactions, and timeouts with retries that can double-apply writes.
10. **[Project — URL Shortener with Click Analytics](10-project-url-shortener.md)** — a tested shortener using
    every type in this level, including the overwrite bug its tests caught.

## After Level 1

You can store and change data correctly with Redis's five core types. Level 2 goes underneath and beyond them:
how each type is encoded in memory and how to shrink it, bitmaps and HyperLogLog for counting at scale,
geospatial indexes, streams with consumer groups, Pub/Sub, and the two ways to make several commands atomic,
`MULTI`/`WATCH` and server-side Lua functions.
