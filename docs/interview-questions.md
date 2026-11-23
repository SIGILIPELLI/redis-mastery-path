---
title: "Redis Interview Questions and Answers (180 Questions)"
description: "180 Redis interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Redis Mastery Path."
---

# Redis Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Redis Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Redis Foundations

### [01 · What Redis Is: One Thread, Memory &amp; a Local Lab](level-1/01-what-redis-is-and-a-lab.md)

**Q1.** What is Redis at its core?

**Answer:** An in-memory data structure server that holds keys mapped to typed values such as strings, lists, sets and hashes

Data lives in RAM, which is why access is fast, and persistence is optional.

**Q2.** Why is Redis so fast for simple commands?

**Answer:** Data is in memory and commands are executed one at a time by a single main thread without lock contention

Single-threaded command execution also makes each command atomic.

**Q3.** What is the main cost of keeping the dataset in RAM?

**Answer:** Capacity is limited by memory, so you must plan sizing, eviction or persistence deliberately

Memory is more expensive than disk.

**Q4.** Which is a typical Redis use?

**Answer:** Caching, sessions, counters, queues, leaderboards and rate limiting

It often sits beside a primary database rather than replacing it.

### [02 · redis-cli &amp; the RESP Protocol](level-1/02-redis-cli-and-resp.md)

**Q5.** What does this print?

```text
SET greeting hello
GET greeting
STRLEN greeting
EXISTS greeting
EXISTS nothere
```

**Answer:** OK
hello
5
1
0

EXISTS returns the number of keys that exist; STRLEN counts bytes. Verified by running it against a real Redis 8 server.

**Q6.** Are Redis command names case-sensitive?

**Answer:** No: commands are case-insensitive, but key names are case-sensitive

SET and set are the same command; user:1 and User:1 are different keys.

**Q7.** What is RESP?

**Answer:** The Redis serialization protocol clients use to talk to the server

redis-cli speaks RESP and prints a friendly version.

**Q8.** What should a healthy server answer to PING?

**Answer:** PONG

A quick liveness check.

### [03 · Strings, Counters &amp; Expiry](level-1/03-strings-counters-expiry.md)

**Q9.** What does this print?

```text
SET n 10
INCR n
INCRBY n 5
DECR n
GET n
```

**Answer:** OK
11
16
15
15

INCR on a string that holds an integer is atomic, so concurrent clients never lose updates. Verified by running it against a real Redis 8 server.

**Q10.** What does TTL return for a key with no expiry and for a missing key?

```text
SET a 1
TTL a
TTL missing
```

**Answer:** OK
-1
-2

-1 means the key exists without an expiry; -2 means the key does not exist. Verified by running it against a real Redis 8 server.

**Q11.** What does this print?

```text
SET k v EX 100
SET k w
TTL k
```

**Answer:** OK
OK
-1

A plain SET overwrites the value and removes the old expiry unless you use KEEPTTL. Verified by running it against a real Redis 8 server.

**Q12.** What does SET ... NX do?

```text
SET lock a NX
SET lock b NX
GET lock
```

**Answer:** OK

a

NX sets only if the key does not exist; the second call returns nil (an empty line in raw output) and the value stays a. Verified by running it against a real Redis 8 server.

### [04 · The Keyspace: Naming, SCAN &amp; Safe Deletes](level-1/04-keyspace-scan-delete.md)

**Q13.** Why is KEYS * dangerous on a production server?

**Answer:** It scans the whole keyspace in one blocking call and stalls other clients

Use SCAN instead.

**Q14.** What is true of SCAN?

**Answer:** It iterates incrementally using a cursor, may return a key more than once, and may miss keys added or deleted during the scan

Clients must tolerate duplicates.

**Q15.** What does UNLINK do differently from DEL?

**Answer:** It removes the key immediately from the keyspace and frees the memory in a background thread

Good for large values.

**Q16.** Which key naming style is recommended?

**Answer:** Structured names such as user:42:profile

Consistent prefixes make SCAN MATCH and monitoring easier.

### [05 · Hashes &amp; Per-Field Expiry](level-1/05-hashes.md)

**Q17.** What does this print?

```text
HSET user:1 name Ana age 30
HGET user:1 name
HINCRBY user:1 age 1
HLEN user:1
```

**Answer:** 2
Ana
31
2

HSET returns the number of fields added. Verified by running it against a real Redis 8 server.

**Q18.** Why use a hash for an object instead of a JSON string?

**Answer:** You can read or update single fields atomically without rewriting the whole value

Hash fields are flat strings.

**Q19.** Why can HGETALL on a huge hash be a problem?

**Answer:** It returns every field in one blocking call; use HSCAN for large hashes

Large replies take time and bandwidth.

**Q20.** Which command increments a numeric field inside a hash atomically?

**Answer:** HINCRBY

HINCRBYFLOAT is the floating-point version.

### [06 · Lists: Queues, Stacks &amp; Blocking Pops](level-1/06-lists-queues.md)

**Q21.** What does this print?

```text
RPUSH q a b c
LPOP q
LRANGE q 0 -1
LLEN q
```

**Answer:** 3
a
b
c
2

RPUSH + LPOP gives FIFO order; LRANGE after the pop shows b and c, so the length is 2. Verified by running it against a real Redis 8 server.

**Q22.** Which command blocks until an item is available?

**Answer:** BLPOP (or BRPOP)

Workers use blocking pops to wait for jobs without polling.

**Q23.** How do you keep only the latest 100 items in a list?

**Answer:** LPUSH followed by LTRIM key 0 99

LTRIM keeps a range.

**Q24.** What is a drawback of a plain list as a job queue?

**Answer:** A popped job is gone if the worker crashes before finishing, so reliable queues use LMOVE or streams

Reliable patterns move the job to a processing list.

### [07 · Sets: Membership, Set Algebra &amp; Random Picks](level-1/07-sets.md)

**Q25.** What does this print?

```text
SADD s a b c
SADD s a
SISMEMBER s b
SCARD s
```

**Answer:** 3
0
1
3

Sets ignore duplicates; SADD returns the number of new members. Verified by running it against a real Redis 8 server.

**Q26.** What does SINTER return for these sets (sorted for stability)?

```text
SADD a 1 2 3
SADD b 2 3 4
SINTER a b
```

**Answer:** 3
3
2
3

The intersection of {1,2,3} and {2,3,4} is {2,3}. The first two lines are the SADD counts. Verified by running it against a real Redis 8 server.

**Q27.** Which structure fits 'unique visitors of a page'?

**Answer:** A set (or a HyperLogLog if approximate is fine)

Sets give exact uniqueness at a memory cost.

**Q28.** What does SPOP do?

**Answer:** Removes and returns random member(s)

SRANDMEMBER returns without removing.

### [08 · Sorted Sets: Rankings, Ranges &amp; Time Windows](level-1/08-sorted-sets.md)

**Q29.** What does this print?

```text
ZADD board 100 ana 250 bo 175 cy
ZREVRANGE board 0 1 WITHSCORES
ZRANK board ana
```

**Answer:** 3
bo
250
cy
175
0

ZREVRANGE lists highest first; ZRANK is the 0-based ascending rank of ana, who has the lowest score. Verified by running it against a real Redis 8 server.

**Q30.** How are members with equal scores ordered?

**Answer:** Lexicographically by member

Sorted sets order by score, then member.

**Q31.** Which command adds to a member's score atomically?

**Answer:** ZINCRBY

Ideal for leaderboards.

**Q32.** What are sorted sets best for?

**Answer:** Leaderboards, priority queues and time-ordered indexes

Skiplist plus hash give O(log N) updates.

### [09 · Clients: redis-py &amp; node-redis](level-1/09-clients-python-node.md)

**Q33.** Why should an application create one Redis client or pool and reuse it?

**Answer:** Opening a connection per request is slow and exhausts server connections

Share a client or a connection pool.

**Q34.** In redis-py, what does decode_responses=True do?

**Answer:** Returns str instead of bytes

Otherwise values arrive as bytes.

**Q35.** What should a client do when Redis is temporarily unavailable?

**Answer:** Use timeouts and retries with backoff, and decide on a fallback

Cache failures should degrade gracefully.

**Q36.** Which are well-known Node.js Redis clients?

**Answer:** node-redis and ioredis

Both support promises.

### [10 · Project — URL Shortener with Click Analytics](level-1/10-project-url-shortener.md)

**Q37.** Why use INCR to generate short-URL IDs?

**Answer:** It is atomic, so concurrent requests never get the same ID

Then encode the number in base62.

**Q38.** How do you avoid overwriting an existing custom alias?

**Answer:** SET alias target NX and check the reply

NX makes the check-and-set atomic.

**Q39.** Where would you count clicks per short link?

**Answer:** HINCRBY on a hash or INCR on a counter key

Counters are atomic.

**Q40.** How do you let links expire?

**Answer:** Set a TTL with EXPIRE or SET ... EX

Redis removes expired keys itself.

## Level 2 · Data Structures in Depth

### [01 · Encodings &amp; Memory](level-2/01-encodings-memory.md)

**Q41.** What encoding does a small integer string use?

```text
SET n 42
OBJECT ENCODING n
SET s hello
OBJECT ENCODING s
```

**Answer:** OK
int
OK
embstr

Integers are stored as integers; short strings use embstr. Verified by running it against a real Redis 8 server.

**Q42.** What encoding does a small hash use?

```text
HSET h a 1 b 2
OBJECT ENCODING h
```

**Answer:** 2
listpack

Small collections use a compact listpack; large ones switch to hash tables. Verified by running it against a real Redis 8 server.

**Q43.** What does MEMORY USAGE key report?

**Answer:** Approximate bytes used by the key and its value

Use it to find expensive keys.

**Q44.** Why do many small keys cost more than one hash?

**Answer:** Each key has per-key overhead; small hashes pack fields compactly

Grouping can save memory.

### [02 · Bitmaps &amp; Bitfields](level-2/02-bitmaps-bitfields.md)

**Q45.** What does this print?

```text
SETBIT visits 7 1
SETBIT visits 9 1
GETBIT visits 7
GETBIT visits 8
BITCOUNT visits
```

**Answer:** 0
0
1
0
2

SETBIT returns the old bit value; BITCOUNT counts set bits. Verified by running it against a real Redis 8 server.

**Q46.** Why are bitmaps good for daily active users?

**Answer:** One bit per user ID gives exact per-day sets at tiny memory, and BITOP combines days

User IDs must be small integers.

**Q47.** What does BITOP AND do across two day bitmaps?

**Answer:** Finds users active on both days

OR would give either day.

**Q48.** What does BITFIELD let you do?

**Answer:** Read and write several fixed-width integers packed inside a string

Useful for compact counters.

### [03 · HyperLogLog, Bloom Filters &amp; Other Sketches](level-2/03-hyperloglog-probabilistic.md)

**Q49.** What does this print?

```text
PFADD v a b c a b
PFCOUNT v
PFADD v a
PFCOUNT v
```

**Answer:** 1
3
0
3

HyperLogLog counts distinct elements; re-adding existing ones changes nothing. Verified by running it against a real Redis 8 server.

**Q50.** What is HyperLogLog's trade-off?

**Answer:** About 0.81% standard error in exchange for a fixed size of up to about 12 KB

You cannot list the members.

**Q51.** Can you merge HyperLogLogs?

**Answer:** Yes with PFMERGE, giving the unique count of the union

Useful for daily to monthly roll-ups.

**Q52.** When is exact counting still necessary?

**Answer:** When you must list members or the count drives billing or security

Approximate for analytics, exact for money.

### [04 · Geospatial Indexes](level-2/04-geospatial.md)

**Q53.** Which order do GEOADD and GEOSEARCH use for coordinates?

**Answer:** Longitude first, then latitude

A frequent source of swapped-map bugs.

**Q54.** How is geo data stored?

```text
GEOADD places 13.361389 38.115556 palermo 15.087269 37.502669 catania
TYPE places
```

**Answer:** 2
zset

Geo members are stored in a sorted set using a geohash as the score. Verified by running it against a real Redis 8 server.

**Q55.** Which command finds members within a radius or box?

**Answer:** GEOSEARCH

It replaced GEORADIUS.

**Q56.** Can you remove a geo member with ZREM?

**Answer:** Yes, because it is a sorted set

There is no GEODEL.

### [05 · Streams: An Append-Only Log](level-2/05-streams-basics.md)

**Q57.** What does this print for entry IDs and length?

```text
XADD s 1-1 temp 20
XADD s 1-2 temp 21
XLEN s
XRANGE s - + COUNT 1
```

**Answer:** 1-1
1-2
2
1-1
temp
20

IDs are millisecondTime-sequence; XRANGE - + returns oldest first. Verified by running it against a real Redis 8 server.

**Q58.** What does XADD key * field value generate?

**Answer:** An auto-generated, increasing ID

Use explicit IDs only for imports.

**Q59.** How do you cap a stream's memory?

**Answer:** XTRIM (or MAXLEN on XADD)

Approximate trimming with ~ is cheaper.

**Q60.** How does a stream differ from a list queue?

**Answer:** It keeps entries after reading, supports IDs, ranges and consumer groups

Streams are an append-only log.

### [06 · Stream Consumer Groups &amp; Recovery](level-2/06-streams-consumer-groups.md)

**Q61.** What does a consumer group give you?

**Answer:** Each entry is delivered to one consumer of the group, with acknowledgement tracking

Use XREADGROUP.

**Q62.** What is the pending entries list (PEL)?

**Answer:** Entries delivered to a consumer but not yet acknowledged with XACK

Inspect with XPENDING.

**Q63.** A consumer crashed with unacked messages. How do other consumers take over?

**Answer:** XAUTOCLAIM or XCLAIM entries idle longer than a threshold

This gives at-least-once processing.

**Q64.** Why must handlers be idempotent with consumer groups?

**Answer:** Entries can be redelivered after failures, so processing may happen more than once

At-least-once, not exactly-once.

### [07 · Pub/Sub, Sharded Pub/Sub &amp; Keyspace Notifications](level-2/07-pubsub-notifications.md)

**Q65.** Is Pub/Sub durable?

**Answer:** No: a message is delivered only to currently connected subscribers and is not stored

Use streams when you need replay.

**Q66.** A subscriber disconnects for a minute. What happens to messages published meanwhile?

**Answer:** It misses them

Fire-and-forget.

**Q67.** Are keyspace notifications enabled by default?

**Answer:** No, you enable them with notify-keyspace-events

They also have overhead and are best-effort.

**Q68.** Which command publishes a message?

**Answer:** PUBLISH channel message

It returns the number of receiving subscribers.

### [08 · Transactions: MULTI, EXEC &amp; WATCH](level-2/08-transactions-watch.md)

**Q69.** What does this print?

```text
MULTI
SET a 1
INCR a
EXEC
```

**Answer:** OK
QUEUED
QUEUED
OK
2

Commands are queued and executed together; EXEC returns one reply per command. Verified by running it against a real Redis 8 server.

**Q70.** What happens if one command in a transaction fails at runtime?

```text
SET s x
MULTI
INCR s
SET ok 1
EXEC
GET ok
```

**Answer:** OK
OK
QUEUED
QUEUED
ERR value is not an integer or out of range

OK
1

Redis does not roll back: the failing command returns an error and the others still run. Verified by running it against a real Redis 8 server.

**Q71.** What does WATCH implement?

**Answer:** Optimistic locking: EXEC aborts if a watched key changed

Retry the transaction when EXEC returns nil.

**Q72.** Are MULTI/EXEC transactions isolated from other clients' commands?

**Answer:** Yes, other commands never run in the middle of EXEC

But there is no rollback.

### [09 · Lua Scripts &amp; Redis Functions](level-2/09-lua-functions.md)

**Q73.** What does this Lua script return?

```text
EVAL "return redis.call('INCRBY', KEYS[1], ARGV[1])" 1 counter 5
EVAL "return redis.call('INCRBY', KEYS[1], ARGV[1])" 1 counter 3
```

**Answer:** 5
8

The script runs atomically; KEYS and ARGV carry key names and arguments. Verified by running it against a real Redis 8 server.

**Q74.** Why pass key names through KEYS instead of hard-coding them?

**Answer:** So the server (and cluster) know which keys the script touches

Required for cluster routing.

**Q75.** What is the risk of a long-running script?

**Answer:** It blocks the whole server while it runs

Keep scripts short.

**Q76.** How do Redis Functions differ from EVAL?

**Answer:** Functions are named, loaded libraries that persist and replicate with the data

FUNCTION LOAD.

### [10 · Project — Game Leaderboard &amp; Activity Feed](level-2/10-project-leaderboard-feed.md)

**Q77.** How would you build a leaderboard that returns the top 10 and a user's rank?

**Answer:** A sorted set with ZREVRANGE for the top 10 and ZREVRANK for the user

Both are O(log N).

**Q78.** Why combine a sorted set with a hash in the project?

**Answer:** The sorted set ranks IDs, the hash holds each player's details

Keep rank data separate from the profile.

**Q79.** How would you build a bounded activity feed per user?

**Answer:** LPUSH/XADD then LTRIM or XTRIM to cap size

Cap to protect memory.

**Q80.** When should the feed use a stream instead of a list?

**Answer:** When you need IDs, ranges by time and consumer groups

Streams are a log.

## Level 3 · Caching & Application Patterns

### [01 · Caching Patterns: Cache-Aside to Write-Behind](level-3/01-caching-patterns.md)

**Q81.** What is the cache-aside pattern?

**Answer:** The app reads the cache first, on a miss loads from the database and writes the value into the cache

The application owns the logic.

**Q82.** What is write-through?

**Answer:** Writes go to the cache and the database together so the cache stays current

Write-behind delays the database write.

**Q83.** Why always set a TTL on cache entries?

**Answer:** To bound staleness and memory, even if invalidation is missed

TTL is the safety net.

**Q84.** What is negative caching?

**Answer:** Caching 'not found' results briefly to avoid repeated expensive misses

Use a short TTL.

### [02 · maxmemory, Eviction &amp; How Expiry Really Works](level-3/02-eviction-expiry-internals.md)

**Q85.** What does maxmemory-policy allkeys-lru do when memory is full?

**Answer:** Evicts the least recently used keys from all keys

noeviction rejects writes instead.

**Q86.** Which policy evicts only keys that have an expiry?

**Answer:** The volatile-* policies such as volatile-lru or volatile-ttl

If nothing is evictable, writes fail.

**Q87.** How does Redis expire keys?

**Answer:** Lazily when accessed plus an active background sample of keys with TTLs

Expired keys may linger briefly.

**Q88.** Is Redis' LRU exact?

**Answer:** No, it samples a few keys and approximates LRU

maxmemory-samples tunes accuracy.

### [03 · Cache Stampedes &amp; Invalidation](level-3/03-stampedes-invalidation.md)

**Q89.** What is a cache stampede?

**Answer:** Many requests miss at once for a hot key and all hit the database

Often when a popular key expires.

**Q90.** Which technique reduces stampedes?

**Answer:** Single-flight locking or request coalescing, plus TTL jitter and early refresh

Only one request rebuilds the value.

**Q91.** Why add random jitter to TTLs?

**Answer:** So many keys do not all expire at the same moment

Spread the load.

**Q92.** What is a clean way to invalidate many related keys at once?

**Answer:** Include a version number in the key and bump it

Old versions simply expire.

### [04 · Rate Limiting Algorithms](level-3/04-rate-limiting.md)

**Q93.** What does a fixed-window counter print?

```text
INCR rl:user1
EXPIRE rl:user1 60
INCR rl:user1
INCR rl:user1
TTL rl:user1
```

**Answer:** 1
1
2
3
60

INCR counts requests; the EXPIRE gives the window. (TTL may read 59 on a slow machine.) Verified by running it against a real Redis 8 server.

**Q94.** What is the weakness of a fixed-window limiter?

**Answer:** A client can send a burst at the end of one window and the start of the next

Sliding windows fix it.

**Q95.** How does a sliding-window limiter with a sorted set work?

**Answer:** Add each request timestamp, remove old ones with ZREMRANGEBYSCORE and count with ZCARD

Run it atomically.

**Q96.** Why is a Lua script (or MULTI) useful for rate limiting?

**Answer:** The check-and-increment must be atomic or concurrent requests slip through

Avoid read-modify-write races.

### [05 · Distributed Locks &amp; Their Limits](level-3/05-distributed-locks.md)

**Q97.** What is the correct way to acquire a simple lock?

**Answer:** SET lock:key &lt;unique token&gt; NX PX &lt;ttl&gt;

NX plus an expiry in one atomic command.

**Q98.** Why release a lock with a script that checks the token?

**Answer:** So you never delete a lock that expired and is now held by another client

Compare the value, then DEL, atomically.

**Q99.** What is a fencing token for?

**Answer:** A monotonically increasing number the protected resource checks so stale lock holders are rejected

Locks with TTLs can expire during pauses.

**Q100.** When is a Redis lock NOT safe enough?

**Answer:** When correctness (not just efficiency) depends on it, because of expiry, failover and clock issues

Use the database or a consensus system for strict guarantees.

### [06 · Sessions &amp; Client-Side Caching](level-3/06-sessions-client-caching.md)

**Q101.** How would you store a web session?

**Answer:** A hash or string under a random session ID with a TTL that refreshes on activity

Sliding expiration with EXPIRE.

**Q102.** What does CLIENT TRACKING enable?

**Answer:** Server-assisted client-side caching with invalidation messages

The server tells the client when a key changed.

**Q103.** Why is a random, unguessable session ID important?

**Answer:** Guessable IDs let attackers hijack sessions

Use a secure random generator.

**Q104.** What belongs in a session?

**Answer:** Small identifiers and flags, not large data

Keep sessions small.

### [07 · Reliable Job Queues](level-3/07-job-queues.md)

**Q105.** Why is LMOVE/BLMOVE useful for a reliable queue?

**Answer:** It atomically moves a job to a processing list so it is not lost if the worker crashes

Requeue jobs stuck in processing.

**Q106.** What do libraries such as BullMQ or RQ add?

**Answer:** Retries, delays, priorities and visibility into jobs on top of Redis

Do not reinvent them.

**Q107.** How do you delay a job?

**Answer:** Store it in a sorted set scored by run time and move due jobs to the queue

Poll ZRANGEBYSCORE.

**Q108.** What happens to jobs that keep failing?

**Answer:** Move them to a dead-letter list after a retry limit

Inspect and fix them.

### [08 · JSON Documents](level-3/08-json-documents.md)

**Q109.** What does JSON.SET let you do?

**Answer:** Store a JSON document and read or update a nested path atomically

Redis 8 includes JSON as part of the core.

**Q110.** Why use JSON.NUMINCRBY on a path?

**Answer:** Atomically update a nested number without rewriting the document

Path-based updates.

**Q111.** When is a hash a better choice than JSON?

**Answer:** For flat objects with simple fields

Hashes are simpler and lighter.

**Q112.** How do you index JSON for queries?

**Answer:** Create a search index over JSON paths with FT.CREATE

Then query with FT.SEARCH.

### [09 · Query Engine: Secondary Indexes &amp; Vector Search](level-3/09-search-vector.md)

**Q113.** What does FT.CREATE do?

**Answer:** Defines a secondary index over documents with a given prefix and schema

Then use FT.SEARCH.

**Q114.** What is KNN vector search?

**Answer:** Finding the K stored vectors closest to a query vector

Used for similarity and embeddings.

**Q115.** What is the trade-off between FLAT and HNSW vector indexes?

**Answer:** FLAT is exact but scans everything; HNSW is approximate but much faster at scale

Pick by dataset size and recall needs.

**Q116.** What must stay consistent between stored vectors and the query?

**Answer:** Dimension and embedding model (and the distance metric)

Mixed models give meaningless distances.

### [10 · Project — Caching &amp; Rate-Limiting Layer for an API](level-3/10-project-api-cache-limiter.md)

**Q117.** In the API cache and limiter project, which layer should reject excess traffic?

**Answer:** The limiter, before the cache or database are touched

Protect expensive work first.

**Q118.** How does the project avoid caching errors forever?

**Answer:** Use short TTLs for errors or do not cache them

Errors are often transient.

**Q119.** Why add metrics for cache hit ratio?

**Answer:** To know whether the cache is helping and tune TTLs

Hits divided by hits plus misses.

**Q120.** What is a sensible fallback if Redis is down?

**Answer:** Serve directly from the source and fail open or closed deliberately for the limiter

Decide per feature.

## Level 4 · Operations & Production

### [01 · Persistence I: RDB Snapshots &amp; fork()](level-4/01-rdb-snapshots.md)

**Q121.** What is an RDB file?

**Answer:** A compact point-in-time snapshot of the dataset

Created by SAVE, BGSAVE or save rules.

**Q122.** How does BGSAVE avoid blocking clients?

**Answer:** It forks a child process that writes the snapshot

Copy-on-write memory can spike.

**Q123.** What can you lose with RDB-only persistence?

**Answer:** Writes since the last snapshot

Snapshots are periodic.

**Q124.** Why are RDB files good for backups and restarts?

**Answer:** They are compact and load fast

Copy them off-box.

### [02 · Persistence II: AOF, fsync &amp; Hybrid Persistence](level-4/02-aof-durability.md)

**Q125.** What does appendfsync everysec risk?

**Answer:** Up to about one second of writes after a crash

A common balance.

**Q126.** What does appendfsync always do?

**Answer:** fsync after every write command: safest and slowest

Throughput drops.

**Q127.** What is an AOF rewrite?

**Answer:** Compacting the log into the shortest set of commands that rebuilds the dataset

BGREWRITEAOF.

**Q128.** Why do many production setups use both RDB and AOF?

**Answer:** RDB for fast restarts and backups, AOF for better durability

Redis can load the AOF on restart.

### [03 · Replication: PSYNC, the Backlog &amp; Consistency](level-4/03-replication.md)

**Q129.** Is Redis replication synchronous by default?

**Answer:** No, it is asynchronous so a failover can lose recent writes

WAIT can wait for acknowledgements but does not make it strictly consistent.

**Q130.** Which command makes a server a replica?

**Answer:** REPLICAOF host port

Use REPLICAOF NO ONE to promote.

**Q131.** What does the replication backlog allow?

**Answer:** A reconnecting replica to do a partial resync instead of a full one

Size it for outages.

**Q132.** Why direct reads to replicas with care?

**Answer:** Replicas can be slightly behind, so reads may be stale

Read-your-writes is not guaranteed.

### [04 · High Availability with Sentinel](level-4/04-sentinel.md)

**Q133.** What does Sentinel do?

**Answer:** Monitors masters and replicas and promotes a replica when the master fails

It also tells clients the current master.

**Q134.** Why run at least three Sentinels?

**Answer:** A majority must agree to avoid split-brain decisions

Use odd numbers on separate failure domains.

**Q135.** What is the Sentinel quorum?

**Answer:** The number of Sentinels that must agree the master is down

A majority is still needed to authorise the failover.

**Q136.** How should clients find the master with Sentinel?

**Answer:** Ask Sentinel for the current master address (use a Sentinel-aware client)

The address changes after failover.

### [05 · Redis Cluster: Hash Slots, MOVED &amp; ASK](level-4/05-cluster-architecture.md)

**Q137.** How many hash slots does Redis Cluster have?

**Answer:** 16384

Keys map by CRC16(key) mod 16384.

**Q138.** What does a hash tag like {user42} do?

**Answer:** Only the text inside braces is hashed, so related keys land in the same slot

Needed for multi-key operations.

**Q139.** What does MOVED mean?

**Answer:** The key lives on another node; the client should update its slot map and retry there

Cluster clients cache the map.

**Q140.** Why do multi-key commands need the same slot?

**Answer:** Each node only holds some slots and cannot atomically operate across nodes

Otherwise CROSSSLOT error.

### [06 · Cluster Operations: Resharding, Scaling &amp; Failover](level-4/06-cluster-operations.md)

**Q141.** What does ASK mean during a migration?

**Answer:** The key is being moved; ask the target node once without updating the slot map

Unlike MOVED, it is temporary.

**Q142.** Which tool manages cluster creation and resharding?

**Answer:** redis-cli --cluster

Also CLUSTER NODES to inspect.

**Q143.** What should happen before removing a node?

**Answer:** Reshard its slots to other nodes, then remove it

An empty primary can be deleted.

**Q144.** What is needed for cluster failover to work?

**Answer:** Replicas for each primary and a majority of primaries reachable

Cluster has its own failover.

### [07 · Security: ACLs, Protected Mode &amp; TLS](level-4/07-security-acl-tls.md)

**Q145.** Why never expose Redis directly to the internet?

**Answer:** It is designed for trusted networks; use a private network, auth and TLS

Open Redis instances are routinely abused.

**Q146.** What are ACLs?

**Answer:** Per-user rules limiting commands, keys and channels

ACL SETUSER.

**Q147.** Which is a good practice for the default user?

**Answer:** Disable it or give it a strong password and use named users with least privilege

Separate app, admin and replica users.

**Q148.** What does TLS protect?

**Answer:** Traffic in transit between clients, replicas and nodes

Encrypt backups separately.

### [08 · Monitoring &amp; Troubleshooting](level-4/08-monitoring-troubleshooting.md)

**Q149.** What does SLOWLOG show?

**Answer:** Recent commands that exceeded a time threshold

Tune slowlog-log-slower-than.

**Q150.** Why avoid MONITOR in production?

**Answer:** It streams every command and adds significant overhead

Use briefly if at all.

**Q151.** Which INFO fields help detect memory pressure?

**Answer:** used_memory, maxmemory, evicted_keys and fragmentation ratio

Alert on trends.

**Q152.** How can you find big keys?

**Answer:** redis-cli --bigkeys or --memkeys (and MEMORY USAGE)

Large keys cause latency spikes.

### [09 · Versions, Licensing, Valkey &amp; Upgrades](level-4/09-licensing-valkey-upgrades.md)

**Q153.** From the shipped license files, which license applies to Redis 7.2 and earlier?

**Answer:** BSD 3-Clause

Newer versions changed.

**Q154.** What is Valkey?

**Answer:** A BSD-licensed fork of Redis 7.2.4 under the Linux Foundation

Reports redis_version 7.2.4 for compatibility.

**Q155.** Why can't you assume RDB files are portable between Redis and Valkey?

**Answer:** Newer versions use incompatible RDB formats

Test migrations.

**Q156.** What is a safe upgrade approach?

**Answer:** Test in staging, upgrade replicas first, fail over, then upgrade the old primary, with backups

Read the release notes.

### [10 · Capstone — A Secured, Persistent, Failover-Tested Cluster](level-4/10-capstone-production-cluster.md)

**Q157.** What is the capstone goal?

**Answer:** A production-style cluster with persistence, security, monitoring and tested failover

Prove it works.

**Q158.** Which test best proves your HA setup?

**Answer:** Kill a primary and verify automatic failover and client recovery

Measure data loss and downtime.

**Q159.** What should your runbook cover?

**Answer:** Backups and restore, failover, scaling, upgrades and alerts

Practice the restore.

**Q160.** Why restore from backup as a test?

**Answer:** A backup you have never restored is not proven

Verify integrity regularly.

## Mixed questions across all levels

**Q161.** Why is INCR safe for concurrent counters?

**Answer:** Redis executes each command atomically on a single thread, so two clients never read the same old value

Atomic commands need no client-side locking.

**Q162.** You need 'unique visitors per day' for 50 million users where a 1% error is fine. Which structure?

**Answer:** HyperLogLog (about 12 KB)

Exact sets cost far more memory.

**Q163.** You need 'did user 1234567 visit today' for 50 million known integer IDs, exactly. Which structure?

**Answer:** A bitmap with one bit per user ID

Bitmaps answer membership exactly in about 6 MB.

**Q164.** What does this print?

```text
ZADD z 5 a 5 b 3 c
ZRANGE z 0 -1
ZREVRANK z a
```

**Answer:** 3
c
a
b
1

Equal scores are ordered lexicographically; ZREVRANK counts from the highest. Verified by running it against a real Redis 8 server.

**Q165.** Which command lets you cap a stream's memory?

**Answer:** XTRIM (or MAXLEN on XADD)

Streams are append-only logs and must be trimmed.

**Q166.** A consumer in a stream group crashes before XACK. What happens to its messages?

**Answer:** They stay in the pending list and can be claimed by another consumer (at-least-once delivery)

Handlers must be idempotent.

**Q167.** Why is MULTI/EXEC not a database-style transaction?

**Answer:** A runtime error in one command does not roll back the others

Redis queues commands and runs them without interleaving, but does not undo.

**Q168.** What does WATCH do?

**Answer:** Aborts the next EXEC if a watched key was modified by another client (optimistic locking)

Retry on failure.

**Q169.** Which lock acquire command is correct?

**Answer:** SET lock:order:42 &lt;random token&gt; NX PX 30000

One atomic command with an expiry.

**Q170.** Why is KEYS * banned in production?

**Answer:** It walks the whole keyspace in one blocking call; use SCAN

SCAN may return duplicates.

**Q171.** What does maxmemory-policy noeviction do when memory is full?

**Answer:** Rejects write commands that need more memory

Choose an eviction policy for caches.

**Q172.** A hot cache key expires and 5,000 requests hit the database at once. What is this and one fix?

**Answer:** A cache stampede; use single-flight locking or refresh early with TTL jitter

Only one request should rebuild.

**Q173.** What is the downside of RDB-only persistence?

**Answer:** You can lose writes since the last snapshot

Add AOF for durability.

**Q174.** What risk does appendfsync everysec carry?

**Answer:** Up to about one second of acknowledged writes can be lost in a crash

always is safest but slowest.

**Q175.** Why can a failover lose acknowledged writes?

**Answer:** Replication is asynchronous, so the promoted replica may not have the latest writes

WAIT reduces but does not remove the risk.

**Q176.** Why do you need at least three Sentinels?

**Answer:** A majority must authorise a failover to avoid split-brain

Place them on separate failure domains.

**Q177.** How are keys assigned to slots in Redis Cluster?

**Answer:** CRC16(key) mod 16384, using only the {hash tag} part if present

Hash tags enable multi-key operations on one node.

**Q178.** What does a MOVED reply tell the client?

**Answer:** The slot lives on another node; update the slot map and retry there

ASK is the temporary version during migration.

**Q179.** What is a safe way to protect a Redis server?

**Answer:** Private network, ACL users with least privilege and TLS, never exposed publicly

Redis assumes a trusted network.

**Q180.** Why should you test restoring a backup?

**Answer:** A backup that has never been restored is unproven

Practice before an incident.

## Practical coding questions

### A fixed-window rate limiter in Lua

Write a Lua script that atomically counts requests per key, sets a window expiry on the first hit and allows only the first N requests. Use it for two users and inspect the counter and TTL. This runs against a real Redis server with redis-cli.

Starter code:

```
# Run these lines with redis-cli against an empty database.
# TODO: write a Lua script for EVAL that
#   1. INCRs KEYS[1]
#   2. sets EXPIRE KEYS[1] ARGV[1] seconds on the first hit only
#   3. returns 1 if the count is <= ARGV[2], otherwise 0
# Then call it 5 times for rl:ana (60 s window, limit 3), GET rl:ana,
# call it once for rl:bo, and finish with TTL rl:bo.
```

Expected output:

```text
1
1
1
0
0
5
1
60
```

??? success "Show a solution"

    ```
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:ana 60 3
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:ana 60 3
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:ana 60 3
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:ana 60 3
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:ana 60 3
    GET rl:ana
    EVAL "local c = redis.call('INCR', KEYS[1]) if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end if c > tonumber(ARGV[2]) then return 0 end return 1" 1 rl:bo 60 3
    TTL rl:bo
    ```

    Doing INCR, EXPIRE and the comparison inside one script makes the check-and-increment atomic, so concurrent requests cannot slip through. Calls 4 and 5 return 0 but still count, so GET shows 5. Verified by running it against Redis 8; your TTL may show 59 if a second passes.

### A reliable job queue with LMOVE

Model a queue where a claimed job moves to a processing list, is removed when finished and is returned to the queue if the worker crashes. Run the sequence of commands and compare your output.

Starter code:

```
# Reliable queue: a job must not be lost if a worker crashes mid-job.
# 1. Enqueue j1, j2, j3 onto the list "jobs".
# 2. A worker claims two jobs by moving them to the list "processing" (oldest first, appended at the end).
# 3. Show both lists.
# 4. The worker finishes j1: remove it from "processing" and show the list.
# 5. The worker crashes while handling j2: move it back from "processing" to the end of "jobs", show "jobs".
# 6. Finish with the length of "processing".
```

Expected output:

```text
3
j1
j2
j3
j1
j2
1
j2
j2
j3
j2
0
```

??? success "Show a solution"

    ```
    RPUSH jobs j1 j2 j3
    LMOVE jobs processing LEFT RIGHT
    LMOVE jobs processing LEFT RIGHT
    LRANGE jobs 0 -1
    LRANGE processing 0 -1
    LREM processing 1 j1
    LRANGE processing 0 -1
    LMOVE processing jobs LEFT RIGHT
    LRANGE jobs 0 -1
    LLEN processing
    ```

    LMOVE is atomic, so a job always lives in exactly one list: waiting, processing or (after LREM) done. A crash leaves the job in processing, where a janitor can move it back. Verified by running it against Redis 8.

---

More free practice: the [Redis Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
