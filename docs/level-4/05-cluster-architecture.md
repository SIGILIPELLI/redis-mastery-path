---
description: "Redis Cluster basics: 16,384 hash slots, creating a 6-node cluster, MOVED redirects, CROSSSLOT errors, hash tags and cluster-aware clients."
---

# 05 · Redis Cluster: Hash Slots, MOVED & ASK

Sentinel gives one dataset high availability, but the data still has to fit on one primary, and all writes go to that
one machine. **Redis Cluster** splits the keyspace across several primaries (**sharding**), each with its own
replicas, and handles failover itself without Sentinel. This lesson builds a 6-node cluster (3 primaries, 3 replicas)
and explores the rules it imposes on your keys and commands.

## Creating a cluster

Each node is an ordinary `redis-server` with cluster mode enabled:

```bash
for p in 7100 7101 7102 7103 7104 7105; do
  mkdir -p $p
  redis-server --port $p --dir ./$p --cluster-enabled yes --cluster-config-file nodes.conf \
               --cluster-node-timeout 3000 --appendonly yes --save '' --daemonize yes --logfile ./$p/log
done
redis-cli --cluster create 127.0.0.1:7100 127.0.0.1:7101 127.0.0.1:7102 \
          127.0.0.1:7103 127.0.0.1:7104 127.0.0.1:7105 --cluster-replicas 1 --cluster-yes
```

(Why 7100 and not the 7000 used in many tutorials? On this Mac, port 7000 is taken by the AirPlay receiver
(`ControlCenter`), and the first attempt failed with `[ERR] Node 127.0.0.1:7000 is not configured as a cluster node`
because the tool was talking to the wrong program. Check ports before blaming Redis.)

The tool assigns slots and pairs replicas with primaries:

```text
M: 7ef3b99d239cba2f5e4a691f3af306ba0703f9ce 127.0.0.1:7100
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 0e74daaaa2fb05870401488fe38be348fd9fb8b7 127.0.0.1:7101
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: e705a36b7eaf6608762eb76fb9d7f982826082c1 127.0.0.1:7102
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 532f511ae592f611e6c6d6ce752f4e368be6015d 127.0.0.1:7103
   replicates 0e74daaaa2fb05870401488fe38be348fd9fb8b7
...
[OK] All nodes agree about slots configuration.
[OK] All 16384 slots covered.
```

`cluster-config-file nodes.conf` isn't for you to edit. Each node writes its view of the cluster there and reloads it
on restart. `cluster-node-timeout` (3 s here) is how long a node can be unreachable before it's considered failing.

## Hash slots

The keyspace is divided into **16,384 slots**. A key's slot is `CRC16(key) mod 16384`, and each slot belongs to exactly
one primary:

```text
127.0.0.1:7100> CLUSTER KEYSLOT user:1000
(integer) 1649
127.0.0.1:7100> CLUSTER KEYSLOT user:1001
(integer) 5712
```

Slot 1649 is on 7100 and slot 5712 is on 7101. Ask the wrong node and it tells you where to go:

```text
127.0.0.1:7100> SET user:1000 ana
OK
127.0.0.1:7100> SET user:1001 ben
(error) MOVED 5712 127.0.0.1:7101
```

**`MOVED`** means "that slot lives on another node. Update your map and go there." A node never proxies the request
for you. With `-c`, `redis-cli` follows redirects:

```text
127.0.0.1:7100> SET user:1001 ben
-> Redirected to slot [5712] located at 127.0.0.1:7101
OK
127.0.0.1:7101> GET user:1000
-> Redirected to slot [1649] located at 127.0.0.1:7100
"ana"
```

## Multi-key operations and hash tags

A single command (or transaction, or script) can only touch keys in **one slot**, because one node has to execute it
atomically:

```text
127.0.0.1:7100> MSET user:1000 a user:1001 b
(error) CROSSSLOT Keys in request don't hash to the same slot
```

**Hash tags** control the slot. If a key contains `{…}`, only the part inside the first pair of braces is hashed:

```text
127.0.0.1:7100> CLUSTER KEYSLOT {user:1000}:profile
(integer) 1649
127.0.0.1:7100> CLUSTER KEYSLOT {user:1000}:sessions
(integer) 1649
127.0.0.1:7100> MSET {user:1000}:profile a {user:1000}:sessions b
OK
```

This is why earlier lessons put tags in key names (`lb:{arena}:…`, `rl:{client}:…`, `api:rl:{client}:…`): every key
a multi-key operation needs must share a tag. Design the tag around the **unit of atomicity** in your application (a
user, a cart, a tenant, a game). Don't make it too coarse: a tag like `{global}` puts everything in one slot on one
node and defeats sharding.

## The cluster-aware client

A cluster client keeps a map of slots to nodes, computes the slot of every key locally, and sends each command
straight to the right node:

```python
import time
from redis.cluster import RedisCluster, ClusterNode
rc = RedisCluster(startup_nodes=[ClusterNode("127.0.0.1", 7100)], decode_responses=True)
print("nodes known to the client:", sorted(n.name for n in rc.get_nodes()))
p = rc.pipeline()
for i in range(100_000): p.set(f"product:{i}", i)
t = time.perf_counter(); p.execute(); print(f"100,000 SETs via cluster pipeline: {time.perf_counter()-t:.2f}s")
for n in rc.get_primaries():
    print(n.name, "keys:", rc.dbsize(target_nodes=n))
print("MGET across slots:", rc.mget_nonatomic(["product:1", "product:2", "product:3"]))
try:
    rc.mget("product:1", "product:2")
except Exception as e:
    print("plain MGET across slots:", type(e).__name__, e)
try:
    with rc.pipeline(transaction=True) as t:
        t.incr("product:1"); t.incr("product:2"); t.execute()
except Exception as e:
    print("MULTI across slots:", type(e).__name__, str(e)[:90])
with rc.pipeline(transaction=True) as t:
    t.incr("{cart:9}:items"); t.expire("{cart:9}:items", 600); t.execute()
print("MULTI on one hash tag: ok, slot", rc.keyslot("{cart:9}:items"))
```

```text
nodes known to the client: ['127.0.0.1:7100', '127.0.0.1:7101', '127.0.0.1:7102', '127.0.0.1:7103', '127.0.0.1:7104', '127.0.0.1:7105']
100,000 SETs via cluster pipeline: 0.91s
127.0.0.1:7100 keys: 33309
127.0.0.1:7101 keys: 33386
127.0.0.1:7102 keys: 33309
MGET across slots: ['1', '2', '3']
plain MGET across slots: RedisClusterException MGET - all keys must map to the same key slot
MULTI across slots: CrossSlotTransactionError All keys involved in a cluster transaction must map to the same slot
MULTI on one hash tag: ok, slot 1156
```

- From one startup node the client discovered all six nodes (`CLUSTER SLOTS`/`SHARDS`).
- The cluster pipeline split 100,000 commands by node and sent them in parallel, and keys spread evenly: about a third
  per primary. (The first node holds a few more keys from the earlier `redis-cli` examples.)
- `mget_nonatomic` splits a cross-slot `MGET` into per-node requests and reassembles the results. As the name warns,
  the combined read isn't atomic.
- Cross-slot `MGET` and transactions are rejected before anything is sent. With a hash tag, a transaction works.

## Other Cluster restrictions

- **Only database 0.** `SELECT` with any other number fails.
- **Lua scripts and Functions** must declare their keys, and all of them must be in one slot (Level 2 · 09).
- **Pub/Sub**: classic channels are broadcast to every node. Use sharded Pub/Sub (Level 2 · 07).
- **Big keys** can't be split. A 10 GB sorted set lives entirely on one node and makes that node the bottleneck.
- **Keyspace-wide commands** (`SCAN`, `DBSIZE`, `FLUSHALL`, `KEYS`) act on one node. Clients must run them per node.

## How It Actually Works

Every node keeps a full map of the cluster: node IDs, addresses, roles, which primary each replica follows, and the
16,384-slot ownership table, each claim tagged with a **configuration epoch**. Nodes talk to each other on a second
port, the **cluster bus** (data port + 10,000, so 17100–17105 here), with a binary gossip protocol: every node pings a
few others each second, and each message carries the sender's slot ownership and a sample of what it knows about other
nodes. Conflicting claims to a slot are resolved by the higher config epoch, which is how a promoted replica's
ownership wins over the old primary's.

When a command arrives, the node computes the key's slot (CRC16 of the key or its hash tag, modulo 16,384). If it owns
the slot, it executes the command. If not, it replies `MOVED <slot> <ip:port>` according to its current map. Clients
cache the slot map and refresh it on `MOVED`, so in steady state almost no redirects happen. There's no proxy layer
and no central coordinator. **`ASK`** is a related, temporary redirect used while a slot is being moved between nodes.
The next lesson triggers it.

Why 16,384 slots? Each node's heartbeat carries a bitmap of the slots it owns. 16,384 bits is 2 KB, small enough to
gossip constantly, and still allows up to about 1,000 primaries with reasonable balance.

## Common mistakes

- **No hash-tag design** before moving to Cluster. Multi-key commands, transactions and scripts start failing with
  `CROSSSLOT`.
- **One tag for everything** (`{app}:…`). Everything lands on one node.
- **Hot keys and big keys.** Sharding spreads keys, not load within a key.
- **A non-cluster client** against a cluster. Every redirect fails or gets followed slowly.
- **Forgetting the bus port** in firewalls (data port + 10000). Nodes can't gossip, and the cluster never forms or
  splits.
- **Running `KEYS`/`SCAN` against one node** and believing you saw the whole keyspace.

## Exercise

1. Compute `CLUSTER KEYSLOT` for `{a}b`, `a{}b`, `{}a`, `{a}{b}` and `a{b`. Work out the hash-tag rule from the results
   (empty braces and unmatched braces behave in specific ways).
2. Store 100,000 users with keys `user:<id>:profile` and `user:<id>:cart`, then redesign the names so a script can
   update a user's profile and cart atomically. Measure how evenly the redesigned keys spread over the three primaries.
3. Write a `SCAN`-over-the-whole-cluster helper with a cluster client, iterating each primary. Why primaries only?
4. Block the cluster bus port of one primary with the proxy idea from Level 4 · 03 (or stop the node with `SIGSTOP`)
   and watch `CLUSTER NODES` on the others change its flags to `fail?` and then `fail`.
