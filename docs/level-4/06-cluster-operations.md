---
description: "Operating Redis Cluster: adding nodes, ASK during a manual slot move, resharding under load with atomic slot migration, and failover measured."
---

# 06 · Cluster Operations: Resharding, Scaling & Failover

A cluster earns its keep when it changes shape without downtime: a new primary to absorb growth, slots moved off a hot
node, a node replaced, a failover during maintenance. This lesson does each of those to the 6-node cluster from the
previous lesson, holding 100,000 keys, while a client keeps reading and writing. Every operation is measured from
the client's point of view: operations completed, errors, and the slowest single operation.

## Adding a primary and its replica

```bash
redis-server --port 7106 … --cluster-enabled yes …     # and 7107, configured the same way
redis-cli --cluster add-node 127.0.0.1:7106 127.0.0.1:7100
redis-cli --cluster add-node 127.0.0.1:7107 127.0.0.1:7100 --cluster-slave --cluster-master-id <id of 7106>
```

```text
>>> Send CLUSTER MEET to node 127.0.0.1:7106 to make it join the cluster.
[OK] New node added correctly.
```

A new primary joins with **zero slots**. It holds no data and serves no keys until slots are moved to it.

## Moving a slot by hand: the ASK redirect

Moving one slot by hand shows the protocol that clients must handle. Slot 1649 (`user:1000` and friends) moves from
7100 to the new node 7106. First, both sides are told what's happening:

```bash
redis-cli -p 7106 CLUSTER SETSLOT 1649 IMPORTING <id of 7100>
redis-cli -p 7100 CLUSTER SETSLOT 1649 MIGRATING <id of 7106>
redis-cli -p 7100 MIGRATE 127.0.0.1 7106 user:1000 0 5000      # move one key
```

Mid-migration, the slot's keys are split between the two nodes:

```text
127.0.0.1:7100> GET user:1000
(error) ASK 1649 127.0.0.1:7106
127.0.0.1:7100> GET {user:1000}:profile
"a"
127.0.0.1:7106> GET user:1000
(error) MOVED 1649 127.0.0.1:7100
127.0.0.1:7106> ASKING
OK
127.0.0.1:7106> GET user:1000
"ana"
```

- The source still serves keys it **has** (`{user:1000}:profile`). For a key it no longer has, it answers **`ASK`**:
  "try the other node, for this one request".
- The target **refuses** with `MOVED` back to the source, because officially the slot isn't its yet, **unless** the
  request is preceded by **`ASKING`**, which grants a one-time exception.
- `ASK` differs from `MOVED` in one crucial way: the client must **not** update its slot map, because the migration
  might not finish. Cluster clients implement exactly this dance.

The rest of the slot's keys then move with one `MIGRATE … KEYS`, and `CLUSTER SETSLOT 1649 NODE <id>` on the nodes makes
the new owner official:

```text
[OK] 100005 keys in 4 masters.
M: …7100   slots:[0-1648],[1650-5460] (5460 slots) master
M: …7106   slots:[1649] (1 slots) master
```

## Resharding under load

`load.py` increments random keys through a cluster client for a fixed time and reports errors and the slowest
operation:

```python
"""Background load: read/increment random product keys through a cluster client; report errors and latency."""
import random, sys, time, threading
import redis
from redis.cluster import RedisCluster, ClusterNode
seconds = float(sys.argv[1])
rc = RedisCluster(startup_nodes=[ClusterNode("127.0.0.1", 7100)], decode_responses=True)
ops = errs = 0; worst = 0.0; errtypes = {}
end = time.time() + seconds
while time.time() < end:
    k = f"product:{random.randrange(100_000)}"
    t = time.perf_counter()
    try:
        rc.incr(k); ops += 1
    except Exception as e:
        errs += 1; errtypes[type(e).__name__] = errtypes.get(type(e).__name__, 0) + 1
    worst = max(worst, time.perf_counter() - t)
print(f"load: {ops} ops, {errs} errors {errtypes}, worst op {worst*1000:.0f} ms")
```

With that running, move 1,000 slots from 7100 to 7106 with the standard tool:

```bash
redis-cli --cluster reshard 127.0.0.1:7100 --cluster-from <id 7100> --cluster-to <id 7106> \
          --cluster-slots 1000 --cluster-yes
```

```text
Moving 1000 slots from 127.0.0.1:7100 to 127.0.0.1:7106
Waiting for migration task d15172652a86a6f971d40187ba0859f9ee83efed to complete.
classic reshard of 1000 slots: 1.0 s
load: 158917 ops, 0 errors {}, worst op 48 ms
```

That "migration task" line was a surprise. The `redis-cli` shipped with Redis 8.10 didn't move the slots key by key
with `MIGRATE`. It started a **server-side atomic slot migration**, a feature added in Redis 8.4. The target node's log
shows how it works:

```text
* Import task d15172652a86a6f971d40187ba0859f9ee83efed started for slots: 0-999, source address: 127.0.0.1:7100
* RDB channel snapshot transfer completed for the import task.
* Starting to stream accumulated buffer for the import task (85 bytes)
```

The same mechanism can be driven directly from the **target** node:

```bash
redis-cli -p 7106 CLUSTER MIGRATION IMPORT 6000 6999
redis-cli -p 7106 CLUSTER MIGRATION STATUS ALL
```

```text
id 9e9f5759da709d795d7567123ffdddcc709fbf59 slots 6000-6999 source 0e74daaa… dest 546a8e77… operation import
state completed last_error  retries 0 … start_time 1791908390282 end_time 1791908390303 write_pause_ms 0
```

```text
* Import task 9e9f… started for slots: 6000-6999, source address: 127.0.0.1:7101
* RDB channel snapshot transfer completed for the import task.
* Successfully streamed accumulated buffer for the import task, applied offset: 86
* Import task 9e9f… is ready to takeover slots: 6000-6999
* Import task 9e9f… completed for slots: 6000-6999 (imported 6110 keys)
load: 76925 ops, 0 errors {}, worst op 67 ms
```

The 6,110 keys moved in **21 ms** of task time (`start_time` to `end_time`), the reported write pause was **0 ms**, and
the client saw no errors. Instead of the key-by-key `MIGRATE` and `ASK` dance, the target receives a snapshot of the
slots (like a replica's full sync, over an RDB channel), then the writes that happened meanwhile, then takes ownership
in one step. No client ever sees the slot half-moved.

This is very new. Older `redis-cli` versions, older servers and other forks still use the classic `MIGRATE`-based
method, during which clients get `ASK` redirects and multi-key operations on a migrating slot can fail with
`TRYAGAIN`. Know which one your versions use before planning a big reshard. `redis-cli --cluster rebalance` uses the
same machinery to even out slots across primaries, optionally weighted.

## Failover: killed vs graceful

**Killing a primary** (`SIGKILL` on 7101) under load:

```text
kill: action at t=3.0s; ops=96211 errors=163 {'ConnectionError': 163}; errors spanned 4.3s
```

The replica's log:

```text
S 21:50:32.551 * FAIL message received from 7ef3b99d… about 0e74daaa…
S 21:50:32.552 # Cluster state changed: fail
S 21:50:33.212 * Starting a failover election for epoch 8.
S 21:50:33.253 * configEpoch set to 8 after successful failover
M 21:50:33.254 * Cluster state changed: ok
```

Other nodes declared 7101 failed after `cluster-node-timeout` (3 s). Its replica 7103 waited a short, rank-based delay,
asked the primaries for votes, won epoch 8 and took over the slots. Client errors spanned 4.3 s, and all 163 were
`ConnectionError`s from requests routed to the dead node. Requests for other slots kept succeeding. Note the
`Cluster state changed: fail` line, though: with the default `cluster-require-full-coverage yes`, a node that sees
uncovered slots refuses **all** queries with `CLUSTERDOWN` until coverage returns. Here that state lasted only about
0.7 s on the replica, and the client didn't record any `CLUSTERDOWN` errors. If a shard stays down (primary and replica
both lost), it would stop the whole cluster. Setting the option to `no` keeps healthy slots available (Exercise 3).

**A graceful failover** for maintenance: run `CLUSTER FAILOVER` on the replica (7104, replica of 7102):

```text
CLUSTER FAILOVER on 7104 -> b'OK'
manual: action at t=3.0s; ops=129708 errors=0 {}
```

```text
S * All master replication stream processed, manual failover can start.
S * Starting a failover election for epoch 9.
S * configEpoch set to 9 after successful failover
```

**Zero errors.** A manual failover makes the primary pause clients, waits until the replica has processed the
**entire** replication stream, and only then switches roles. No acknowledged write can be lost, which is unlike the
kill. Always fail over gracefully before taking a primary down for patching.

The killed node, restarted, rejoined as a **replica** of the node that replaced it:

```text
role:slave
master_port:7103
```

```text
[OK] 100005 keys in 4 masters.
[OK] All nodes agree about slots configuration.
[OK] All 16384 slots covered.
```

## How It Actually Works

**Failure detection** is gossip-based. When a node hasn't answered pings for `cluster-node-timeout`, the observer flags
it `PFAIL` (possibly failing) and mentions that in its gossip. When a primary collects `PFAIL` reports about a node from
a majority of primaries, it marks the node `FAIL` and broadcasts a `FAIL` message. That's the line in the replica's
log.

**Replica promotion** is an election among primaries. The failed primary's replicas wait a delay of about
`500 ms + random + rank × 1000 ms`, where replicas with more replicated data get a lower rank and go first. Then they
request votes for a new **config epoch**. Each primary votes once per epoch, so a majority vote can elect only one
replica. The winner takes the slots and announces them with the new epoch, which overrides the old claim
everywhere. This is why a cluster needs **at least three primaries**: with two, losing one leaves no majority to elect
a replacement.

A **manual failover** adds a handshake. The replica asks its primary to stop accepting writes and send its current
offset. The replica waits until it has processed up to that offset, and then runs the election without the delay.
Hence no lost writes.

Classic **slot migration** keeps the slot in `MIGRATING`/`IMPORTING` states while `MIGRATE` moves keys one batch at a
time (each key is serialized, restored on the target with `RESTORE`, and deleted from the source, atomically per
key). That's what produces `ASK` for keys that already moved. **Atomic slot migration** (8.4+) replaces this with a
replication-like transfer of the whole slot range followed by an ownership switch, observed above as one 21 ms task.

## Common mistakes

- **Fewer than three primaries.** No majority means no automatic failover.
- **Replicas on the same host as their primary.** One machine failure takes out both.
- **Killing primaries for maintenance** instead of `CLUSTER FAILOVER` on the replica first.
- **`cluster-require-full-coverage yes`** when partial availability would be acceptable.
- **Assuming every tool version reshards the same way.** Classic migration produces `ASK`/`TRYAGAIN`, and atomic
  migration doesn't. Check before production.
- **Forgetting to rebalance** after adding a primary. It sits empty.
- **Hand-editing `nodes.conf`.** Let the cluster manage it.

## Exercise

1. Remove node 7106 from the cluster without losing data: move its slots away (`reshard` or `CLUSTER MIGRATION`), then
   `redis-cli --cluster del-node`. What happens if you try `del-node` while it still owns slots?
2. Rerun the kill-failover test with `cluster-node-timeout` at 1000 and 15000 ms and compare the error span.
3. Set `cluster-require-full-coverage no` on all nodes, kill a primary **and** its replica, and measure how many of the
   load script's operations still succeed.
4. Using the manual `SETSLOT … MIGRATING/IMPORTING` method, migrate a slot containing a 2-million-element sorted set.
   What happens to latency on the source while that single `MIGRATE` runs? How does that compare with atomic
   migration?
