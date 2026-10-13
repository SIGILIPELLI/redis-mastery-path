---
description: "Automatic failover with Redis Sentinel: three sentinels, a primary killed under load, the failover log step by step, and a 3.2-second outage."
---

# 04 · High Availability with Sentinel

Replication gives you a copy of the data. **Sentinel** turns that copy into automatic failover. A group of Sentinel
processes watches the primary, agrees when it's down, promotes the best replica, reconfigures the other replicas, and
tells clients where the new primary is. This lesson builds the standard layout (one primary, two replicas, three
Sentinels) on one machine, kills the primary while a client writes, and reads the whole failover from the logs.

## The setup

Primary on 6480, replicas on 6481 and 6482 (`--replicaof 127.0.0.1 6480`), all with AOF on. Each of three Sentinels
(26379, 26380, 26381) gets a config like this:

```text
port 26379
daemonize yes
logfile /…/sentinel-26379.log
dir /…/l404
sentinel monitor shop 127.0.0.1 6480 2
sentinel down-after-milliseconds shop 2000
sentinel failover-timeout shop 10000
sentinel parallel-syncs shop 1
```

```bash
redis-sentinel sentinel-26379.conf      # same as: redis-server sentinel-26379.conf --sentinel
```

- `monitor shop 127.0.0.1 6480 2` names the group (`shop`), its current primary, and the **quorum**: how many
  Sentinels must agree the primary is unreachable before it's considered down.
- `down-after-milliseconds 2000` is how long a primary can be unresponsive before a Sentinel suspects it. The default
  is 30,000. 2 seconds keeps this demo fast. In production, choose it from your tolerance for false alarms against
  outage length.
- You only list the primary. Sentinels discover the replicas from the primary and discover **each other** through a
  Pub/Sub channel on the monitored servers.

After a few seconds:

```text
127.0.0.1:26379> SENTINEL get-master-addr-by-name shop
1) "127.0.0.1"
2) "6480"
master0:name=shop,status=ok,address=127.0.0.1:6480,slaves=2,sentinels=3
127.0.0.1:26379> SENTINEL ckquorum shop
OK 3 usable Sentinels. Quorum and failover authorization can be reached
```

`ckquorum` is worth running after every change: it checks that a failover could actually be authorized.

## Killing the primary under load

A client finds the primary through the Sentinels and increments a counter every 10 ms. Three seconds in, the primary
gets `SIGKILL`:

```python
import time, os, signal
import redis
from redis.sentinel import Sentinel
sentinel = Sentinel([("127.0.0.1", 26379), ("127.0.0.1", 26380), ("127.0.0.1", 26381)], socket_timeout=0.5)
master = sentinel.master_for("shop", socket_timeout=0.5, decode_responses=True,
                             retry=redis.retry.Retry(redis.backoff.NoBackoff(), 0))
print("client sees master at", sentinel.discover_master("shop"))
pid = redis.Redis(port=6480).info("server")["process_id"]
ok = err = 0; first_err = last_err = None; t0 = time.time()
killed_at = None
while time.time() - t0 < 25:
    if killed_at is None and time.time() - t0 > 3:
        os.kill(pid, signal.SIGKILL); killed_at = time.time(); print("killed primary (pid %d) at t=3.0s" % pid)
    try:
        master.incr("orders"); ok += 1
        if first_err and last_err and not hasattr(master, "_recovered"):
            master._recovered = time.time()
            print(f"writes succeeding again at t={time.time()-t0:.1f}s, master now {sentinel.discover_master('shop')}")
    except (redis.ConnectionError, redis.TimeoutError, redis.ReadOnlyError) as e:
        err += 1; last_err = type(e).__name__
        if first_err is None: first_err = time.time(); print(f"first error at t={first_err-t0:.1f}s: {last_err}")
    time.sleep(0.01)
print(f"ok={ok} errors={err}; write outage ~{master._recovered - first_err:.1f}s")
```

```text
client sees master at ('127.0.0.1', 6480)
killed primary (pid 7309) at t=3.0s
first error at t=3.0s: ConnectionError
writes succeeding again at t=6.2s, master now ('127.0.0.1', 6482)
ok=1662 errors=225; write outage ~3.2s
```

Writes failed for **3.2 seconds**, then resumed against the new primary, 6482, with no code change and no restart.
`master_for` returns a client that asks the Sentinels for the current primary whenever it reconnects. Afterwards the
new primary held `orders = 1662`, exactly the number of acknowledged increments, so nothing was lost this time (the
client wrote slowly and replication kept up; Level 4 · 03 showed it isn't guaranteed).

## The failover, step by step

The Sentinel that led the failover logged every state change:

```text
21:43:58.964 # +sdown master shop 127.0.0.1 6480
21:43:59.022 # +odown master shop 127.0.0.1 6480 #quorum 2/2
21:43:59.022 # +new-epoch 1
21:43:59.023 # +try-failover master shop 127.0.0.1 6480
21:43:59.032 # +vote-for-leader 1035f0f2c138b0a1e7514158c5df2fd804e39ae7 1
21:43:59.124 # +elected-leader master shop 127.0.0.1 6480
21:43:59.124 # +failover-state-select-slave master shop 127.0.0.1 6480
21:43:59.181 # +selected-slave slave 127.0.0.1:6482 127.0.0.1 6482 @ shop 127.0.0.1 6480
21:43:59.182 * +failover-state-send-slaveof-noone slave 127.0.0.1:6482 …
21:43:59.266 * +failover-state-wait-promotion slave 127.0.0.1:6482 …
21:44:00.045 # +promoted-slave slave 127.0.0.1:6482 …
21:44:00.045 # +failover-state-reconf-slaves master shop 127.0.0.1 6480
21:44:00.096 * +slave-reconf-sent slave 127.0.0.1:6481 …
21:44:01.050 * +slave-reconf-done slave 127.0.0.1:6481 …
21:44:01.115 # +failover-end master shop 127.0.0.1 6480
21:44:01.119 # +switch-master shop 127.0.0.1 6480 127.0.0.1 6482
```

1. **`+sdown`** (subjectively down): this Sentinel hadn't heard from the primary for `down-after-milliseconds`, two
   seconds after the kill.
2. **`+odown`** (objectively down): it asked the others, and the quorum (2) agreed.
3. **`+new-epoch`, `+try-failover`, `+vote-for-leader`, `+elected-leader`**: only one Sentinel may run the failover. They
   hold an election for a new **configuration epoch**, and a candidate needs votes from a **majority of all
   Sentinels** (2 of 3), whatever the quorum setting.
4. **`+selected-slave`**: the leader picks a replica. Unreachable or long-disconnected replicas are excluded, then it
   prefers lower `replica-priority`, then the **highest replication offset** (most data), then the lowest run ID.
5. **`send-slaveof-noone` → `+promoted-slave`**: 6482 becomes a primary.
6. **`reconf-slaves`**: 6481 is told to replicate from 6482 (`parallel-syncs 1` means one at a time, so not all
   replicas resync at once).
7. **`+switch-master`**: the new address is published to clients and recorded in every Sentinel's config.

About 2 seconds of the 3.2-second outage was detection (`down-after-milliseconds`). The rest was the election,
promotion and the client noticing.

When the old primary came back, the Sentinels didn't let it resume as primary. They **converted it into a replica**
of the new one:

```text
role:slave
master_port:6482
master_link_status:up
```

Sentinels also **rewrite their own config files** to record what they've learned, so a restarted Sentinel remembers
the current topology:

```text
sentinel myid 18bfa71c3d54fee364fc1a827079dc5e48e19496
sentinel current-epoch 1
sentinel known-replica shop 127.0.0.1 6481
sentinel known-replica shop 127.0.0.1 6480
sentinel known-sentinel shop 127.0.0.1 26380 1035f0f2c138b0a1e7514158c5df2fd804e…
```

That means the config file must be writable, and you shouldn't manage it with a tool that overwrites it on every run.

## Deployment rules

- **At least three Sentinels, on independent machines** (or zones). Failover needs a majority, and with two Sentinels,
  losing one blocks every failover.
- **Don't put Sentinels only next to the primary.** If that machine dies, they die with it.
- **Quorum ≤ majority.** Quorum controls *detection*, the majority controls *authorization*.
- **Clients must be Sentinel-aware** (`master_for` in redis-py, and equivalents in other libraries) or go through a
  proxy that is. A hard-coded primary address defeats the whole setup.
- **Sentinel doesn't prevent split-brain writes** on an isolated old primary. Its clients may keep writing there
  until it's demoted, and those writes are lost. Combine with `min-replicas-to-write` (Level 4 · 03) to limit the
  window.
- **Test failover regularly** with `SENTINEL failover shop`, which forces a graceful failover without killing
  anything.

Everything here ran on one machine, which is fine for learning and useless for availability.

## How It Actually Works

Each Sentinel sends `PING` to every monitored server once a second and `INFO` every few seconds (which is how replicas
are discovered). It also publishes a hello message on the `__sentinel__:hello` channel of each server every 2
seconds, carrying its address, ID and current view of the configuration. That's how Sentinels find one another and
converge on the latest configuration, identified by the highest epoch.

Subjective down is a local timeout. Objective down is gathered with `SENTINEL is-master-down-by-addr` requests to the
other Sentinels. The same request, carrying a new epoch and the candidate's ID, doubles as a vote request. Each
Sentinel votes at most once per epoch, so at most one leader can win a majority for a given epoch. That's a
simplified Raft-style election, used only to pick who performs the failover. Once 6482 was promoted, the leader
broadcast the new configuration with epoch 1, and any Sentinel holding an older configuration adopted it.

Promotion itself is `REPLICAOF NO ONE` plus `CONFIG REWRITE` on the chosen replica, then `REPLICAOF <new primary>` on
the others. All are ordinary commands, which is why you need the Sentinels' commands to be allowed if you rename or
restrict commands with ACLs.

## Common mistakes

- **Two Sentinels**, or three on the same host as the primary.
- **`down-after-milliseconds` too low** on a network with hiccups. You get unnecessary failovers (each of which can lose
  data).
- **Clients connecting to a fixed address.** After failover they write to a replica (`READONLY` errors) or to nothing.
- **Read-only config files for Sentinel.** It needs to rewrite them.
- **Assuming no data loss.** Failover is asynchronous replication plus a promotion.
- **Never testing.** The first failover shouldn't be in production at 3 a.m.

## Exercise

1. Rerun the kill test with `down-after-milliseconds` at 1000, 5000 and 30000 (the default). Plot the write outage
   against the setting.
2. Kill two of the three Sentinels, then the primary. What happens, and what do the remaining Sentinel's logs say?
3. Set `replica-priority 0` on 6481 and force a failover with `SENTINEL failover shop`. Which replica is chosen, and
   can 6481 ever become primary?
4. Use the proxy from Level 4 · 03 to partition the old primary from the Sentinels but not from a client. Keep
   writing through that client during the failover and count how many of its writes are lost when the partition
   heals.
