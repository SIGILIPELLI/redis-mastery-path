---
description: "Redis licensing from the shipped license files, Redis 8.10 vs Valkey 9.1 compared (defaults, commands, incompatible RDB files), and upgrades."
---

# 09 · Versions, Licensing, Valkey & Upgrades

"Which Redis?" used to have one answer. Since 2024 there are two major lineages with the same heritage: **Redis**
itself and **Valkey**, a fork. Managed services may run either, or something compatible. This lesson covers what the
licenses say (from the license files that ship with the software), compares Redis 8.10.2 and Valkey 9.1.2 on the same
machine, finds where they're no longer interchangeable, and ends with how to upgrade safely.

!!! note "Not legal advice"
    The summary below is a reading of the license files included with the installed software. Whether a license fits
    your use (particularly if you offer Redis as a service to others) is a question for your legal team, using the
    current license texts.

## What the license files say

The Homebrew Redis 8.10.2 package ships `LICENSE.txt`, which begins:

```text
Starting with Redis 8, Redis Open Source is moving to a tri-licensing model with all new Redis code
contributions governed by the updated Redis Software Grant and Contributor License Agreement.
After this release, contributions are subject to your choice of: (a) the Redis Source Available License v2
(RSALv2);or (b) the Server Side Public License v1 (SSPLv1); or (c) the GNU Affero General Public License v3 (AGPLv3).
Redis Open Source 7.2 and prior releases remain subject to the BSDv3 clause license as referenced
in the REDISCONTRIBUTIONS.txt file.
```

In outline, the history is:

- **Up to 7.2**: BSD 3-Clause, a permissive open-source license.
- **7.4** (2024): dual-licensed under RSALv2 and SSPLv1, both source-available licenses that restrict offering Redis as a
  competing managed service. Neither is OSI-approved open source.
- **8.0 and later** (2025): a third option, **AGPLv3**, was added, which is an OSI-approved open-source license with
  network-copyleft obligations.

The Valkey 9.1.2 package ships `COPYING`:

```text
SPDX-License-Identifier: BSD-3-Clause

BSD 3-Clause License

Copyright (c) 2024-present, Valkey contributors
```

Valkey was forked from Redis 7.2.4 in 2024 (under the Linux Foundation) when Redis moved away from BSD, and it remains
BSD-licensed. That heritage is visible in its `INFO`:

```text
redis_version:7.2.4
server_name:valkey
valkey_version:9.1.2
```

Valkey reports `redis_version:7.2.4` so that clients and tools that check the version keep working. `valkey_version`
is the real one. Redis 8.10.2 reports `redis_version:8.10.2`.

## Same commands, different defaults

The core of both is the same: data types, RESP, persistence, replication, Cluster. Every Level 1 lesson works on
either. The differences are in defaults and in what each project has added since 2024. Running the same `CONFIG GET`s
against both servers:

```text
                        Redis 8.10.2   Valkey 9.1.2
lazyfree-lazy-user-del       no            yes
lazyfree-lazy-expire         no            yes
lazyfree-lazy-eviction       no            yes
lazyfree-lazy-user-flush     no            yes
hash-max-listpack-entries    512           512
io-threads                   1             1
maxmemory-policy             noeviction    noeviction
```

The lazy-free defaults differ. On Valkey a plain `DEL` of a huge key is already non-blocking. On this Redis build it
blocked for 281 ms (Level 1 · 04) unless you use `UNLINK` or change the setting. Code that's fine on one can stall the
other.

Command availability, checked with `COMMAND INFO` on each server:

```text
DELEX        redis:delex      valkey:(none)
DIGEST       redis:digest     valkey:(none)
HSETEX       redis:hsetex     valkey:hsetex
HGETEX       redis:hgetex     valkey:hgetex
HEXPIRE      redis:hexpire    valkey:hexpire
XACKDEL      redis:xackdel    valkey:(none)
XNACK        redis:xnack      valkey:(none)
VADD         redis:VADD       valkey:(none)
FT.SEARCH    redis:FT.SEARCH  valkey:(none)
JSON.SET     redis:json.set   valkey:(none)
BF.ADD       redis:bf.add     valkey:(none)
COMMANDLOG   redis:(none)     valkey:commandlog
```

- **Hash field expiry** exists on both, and `HSETEX … EX 60 FIELDS 1 f v` worked identically.
- **`SET … IFEQ`** worked on both. Redis's `DELEX` and `DIGEST` don't exist on Valkey.
- Redis's stream additions (`XACKDEL`, `XNACK`, and `XADD … IDMP`) don't exist on Valkey. `XADD s IDMP p1 i1 * a 1` failed
  there with `ERR Invalid stream ID specified as stream command argument`, because Valkey parsed `IDMP` as an ID.
- Vector sets are built into Redis 8. The JSON, search and probabilistic commands are **modules** in both worlds.
  Redis ships its own. Valkey has separate modules of its own (for JSON, Bloom filters and search), which weren't
  installed or tested here, so their compatibility isn't assessed.
- Valkey has commands Redis doesn't, such as `COMMANDLOG` (a generalization of the slow log).

The `BITCOUNT` discrepancy from Level 2 · 02 is a reminder that implementations also differ in bugs: Valkey counted
correctly where this Redis build didn't.

## The file formats have diverged

Both servers wrote an RDB snapshot of the same small dataset. The headers differ:

```text
REDIS0015
VALKEY080
```

Valkey 9.1.2 loading Redis 8.10.2's file:

```text
# Can't handle RDB format version 15
# Fatal error loading the DB, check server logs. Exiting.
```

Redis 8.10.2 loading Valkey 9.1.2's file:

```text
# Wrong signature trying to load DB from file
# Fatal error loading the DB, check server logs. Exiting.
```

**Neither can read the other's snapshots.** The same goes for the other binary paths. A Valkey replica of a Redis 8.10
primary failed to sync:

```text
# Failed to load RDB
* Reconnecting to PRIMARY 127.0.0.1:6500 after failure
```

and `DUMP` payloads from Redis were rejected by `RESTORE` on Valkey:

```text
RESTORE profile:1 into Valkey: DUMP payload version or checksum are wrong
```

What did work was a **logical copy**: reading values with ordinary commands on one server and writing them with
ordinary commands on the other:

```text
logical copy: {b'name': b'Ana', b'plan': b'pro'}
```

So moving between the two lineages, in either direction, at these versions, means a logical migration: dual writes
from the application, or a copy tool that `SCAN`s and re-writes keys type by type (with TTLs), followed by a cut-over.
That's not a file copy or a replica promotion. Plan for it before choosing, and re-test with the versions you actually
run, since compatibility can change from release to release. (Moving from Redis ≤ 7.2 to Valkey was designed to be a
drop-in, because Valkey started from 7.2.4. That wasn't tested here.)

## Upgrading within one lineage

Within Redis (or within Valkey), the documented rule is that **newer versions load older RDB files and accept
replication from older primaries, not the other way round**. (This course only had one version of each installed, so
that direction wasn't tested here.) That makes the standard zero-downtime upgrade:

1. **Read the release notes** for every version you're skipping: changed defaults, deprecated commands, new reserved
   behaviour. The `COMMAND DOCS` history fields seen in this course (`8.4.0 Added 'IFEQ'…`) come from those changes.
2. **Upgrade replicas first.** Stop a replica, install the new version, start it, and let it resync from the old
   primary.
3. **Fail over** to an upgraded replica: gracefully with `SENTINEL failover` or `CLUSTER FAILOVER` (Level 4 · 06
   measured zero client errors for the latter).
4. **Upgrade the old primary**, now a replica, last.
5. **Don't use new features until every node is upgraded.** A new command written to the new primary would fail to
   replicate to an old replica, and an RDB written by the new version can't be loaded by the old one, which removes
   your rollback path. Keep the old version's snapshot until you're sure.

Test the whole procedure, including rollback, on a staging copy with production-like data and traffic.

## How It Actually Works

An RDB file starts with a magic string and a format version (`REDIS0015`). Each release can add new encodings (new
types, new listpack layouts, field TTL metadata, stream extensions), and the loader rejects files whose version it
doesn't know rather than misreading them. Valkey changed the magic string itself (`VALKEY080`), so a Redis loader stops
at the first bytes ("Wrong signature"). Replication full syncs and `DUMP` payloads use the same serialization with a
version and checksum trailer, which is why all three paths failed the same way, while ordinary commands (whose wire
format, RESP, both still speak) succeeded.

Because the RESP protocol and core command semantics are shared, client libraries mostly work against both. Library
features that depend on server-specific commands (Redis's client-side cache helpers, search and JSON helpers, newer
stream commands) work only where the server has them.

## Common mistakes

- **Assuming "Redis-compatible" means file-compatible.** Snapshots, replication and `DUMP` didn't cross between these
  versions.
- **Relying on defaults that differ** (lazy freeing here). Set important options explicitly in your config.
- **Using new commands in libraries or scripts** before every node and environment supports them.
- **Upgrading the primary first**, or without a tested rollback.
- **Choosing a license by headline.** Read the actual license texts for the versions you deploy, and involve legal
  counsel for service offerings.
- **Pinning to "latest"** in containers. Pin exact versions and upgrade deliberately.

## Exercise

1. Write a logical migration tool that copies every key from one server to another using `SCAN`, `TYPE`, `PTTL` and
   type-specific reads and writes (string, hash, list, set, zset, stream). Verify it by comparing `DBSIZE` and a sample of
   values, and run it from Redis 8.10 to Valkey 9.1.
2. Compare `CONFIG GET *` on both servers and list every parameter whose default differs. Which differences could change
   application behaviour?
3. Run the Level 1–3 project test suites (URL shortener, leaderboard, API guard) against Valkey. Which tests fail, and
   why?
4. Draft an upgrade runbook for a 3-shard Cluster with one replica per shard, including the exact `CLUSTER FAILOVER`
   steps, health checks between steps, and how you'd roll back after step 3.
