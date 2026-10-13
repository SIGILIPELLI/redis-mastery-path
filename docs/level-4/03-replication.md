---
description: "Redis replication observed: diskless full sync, partial resync from the backlog, backlog overflow, WAIT, and an acknowledged write lost on failover."
---

# 03 · Replication: PSYNC, the Backlog & Consistency

A **replica** is a Redis server that keeps a copy of another server's data (its **primary**), updated continuously.
Replicas give you a warm standby to fail over to, extra capacity for reads, and a place to take backups without
loading the primary. They're **asynchronous**: the primary acknowledges a write to the client before any replica has
it. This lesson watches every part of that on one laptop, a primary on port 6460 and replicas on 6461 and 6462, and
ends by losing an acknowledged write on purpose.

## Starting replication

With a million keys on the primary:

```bash
redis-cli -p 6461 REPLICAOF 127.0.0.1 6460
redis-cli -p 6462 REPLICAOF 127.0.0.1 6460
```

(In a config file it's `replicaof 127.0.0.1 6460`. Older material says `slaveof`, which still works.) The primary's log
shows the **full synchronization**:

```text
* Full resync requested by replica 127.0.0.1:6461 (rdb-channel)
* Delay next BGSAVE for diskless SYNC
* Full resync requested by replica 127.0.0.1:6462 (rdb-channel)
* Delay next BGSAVE for diskless SYNC
* Starting BGSAVE for SYNC with target: replicas sockets (rdb-channel)
* Synchronization with replica 127.0.0.1:6461 succeeded
* Streamed RDB transfer with replica 127.0.0.1:6461 succeeded (socket). Waiting for REPLCONF ACK from replica …
```

and the replica's:

```text
* Done loading RDB, keys loaded: 1000000, keys expired: 0.
* MASTER <-> REPLICA sync: Finished with success
```

Three things are visible here:

- **Diskless sync** (`repl-diskless-sync yes`, the default): the snapshot is streamed straight to the replicas'
  sockets instead of being written to disk first.
- **`repl-diskless-sync-delay 5`**: the primary waited 5 seconds (21:40:28 → 21:40:33) so that both replicas, which
  asked within milliseconds of each other, could share **one** fork and one stream.
- **`(rdb-channel)`**: on this Redis 8 build, the replica opened a separate connection for the snapshot, so the primary
  can stream new writes on the main replication link while the snapshot transfers. In older versions those writes had
  to be buffered on the primary for the whole transfer.

Once synced:

```text
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6461,state=online,offset=14,lag=1,io-thread=0
slave1:ip=127.0.0.1,port=6462,state=online,offset=14,lag=1,io-thread=0
master_repl_offset:28
```

The **replication offset** counts bytes of the replication stream. Each replica reports the offset it has processed,
so `master_repl_offset − replica offset` is the lag in bytes. `lag` is seconds since the replica last acknowledged.

Replicas are read-only by default:

```text
READONLY You can't write against a read only replica.
```

## Short disconnections: partial resync

The primary keeps a **replication backlog**, a circular buffer of the most recent replication stream (`repl-backlog-size`,
1 MB by default). Kill the replica connections and they reconnect:

```text
S * Trying a partial resynchronization (request 8a067f3e85013107055154c7e4b7326770bd5ffd:43).
S * Successful partial resynchronization with master.
M * Partial resynchronization request from 127.0.0.1:6461 accepted. Sending 0 bytes of backlog starting from offset 43.
```

The replica asked to continue from replication ID `8a06…` at offset 43, and since the backlog still covered offset 43,
the primary just sent what was missing (nothing, here). No snapshot, no reload.

## Long disconnections: backlog overflow

Now freeze one replica (`kill -STOP`), drop its connection, write about 4 MB of commands, and resume it:

```text
master_repl_offset now 8346759
M * Partial resynchronization request from 127.0.0.1:6462 accepted. Sending 0 bytes of backlog starting from offset 4157970.
M * Unable to partial resync with replica 127.0.0.1:6461 for lack of backlog (Replica request was: 4157970).
M * Full resync requested by replica 127.0.0.1:6461 (rdb-channel)
```

The frozen replica needed data from offset 4,157,970, but 4 MB of writes had pushed that out of the 1 MB backlog, so it
had to do a **full resync**: a new snapshot of the whole dataset. On a 50 GB instance that's a fork, gigabytes over
the network, and minutes of loading, triggered by a few seconds of network trouble. Size the backlog to cover your
longest expected disconnection at your peak write rate. For example, 5 MB/s of replication traffic and 60 seconds of
tolerance needs at least 300 MB.

## WAIT: knowing how many replicas have a write

`WAIT numreplicas timeout_ms` blocks until at least that many replicas have acknowledged everything this client has
written, or the timeout passes, and returns how many did:

```text
127.0.0.1:6460> SET order:1 paid
OK
127.0.0.1:6460> WAIT 1 1000
(integer) 1
```

## Losing an acknowledged write

To simulate a network partition, the replica connects through a small TCP proxy that can silently drop traffic:

```python
"""TCP proxy that can simulate a network partition: touch PARTITION file -> bytes are silently dropped."""
import os, socket, sys, threading
LISTEN, TARGET, FLAG = int(sys.argv[1]), int(sys.argv[2]), sys.argv[3]
def pump(src, dst):
    try:
        while True:
            data = src.recv(65536)
            if not data: break
            if not os.path.exists(FLAG): dst.sendall(data)      # partitioned: black-hole the bytes
    except OSError: pass
    finally:
        for s in (src, dst):
            try: s.close()
            except OSError: pass
srv = socket.socket(); srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("127.0.0.1", LISTEN)); srv.listen()
while True:
    c, _ = srv.accept(); u = socket.create_connection(("127.0.0.1", TARGET))
    threading.Thread(target=pump, args=(c, u), daemon=True).start()
    threading.Thread(target=pump, args=(u, c), daemon=True).start()
```

The replica runs with `--replicaof 127.0.0.1 6470` (the proxy). One write before the partition, one during it:

```text
127.0.0.1:6460> SET order:1 paid
OK
127.0.0.1:6460> WAIT 1 1000
(integer) 1
                                   ← touch PARTITION
127.0.0.1:6460> SET order:2 paid
OK
127.0.0.1:6460> WAIT 1 500
(integer) 0
(0.54s)
```

The client got `OK` for `order:2`. Then the primary dies and the replica is promoted:

```bash
redis-cli -p 6460 SHUTDOWN NOSAVE
redis-cli -p 6461 REPLICAOF NO ONE
redis-cli -p 6461 MGET order:1 order:2
```

```text
OK
paid

```

**`order:2` is gone**: the second line of the `MGET` output is empty (nil). The write was acknowledged, then lost.
That's what asynchronous replication means, and no setting removes the possibility entirely. `WAIT` *told* the
client: it returned 0, meaning "no replica has this yet". An application that must not lose a write can check
`WAIT`'s answer and treat 0 as "not safely stored" (retry elsewhere, alert, or refuse to confirm the order). `WAIT`
doesn't make Redis a consensus system. A replica that acknowledged can still fail too, and promotion doesn't pick
the most up-to-date replica unless a tool like Sentinel or Cluster does (next lessons).

A first attempt at this demo froze the replicas with `SIGSTOP` instead of a proxy, and nothing was lost: the bytes were
already in the replica's kernel socket buffer and were processed when it resumed. Real partitions drop packets, which
is why the proxy discards them.

`min-replicas-to-write N` with `min-replicas-max-lag S` makes the primary **refuse writes** when fewer than N replicas
are connected with a lag of at most S seconds. It limits how much an isolated primary can accept (and later lose),
at the cost of availability.

## Reading from replicas

Replicas apply writes slightly after the primary. On this laptop, 10,000 write-then-immediately-read-from-replica
cycles never saw a stale value:

```text
read-your-write from replica: 0 of 10000 reads returned the previous value
with WAIT 1 before reading: 0 of 2000 stale
```

That's a loopback result: the replication stream (microseconds) always beat the client's next round trip. Across a
real network, under load, or during a full resync, a replica *will* return stale data, so don't send reads that need
your own latest write to a replica. Use replicas for reads that tolerate a little lag (analytics, listings), or `WAIT`
first.

## How It Actually Works

A replica connects and sends `PSYNC <replid> <offset>`: "I have history `replid` up to `offset`". The primary has a
replication ID and an offset (plus a previous ID, `master_replid2`, which lets replicas of a promoted replica continue
partially). If the ID matches and the offset is still inside the backlog, it replies `+CONTINUE` and streams the gap.
Otherwise it replies `+FULLRESYNC <replid> <offset>`, forks, and streams an RDB snapshot. Every write command the
primary executes is appended to the backlog and sent to each replica's output buffer, in the same effect-based form as
the AOF. Replicas acknowledge their offset every second (`REPLCONF ACK`), which is the `lag` and what `WAIT`
counts.

When the replica connection's output buffer exceeds `client-output-buffer-limit replica` (256 MB hard, 64 MB for
60 s soft, Level 2 · 07 showed the defaults), the primary disconnects the replica, which then needs a backlog
resync or, failing that, a full one. A replica that can't keep up can get stuck in a loop of full resyncs. Watch for
repeated "Full resync" lines in the log.

Expiry is primary-driven: replicas don't delete expired keys themselves. They wait for the primary's `DEL`, but they
hide logically expired keys from readers in the meantime.

## Common mistakes

- **Assuming an acknowledged write is replicated.** It isn't, until `WAIT` says so.
- **A 1 MB backlog on a busy primary.** Every blip becomes a full resync.
- **Promoting a replica by hand** without checking which replica has the highest offset.
- **Read-your-writes from replicas.** It works on a laptop and fails in production.
- **Replicas without persistence behind a primary without persistence** that restarts empty. The replicas sync to the
  empty dataset and wipe themselves. If the primary has no persistence, don't let it auto-restart.
- **Ignoring `master_link_status:down`** on replicas you rely on.

## Exercise

1. Measure the time of a full resync for 1, 2 and 4 million keys, from the replica's "sync started" to "Finished with
   success" log lines. Compare disk-based (`repl-diskless-sync no`) and diskless.
2. Find the smallest `repl-backlog-size` that lets a replica survive a 10-second partition while you write 1,000
   `SET`s per second of 1 KB values.
3. Using the proxy, show that `min-replicas-to-write 1` with `min-replicas-max-lag 5` makes the partitioned primary
   reject writes after about 5 seconds. What error do clients get?
4. Put 50 ms of artificial latency in the proxy and rerun the read-your-write test. How many stale reads do you see
   now?
