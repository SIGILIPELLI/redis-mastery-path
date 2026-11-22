---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Redis Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Why is INCR safe for concurrent counters?",
    "options": [
      "It uses optimistic retries",
      "It is eventually consistent",
      "Redis executes each command atomically on a single thread, so two clients never read the same old value",
      "It locks the whole database"
    ],
    "correctIndex": 2,
    "explanation": "Atomic commands need no client-side locking."
  },
  {
    "question": "You need 'unique visitors per day' for 50 million users where a 1% error is fine. Which structure?",
    "options": [
      "HyperLogLog (about 12 KB)",
      "A hash per user",
      "A list",
      "A set of all IDs"
    ],
    "correctIndex": 0,
    "explanation": "Exact sets cost far more memory."
  },
  {
    "question": "You need 'did user 1234567 visit today' for 50 million known integer IDs, exactly. Which structure?",
    "options": [
      "A stream",
      "A list",
      "HyperLogLog",
      "A bitmap with one bit per user ID"
    ],
    "correctIndex": 3,
    "explanation": "Bitmaps answer membership exactly in about 6 MB."
  },
  {
    "question": "What does this print?\n```text\nZADD z 5 a 5 b 3 c\nZRANGE z 0 -1\nZREVRANK z a\n```\n",
    "options": [
      "3\nc\nb\na\n1",
      "3\nc\na\nb\n1",
      "3\nc\na\nb\n2",
      "3\na\nb\nc\n0"
    ],
    "correctIndex": 1,
    "explanation": "Equal scores are ordered lexicographically; ZREVRANK counts from the highest. Verified by running it against a real Redis 8 server."
  },
  {
    "question": "Which command lets you cap a stream's memory?",
    "options": [
      "EXPIRE on entries",
      "LTRIM",
      "DEL",
      "XTRIM (or MAXLEN on XADD)"
    ],
    "correctIndex": 3,
    "explanation": "Streams are append-only logs and must be trimmed."
  },
  {
    "question": "A consumer in a stream group crashes before XACK. What happens to its messages?",
    "options": [
      "They are deleted",
      "They stay in the pending list and can be claimed by another consumer (at-least-once delivery)",
      "They are lost",
      "They are delivered to everyone"
    ],
    "correctIndex": 1,
    "explanation": "Handlers must be idempotent."
  },
  {
    "question": "Why is MULTI/EXEC not a database-style transaction?",
    "options": [
      "It is not atomic",
      "It blocks the server forever",
      "It is only for hashes",
      "A runtime error in one command does not roll back the others"
    ],
    "correctIndex": 3,
    "explanation": "Redis queues commands and runs them without interleaving, but does not undo."
  },
  {
    "question": "What does WATCH do?",
    "options": [
      "Aborts the next EXEC if a watched key was modified by another client (optimistic locking)",
      "Publishes changes",
      "Replicates",
      "Locks keys"
    ],
    "correctIndex": 0,
    "explanation": "Retry on failure."
  },
  {
    "question": "Which lock acquire command is correct?",
    "options": [
      "GET then SET",
      "SET lock:order:42 <random token> NX PX 30000",
      "INCR lock",
      "SETNX lock 1"
    ],
    "correctIndex": 1,
    "explanation": "One atomic command with an expiry."
  },
  {
    "question": "Why is KEYS * banned in production?",
    "options": [
      "It needs auth",
      "It walks the whole keyspace in one blocking call; use SCAN",
      "It only works on replicas",
      "It deletes"
    ],
    "correctIndex": 1,
    "explanation": "SCAN may return duplicates."
  },
  {
    "question": "What does maxmemory-policy noeviction do when memory is full?",
    "options": [
      "Rejects write commands that need more memory",
      "Crashes",
      "Evicts random keys",
      "Evicts LRU keys"
    ],
    "correctIndex": 0,
    "explanation": "Choose an eviction policy for caches."
  },
  {
    "question": "A hot cache key expires and 5,000 requests hit the database at once. What is this and one fix?",
    "options": [
      "Eviction; add RAM",
      "A cluster bug; reshard",
      "A replication bug; add replicas",
      "A cache stampede; use single-flight locking or refresh early with TTL jitter"
    ],
    "correctIndex": 3,
    "explanation": "Only one request should rebuild."
  },
  {
    "question": "What is the downside of RDB-only persistence?",
    "options": [
      "No snapshots",
      "It blocks forever",
      "It cannot restore",
      "You can lose writes since the last snapshot"
    ],
    "correctIndex": 3,
    "explanation": "Add AOF for durability."
  },
  {
    "question": "What risk does appendfsync everysec carry?",
    "options": [
      "All data",
      "Up to about one second of acknowledged writes can be lost in a crash",
      "Nothing",
      "One hour"
    ],
    "correctIndex": 1,
    "explanation": "always is safest but slowest."
  },
  {
    "question": "Why can a failover lose acknowledged writes?",
    "options": [
      "Replication is asynchronous, so the promoted replica may not have the latest writes",
      "Eviction removes them",
      "TLS drops them",
      "Sentinel deletes them"
    ],
    "correctIndex": 0,
    "explanation": "WAIT reduces but does not remove the risk."
  },
  {
    "question": "Why do you need at least three Sentinels?",
    "options": [
      "A majority must authorise a failover to avoid split-brain",
      "Syntax",
      "Memory",
      "Licence"
    ],
    "correctIndex": 0,
    "explanation": "Place them on separate failure domains."
  },
  {
    "question": "How are keys assigned to slots in Redis Cluster?",
    "options": [
      "Random",
      "CRC16(key) mod 16384, using only the {hash tag} part if present",
      "By size",
      "By insertion order"
    ],
    "correctIndex": 1,
    "explanation": "Hash tags enable multi-key operations on one node."
  },
  {
    "question": "What does a MOVED reply tell the client?",
    "options": [
      "The node is read-only",
      "Auth failed",
      "The slot lives on another node; update the slot map and retry there",
      "The key was deleted"
    ],
    "correctIndex": 2,
    "explanation": "ASK is the temporary version during migration."
  },
  {
    "question": "What is a safe way to protect a Redis server?",
    "options": [
      "Share default password",
      "Open to internet",
      "Disable logging",
      "Private network, ACL users with least privilege and TLS, never exposed publicly"
    ],
    "correctIndex": 3,
    "explanation": "Redis assumes a trusted network."
  },
  {
    "question": "Why should you test restoring a backup?",
    "options": [
      "Cost",
      "Speed",
      "A backup that has never been restored is unproven",
      "Law"
    ],
    "correctIndex": 2,
    "explanation": "Practice before an incident."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A fixed-window rate limiter in Lua

Write a Lua script that atomically counts requests per key, sets a window expiry on the first hit and allows only the first N requests. Use it for two users and inspect the counter and TTL. This runs against a real Redis server with redis-cli.

**Starter code**

```bash
# Run these lines with redis-cli against an empty database.
# TODO: write a Lua script for EVAL that
#   1. INCRs KEYS[1]
#   2. sets EXPIRE KEYS[1] ARGV[1] seconds on the first hit only
#   3. returns 1 if the count is <= ARGV[2], otherwise 0
# Then call it 5 times for rl:ana (60 s window, limit 3), GET rl:ana,
# call it once for rl:bo, and finish with TTL rl:bo.
```

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:ana 60 3
EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:ana 60 3
EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:ana 60 3
EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:ana 60 3
EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:ana 60 3
GET rl:ana
EVAL &quot;local c = redis.call(&#x27;INCR&#x27;, KEYS[1]) if c == 1 then redis.call(&#x27;EXPIRE&#x27;, KEYS[1], ARGV[1]) end if c &gt; tonumber(ARGV[2]) then return 0 end return 1&quot; 1 rl:bo 60 3
TTL rl:bo</code></pre>

<p>Doing INCR, EXPIRE and the comparison inside one script makes the check-and-increment atomic, so concurrent requests cannot slip through. Calls 4 and 5 return 0 but still count, so GET shows 5. Verified by running it against Redis 8; your TTL may show 59 if a second passes.</p>
</details>

### A reliable job queue with LMOVE

Model a queue where a claimed job moves to a processing list, is removed when finished and is returned to the queue if the worker crashes. Run the sequence of commands and compare your output.

**Starter code**

```bash
# Reliable queue: a job must not be lost if a worker crashes mid-job.
# 1. Enqueue j1, j2, j3 onto the list "jobs".
# 2. A worker claims two jobs by moving them to the list "processing" (oldest first, appended at the end).
# 3. Show both lists.
# 4. The worker finishes j1: remove it from "processing" and show the list.
# 5. The worker crashes while handling j2: move it back from "processing" to the end of "jobs", show "jobs".
# 6. Finish with the length of "processing".
```

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>RPUSH jobs j1 j2 j3
LMOVE jobs processing LEFT RIGHT
LMOVE jobs processing LEFT RIGHT
LRANGE jobs 0 -1
LRANGE processing 0 -1
LREM processing 1 j1
LRANGE processing 0 -1
LMOVE processing jobs LEFT RIGHT
LRANGE jobs 0 -1
LLEN processing</code></pre>

<p>LMOVE is atomic, so a job always lives in exactly one list: waiting, processing or (after LREM) done. A crash leaves the job in processing, where a janitor can move it back. Verified by running it against Redis 8.</p>
</details>
