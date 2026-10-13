---
description: "Redis Level 4: RDB and AOF, replication and Sentinel, Cluster and resharding, ACLs and TLS, monitoring, Valkey and upgrades, and a capstone."
---

# Level 4 · Operations & Production

Level 4 is about the server your application depends on: what survives a crash, what a failover loses, how a cluster
changes shape under load, who's allowed to do what, how to find the cause of a slowdown, and how to upgrade without
breaking anything. Every claim in this level was tested by breaking something on purpose:

- `kill -9` after 5,000 writes on a snapshot-only server, which lost all 5,000;
- `appendfsync always` cutting a single client to under 30 writes per second on this laptop's disk;
- an **acknowledged** write lost when a partitioned primary died, while `WAIT` had warned it wasn't replicated;
- a Sentinel failover with a 3.2-second write outage and every acknowledged write intact;
- a cluster resharded under load with zero client errors, through 8.4's atomic slot migration;
- Python 3.14 rejecting TLS certificates that `redis-cli` accepted;
- Redis 8.10 and Valkey 9.1 unable to read each other's snapshots, replicate, or restore each other's `DUMP`s.

Servers used: **Redis 8.10.2** (Homebrew, macOS, Apple Silicon) and **Valkey 9.1.2** for the comparison lesson.
Multi-server lessons run every node on one laptop, which demonstrates behaviour and procedures, but not real
availability. Things that couldn't be tested here are stated where they come up: OS-crash and power-loss durability,
copy-on-write measurement (Linux-only instrumentation), and Valkey's own modules.

**Prerequisites:** Levels 1–3, and comfort running several processes from a terminal.

## Modules

1. **[Persistence I: RDB Snapshots & fork()](01-rdb-snapshots.md)** — `SAVE` vs `BGSAVE`, fork cost, copy-on-write
   headroom, what a crash loses, load times and `redis-check-rdb`.
2. **[Persistence II: AOF, fsync & Hybrid Persistence](02-aof-durability.md)** — fsync policies measured, process crash
   vs machine crash, multi-part AOF, rewrites, and truncated vs corrupt files.
3. **[Replication: PSYNC, the Backlog & Consistency](03-replication.md)** — diskless full sync, partial resync, backlog
   overflow, `WAIT`, and losing an acknowledged write.
4. **[High Availability with Sentinel](04-sentinel.md)** — three Sentinels, a primary killed under load, the failover
   log step by step, and Sentinel-aware clients.
5. **[Redis Cluster: Hash Slots, MOVED & ASK](05-cluster-architecture.md)** — 16,384 slots, creating a cluster,
   redirects, `CROSSSLOT`, hash tags and cluster clients.
6. **[Cluster Operations: Resharding, Scaling & Failover](06-cluster-operations.md)** — adding nodes, `ASK` during a
   manual slot move, atomic slot migration under load, killed vs graceful failover.
7. **[Security: ACLs, Protected Mode & TLS](07-security-acl-tls.md)** — protected mode, least-privilege users,
   `ACL DRYRUN` and `ACL LOG`, and mutual TLS.
8. **[Monitoring & Troubleshooting](08-monitoring-troubleshooting.md)** — the slow log, latency monitor, command stats,
   latency percentiles, hot and big keys, and what to alert on.
9. **[Versions, Licensing, Valkey & Upgrades](09-licensing-valkey-upgrades.md)** — what the license files say, Redis vs
   Valkey compared on one machine, and the rolling-upgrade procedure.
10. **[Capstone — A Secured, Persistent, Failover-Tested Cluster](10-capstone-production-cluster.md)** — a 6-node
    cluster with ACLs, authenticated replication and AOF, proven by a 13-check acceptance test.

## After Level 4

You can run Redis in production and prove that it behaves as intended. Keep the habit this course has tried to
build: before trusting a setting, a library default or a version upgrade, reproduce the failure it's supposed to
handle, on a lab you can throw away.
