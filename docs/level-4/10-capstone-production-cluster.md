---
description: "Capstone: a 6-node Redis Cluster with ACLs, authenticated replication and AOF, proven by a 13-check test including kill -9 and full restart."
---

# 10 · Capstone — A Secured, Persistent, Failover-Tested Cluster

The capstone puts Level 4 together into one deployment and, more importantly, into an **automated acceptance test**
that proves the deployment does what it claims. The test checks the topology, the security rules, the persistence
settings, a `kill -9` of a primary under write load (with an exact accounting of which writes survived), the dead node
coming back, and a full restart of every node.

Everything runs on one machine on ports 7200–7205. That proves the configuration and the procedures, not
availability: a real deployment puts each primary and its replica on different hosts or zones.

## The design

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

## The users

`users.acl`:

```text
user default off
user admin on >admin-Pa55-change-me ~* &* +@all
user replicator on >repl-Pa55-change-me +psync +replconf +ping
user orders-app on >app-Pa55-change-me ~{order}:* ~order:* resetchannels -@all +@read +@write +@string +@hash +@stream +@connection +wait +cluster|slots +cluster|shards +cluster|nodes -@dangerous
```

- `default off`: no anonymous access.
- `replicator` can only do what replication needs (`PSYNC`, `REPLCONF`, `PING`). Every node uses it as
  `masteruser`, so replicas authenticate to their primaries, including after a failover changes who is primary.
- `orders-app` is limited to `order:*` keys (and the `{order}` hash tag), to string, hash and stream commands, to `WAIT`,
  and to the cluster-topology commands a cluster client needs to discover nodes. `@dangerous` is removed.

The passwords are lab examples. In production they come from a secret store, and the ACL file is readable only by the
Redis user.

## The node configuration

`node.conf.tpl` (rendered per node by `up.sh`):

```text
port PORT
bind 127.0.0.1
protected-mode yes
dir DIR
daemonize yes
logfile DIR/redis.log
pidfile DIR/redis.pid
aclfile ACLFILE
masteruser replicator
masterauth repl-Pa55-change-me
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 3000
cluster-require-full-coverage no
appendonly yes
appendfsync everysec
save 300 100
maxmemory 256mb
maxmemory-policy noeviction
lazyfree-lazy-user-del yes
lazyfree-lazy-expire yes
latency-monitor-threshold 25
min-replicas-to-write 1
min-replicas-max-lag 10
```

`up.sh`:

```bash
#!/bin/bash
# start (or restart) the six nodes; data and nodes.conf persist between runs
set -e
C="$(cd "$(dirname "$0")" && pwd)"
for p in 7200 7201 7202 7203 7204 7205; do
  mkdir -p "$C/$p"
  sed -e "s#PORT#$p#; s#DIR#$C/$p#g; s#ACLFILE#$C/users.acl#" "$C/node.conf.tpl" > "$C/$p/redis.conf"
  redis-server "$C/$p/redis.conf"
done
```

Create the cluster once, authenticating as admin:

```bash
./up.sh
redis-cli --user admin --pass admin-Pa55-change-me --cluster create \
  127.0.0.1:7200 127.0.0.1:7201 127.0.0.1:7202 127.0.0.1:7203 127.0.0.1:7204 127.0.0.1:7205 \
  --cluster-replicas 1 --cluster-yes
```

```text
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

Each node's log contains a warning worth reading:

```text
# WARNING: the cluster bus port is not authenticated: 'tls-cluster' is disabled and protected mode was waived with
'cluster-bus-port-protected-mode no'. Any host able to reach the cluster bus port can speak the cluster protocol, so
make sure the port is reachable by trusted hosts only.
```

ACLs protect the data port, but the **cluster bus** (ports 17200–17205) is a separate, unauthenticated protocol
(`cluster-bus-port-protected-mode` reported `no` here). Anyone who can reach it can join or reconfigure the cluster. In
production: firewall the bus ports to the cluster's own hosts, and consider `tls-cluster yes` with certificates as in
Level 4 · 07. That wasn't enabled in this capstone.

## The acceptance test

`acceptance.py`:

```python
"""Acceptance test for the capstone cluster. Prints PASS/FAIL per check; exits non-zero on any failure."""
import os, signal, subprocess, sys, threading, time
import redis
from redis.cluster import RedisCluster, ClusterNode

C = os.path.dirname(os.path.abspath(__file__))
ADMIN = dict(username="admin", password="admin-Pa55-change-me")
APP = dict(username="orders-app", password="app-Pa55-change-me")
NODES = [ClusterNode("127.0.0.1", 7200)]
results = []

def check(name, ok, detail=""):
    results.append(ok); print(f"{'PASS' if ok else 'FAIL'}  {name}" + (f"  ({detail})" if detail else ""))

def node(port): return redis.Redis(port=port, **ADMIN, decode_responses=True)
def primaries():
    return sorted(p for p in range(7200, 7206) if node(p).info("replication")["role"] == "master")

# 1. topology
info = node(7200).cluster("info")
check("cluster state ok, all slots covered", info["cluster_state"] == "ok" and int(info["cluster_slots_ok"]) == 16384)
check("3 primaries each with a linked replica", len(primaries()) == 3 and all(
    node(p).info("replication").get("master_link_status") == "up" for p in range(7200, 7206) if p not in primaries()))

# 2. security
try:
    redis.Redis(port=7200).get("x"); check("anonymous access refused", False)
except redis.AuthenticationError as e:
    check("anonymous access refused", True, str(e)[:40])
app = RedisCluster(startup_nodes=NODES, **APP, decode_responses=True)
app.set("order:probe", "1")
for label, fn in (("app cannot touch other keys", lambda: app.set("session:1", "x")),
                  ("app cannot FLUSHALL", lambda: app.flushall()),
                  ("app cannot CONFIG SET", lambda: app.config_set("maxmemory", "1mb"))):
    try:
        fn(); check(label, False)
    except redis.exceptions.NoPermissionError as e:
        check(label, True, str(e)[:50])

# 3. persistence configuration on every node
cfg_ok = all(node(p).config_get("appendonly")["appendonly"] == "yes" and
             node(p).config_get("appendfsync")["appendfsync"] == "everysec" for p in range(7200, 7206))
check("AOF everysec on all nodes", cfg_ok)

# 4. failover under load: write orders, kill -9 one primary mid-way, count what survives
victim = primaries()[0]
pid = node(victim).info("server")["process_id"]
acked_safe, acked_unsafe, errors = set(), set(), 0
stop_at = time.time() + 12; killed = False; t0 = time.time()
i = 0
while time.time() < stop_at:
    if not killed and time.time() - t0 > 4:
        os.kill(pid, signal.SIGKILL); killed = True; t_kill = time.time()
    key = f"order:{i}"; i += 1
    try:
        app.set(key, "paid")
        n = app.execute_command("WAIT", 1, 200, target_nodes=app.get_node_from_key(key))
        (acked_safe if n >= 1 else acked_unsafe).add(key)
    except Exception:
        errors += 1
        time.sleep(0.05)
check("victim primary was killed", killed, f"port {victim}")
time.sleep(2)
missing_safe = [k for k in acked_safe if app.get(k) != "paid"]
missing_unsafe = [k for k in acked_unsafe if app.get(k) != "paid"]
check("every write acknowledged by WAIT>=1 survived the failover", not missing_safe,
      f"{len(acked_safe)} WAIT-confirmed writes, {len(missing_safe)} missing")
print(f"      info: {len(acked_unsafe)} writes acknowledged without replica confirmation, "
      f"{len(missing_unsafe)} of them lost; {errors} write attempts failed during failover")
check("cluster healthy again after failover", node(7200 if victim != 7200 else 7201).cluster("info")["cluster_state"] == "ok")

# 5. the killed node comes back as a replica
subprocess.run(["redis-server", f"{C}/{victim}/redis.conf"], check=True); time.sleep(5)
check("restarted node rejoined as a replica", node(victim).info("replication")["role"] == "slave")

# 6. full cluster restart: data persists
total_before = sum(node(p).dbsize() for p in primaries())
for p in range(7200, 7206):
    try: node(p).shutdown(nosave=False)          # SHUTDOWN: flush AOF, write RDB per 'save' rules
    except redis.ConnectionError: pass
time.sleep(1)
subprocess.run([f"{C}/up.sh"], check=True); time.sleep(6)
total_after = sum(node(p).dbsize() for p in primaries())
check("full restart keeps every key", total_after == total_before, f"{total_before} keys before, {total_after} after")
check("cluster state ok after full restart", node(7200).cluster("info")["cluster_state"] == "ok")

print(f"\n{sum(results)}/{len(results)} checks passed")
sys.exit(0 if all(results) else 1)
```

The failover section is the heart of it. The application writes `order:N` keys as fast as it can. After each write it
calls `WAIT 1 200` **on the node that owns the key** and records the write as *safe* (a replica confirmed it) or
*unsafe* (the primary acknowledged, but no replica confirmed within 200 ms). Four seconds in, one primary gets `SIGKILL`.
Afterwards every recorded key is read back. A write in the *safe* set that is missing would be a real durability
failure. A write in the *unsafe* set that is missing is exactly the loss window Level 4 · 03 demonstrated.

## The result

```text
PASS  cluster state ok, all slots covered
PASS  3 primaries each with a linked replica
PASS  anonymous access refused  (HELLO must be called with the client alr)
PASS  app cannot touch other keys  (No permissions to access a key)
PASS  app cannot FLUSHALL  (User orders-app has no permissions to run the 'flu)
PASS  app cannot CONFIG SET  (User orders-app has no permissions to run the 'con)
PASS  AOF everysec on all nodes
PASS  victim primary was killed  (port 7201)
PASS  every write acknowledged by WAIT>=1 survived the failover  (9363 WAIT-confirmed writes, 0 missing)
      info: 0 writes acknowledged without replica confirmation, 0 of them lost; 106 write attempts failed during failover
PASS  cluster healthy again after failover
PASS  restarted node rejoined as a replica
PASS  full restart keeps every key  (9529 keys before, 9529 after)
PASS  cluster state ok after full restart

13/13 checks passed
```

**13 of 13 checks passed.** In this run:

- **All 9,363 WAIT-confirmed writes survived** the `kill -9` of the primary on port 7201 and the promotion of its
  replica.
- No write was acknowledged *without* replica confirmation. On this machine replication always beat the 200 ms
  `WAIT` timeout, so the unsafe set was empty. On a loaded network some writes will land there, and the test reports
  how many of those were lost rather than hiding them.
- 106 write attempts **failed** with errors during the few seconds of failover. They failed visibly, which is
  correct: the application knows they didn't happen and can retry.
- The killed node rejoined as a replica, and after every node was shut down and restarted, all 9,529 keys came back
  from the AOF.

An earlier run of the same test killed the primary on 7200 instead and also passed 13/13 (9,466 confirmed writes, none
missing).

## How It Actually Works

The pieces interact in specific ways:

- **ACL and replication.** Replicas authenticate as `replicator` via `masteruser`/`masterauth` on *every* node, because any
  node can become a primary or a replica after a failover. The `cluster create` command itself had to authenticate
  (`--user admin`), since `default` is off.
- **`WAIT` in Cluster** applies to the node the command is sent to. That's why the test routes it with `target_nodes`
  to the key's owner. A `WAIT` sent to another node says nothing about this write.
- **`min-replicas-to-write 1`** means a primary that has lost its replica stops accepting writes after 10 seconds. That
  turns a long silent split-brain into errors. With one replica per primary, it also means losing a replica makes that
  shard read-only until it returns or is replaced, which is a deliberate trade of availability for safety.
- **The full restart** works because each node finds its own `nodes.conf` (cluster identity and slot map) and
  `appendonlydir/` (data) in its directory. `SHUTDOWN` (without `NOSAVE`) flushes the AOF and, given the `save` rule,
  writes a final RDB, so the restart loses nothing even beyond `everysec`'s window.
- **What this does not prove**: behaviour under real network partitions (Level 4 · 03's proxy), host or power failure
  (Level 4 · 02), disk-full conditions, or TLS. Each is an exercise.

## Common mistakes

- **Testing by hand once.** A deployment is only as trustworthy as the test you can re-run after every change.
- **Measuring "no errors" instead of "no lost acknowledged writes".** Errors during failover are acceptable. Silent
  loss is not.
- **Forgetting `masteruser`/`masterauth` on primaries.** After the first failover, the old primary can't sync from its
  replacement.
- **Leaving the cluster bus open** while carefully securing the data port.
- **Replicas on the same host as their primary** in production.
- **Shipping the example passwords.** They're in this lesson because it's a lab.

## Exercise

1. Add TLS for clients, replication and the cluster bus (`tls-port`, `tls-replication yes`, `tls-cluster yes`), then
   update `up.sh` and `acceptance.py` to use certificates. Which checks need changing?
2. Add a check that a network partition (using the proxy from Level 4 · 03 in front of one primary) makes that primary
   refuse writes after `min-replicas-max-lag`, and that no WAIT-confirmed write is lost when the partition heals.
3. Add an `orders` stream with a consumer group, and extend the acceptance test to verify that no acknowledged
   stream entry is lost or processed twice across a failover (Level 2 · 06).
4. Turn the test into a CI job that builds the cluster from scratch in containers, runs `acceptance.py`, and tears
   everything down. What would you need to change for Linux hosts?
