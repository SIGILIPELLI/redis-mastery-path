---
description: "How Redis encodes each type in memory: listpack thresholds where one extra member costs 12x, hash bucketing that saved 6x, and fragmentation."
---

# 01 · Encodings & Memory

Level 1 kept running into `OBJECT ENCODING`: `int` and `embstr` strings, `listpack` hashes, `intset` sets,
`skiplist` sorted sets. This lesson puts those pieces together. Every Redis type has a compact encoding for small
values and a general one for large values, and the boundary between them is configurable. Knowing where the
boundaries are, and designing around them, is the most effective memory optimization Redis offers. It can cut
memory use several-fold with no change in behaviour.

## A survey of encodings

This script creates one small and one large value of each type and reports the encoding and `MEMORY USAGE`:

```python
import redis
r = redis.Redis(decode_responses=True)
r.flushall()

def show(key):
    print(f"{key:16} {r.type(key):6} {r.object('encoding', key):11} {r.memory_usage(key):>8} bytes")

r.set("s:int", 123456); r.set("s:short", "hello"); r.set("s:long", "x" * 100)
r.rpush("l:small", *range(10)); r.rpush("l:big", *[f"item{i}" for i in range(10000)])
r.sadd("set:ints", *range(100)); r.sadd("set:small", *[f"m{i}" for i in range(100)])
r.sadd("set:big", *[f"m{i}" for i in range(1000)])
r.hset("h:small", mapping={f"f{i}": i for i in range(100)})
r.hset("h:big", mapping={f"f{i}": i for i in range(1000)})
r.zadd("z:small", {f"m{i}": i for i in range(100)}); r.zadd("z:big", {f"m{i}": i for i in range(1000)})
r.xadd("st", {"a": 1})
for k in ["s:int", "s:short", "s:long", "l:small", "l:big", "set:ints", "set:small", "set:big",
          "h:small", "h:big", "z:small", "z:big", "st"]:
    show(k)
```

```text
s:int            string int               24 bytes
s:short          string embstr            48 bytes
s:long           string raw              137 bytes
l:small          list   listpack          53 bytes
l:big            list   quicklist      99677 bytes
set:ints         set    intset           235 bytes
set:small        set    listpack         525 bytes
set:big          set    hashtable      30164 bytes
h:small          hash   listpack         723 bytes
h:big            hash   hashtable      37052 bytes
z:small          zset   listpack         723 bytes
z:big            zset   skiplist       78352 bytes
st               stream stream           289 bytes
```

Compare the 100-member and 1,000-member versions of the same type. Ten times the members cost about **57× the
memory** for sets (525 → 30,164 bytes), 51× for hashes and 108× for sorted sets. The general encodings aren't just
bigger in total. Each element costs far more.

| Type | Compact encoding | General encoding | Switch when (defaults on 8.10) |
|------|------------------|------------------|--------------------------------|
| string | `int`, `embstr` (≤44 bytes) | `raw` | non-integer, longer, or modified in place |
| list | `listpack` | `quicklist` (linked listpacks) | list exceeds one listpack node (`list-max-listpack-size`, 8 KB) |
| set | `intset`, `listpack` | `hashtable` | >512 ints, >128 members, or a member >64 bytes |
| hash | `listpack` (`listpackex` with field TTLs) | `hashtable` | >512 fields or a value >64 bytes |
| zset | `listpack` | `skiplist` + hash table | >128 members or a member >64 bytes |
| stream | radix tree of listpacks | (always) | — |

## The cliff at the threshold

Here are the costs either side of the default thresholds:

```python
for n in (128, 129):
    r.delete("z"); r.zadd("z", {f"m{i}": i for i in range(n)})
    print(f"zset with {n} members: {r.object('encoding','z'):9} {r.memory_usage('z')} bytes")
for n in (512, 513):
    r.delete("h"); r.hset("h", mapping={f"f{i}": i for i in range(n)})
    print(f"hash with {n} fields : {r.object('encoding','h'):9} {r.memory_usage('h')} bytes")
```

```text
zset with 128 members: listpack  941 bytes
zset with 129 members: skiplist  11692 bytes
hash with 512 fields : listpack  4397 bytes
hash with 513 fields : hashtable 22925 bytes
```

**One extra member made the sorted set 12× bigger.** One extra field made the hash 5× bigger. If your data
naturally sits near a threshold, say leaderboards of about 150 members or hashes of about 600 fields, moving the
threshold or reshaping the data changes your memory bill dramatically.

## Hash bucketing: storing a million small values

A classic case is a large mapping from IDs to small values, such as photo → owner, user → last-seen timestamp,
or SKU → stock. The obvious design is one string key per ID. Here are four layouts for one million pairs:

```python
import redis
r = redis.Redis()
N = 1_000_000

def run(label, write):
    r.flushall(); base = r.info("memory")["used_memory"]
    p = r.pipeline(transaction=False)
    for i in range(N):
        write(p, i)
        if i % 10000 == 9999: p.execute()
    p.execute()
    total = r.info("memory")["used_memory"] - base
    print(f"{label:44} {total/1e6:7.1f} MB  {total/N:6.1f} bytes/item  keys={r.dbsize()}")

run("1M string keys  photo:<id> -> owner id",
    lambda p, i: p.set(f"photo:{i}", i * 7 % 100000))
run("1M string keys  long name prefix",
    lambda p, i: p.set(f"application:media-service:photo-owner:{i}", i * 7 % 100000))
run("1000 hashes x 1000 fields (listpack limit 512)",
    lambda p, i: p.hset(f"photo:{i // 1000}", i % 1000, i * 7 % 100000))
run("2000 hashes x 500 fields (stays listpack)",
    lambda p, i: p.hset(f"photo:{i // 500}", i % 500, i * 7 % 100000))
```

```text
1M string keys  photo:<id> -> owner id          46.1 MB    46.1 bytes/item  keys=1000000
1M string keys  long name prefix                92.5 MB    92.5 bytes/item  keys=1000000
1000 hashes x 1000 fields (listpack limit 512)    30.4 MB    30.4 bytes/item  keys=1000
2000 hashes x 500 fields (stays listpack)        7.9 MB     7.9 bytes/item  keys=2000
```

- A verbose key prefix **doubled** memory for the same data, 46 → 92.5 MB.
- Bucketing into hashes of 1,000 fields helped somewhat (30.4 MB), but those hashes had crossed the 512-field
  threshold into `hashtable`.
- Buckets of 500 fields stayed `listpack`, and the same million pairs took **7.9 MB, about 6× less** than the
  string-per-key layout.

The access pattern changes only slightly. `GET photo:123456` becomes `HGET photo:246 456` (bucket = id // 500,
field = id % 500). What you give up: no per-item TTL (unless you use field TTLs, which switch the encoding to
`listpackex`), and every lookup is a linear scan of up to 500 entries. That's measured in microseconds, but it
isn't free, and raising the bucket size towards thousands makes it worse.

## Reading INFO memory

On a server holding 2 million 100-byte strings:

```bash
redis-cli INFO memory
```

```text
used_memory:320135840
used_memory_rss:302366720
used_memory_peak:320136192
used_memory_overhead:82410032
used_memory_dataset:237725808
used_memory_dataset_perc:74.62%
allocator_frag_ratio:1.00
mem_fragmentation_ratio:0.94
mem_allocator:libc
```

- `used_memory` — bytes Redis's allocator has handed out. This is what `maxmemory` is compared against.
- `used_memory_overhead` — keyspace tables, client buffers, replication backlog and so on. `used_memory_dataset`
  is the rest, the data itself.
- `used_memory_rss` — what the OS says the process holds in RAM. This is what your container limit or the OOM
  killer sees.
- `mem_fragmentation_ratio` — RSS ÷ used_memory.
- `mem_allocator:libc` — **this Homebrew macOS build uses the system malloc.** Official Linux builds and the
  official container image use **jemalloc**, which Redis is tuned for. Some of what follows depends on that.

Then delete every second key (1 million `UNLINK`s) and look again:

```text
used_memory:170446768
used_memory_rss:361021440
mem_fragmentation_ratio:2.12
```

Redis now holds half the data, but the process's RSS didn't shrink. It actually grew, because the pipeline's
client buffers needed memory too. Freed memory sits in the allocator's free lists, in pages that still contain
some live allocations, so it can't go back to the OS. A ratio of 2.12 means half the RAM the process occupies is
holes. `MEMORY DOCTOR` diagnoses it in plain words (its output, trimmed):

```text
 * Peak memory: In the past this instance used more than 150% the memory that is currently using. The allocator
   is normally not able to release memory after a peak, so you can expect to see a big fragmentation ratio ...
 * High total RSS: This instance has a memory fragmentation and RSS overhead greater than 1.4 ... make sure you
   are using the Jemalloc allocator and not the default libc malloc. Note: The currently used allocator is "libc".
```

What you can do about fragmentation:

- **Active defragmentation** (`activedefrag yes`) moves live allocations to compact pages while the server
  runs. It needs Redis's bundled jemalloc, and this build refused:

  ```text
  ERR CONFIG SET failed (possibly related to argument 'activedefrag') - Active defragmentation cannot be
  enabled: it requires a Redis server compiled with a modified Jemalloc like the one shipped by default with
  the Redis source distribution
  ```

- **`MEMORY PURGE`** asks the allocator to return free pages to the OS. On this libc build it returned `OK` and RSS
  fell to 134 MB with a ratio of 0.79. A ratio **below 1** on Linux usually means part of Redis has been swapped
  out, which is a serious latency problem. macOS also compresses idle memory, which can push RSS below
  `used_memory` without swapping. Read the ratio with your OS in mind.
- **Restart** a replica after a big deletion and fail over to it. Loading from a snapshot packs memory tightly.

## How It Actually Works

A **listpack** is a single allocation: a 6-byte header (total bytes and element count), then entries, then an
end marker. Each entry is an encoding byte or two that describes the type and length (small integers are stored
in as little as one byte), the data, and a back-length so the list can be walked in both directions. There are no
pointers and no per-element allocations, and a few hundred entries fit in a handful of cache lines. Lookup is a
linear scan and insertion is a `memmove`, which is why the thresholds exist. Beyond a few hundred entries, the
O(N) operations become noticeable, so Redis switches to structures with O(1) or O(log N) access and pays for them
with pointers and headers per element.

The general encodings are where the per-element overhead comes from. A `hashtable` entry is a bucket pointer,
an entry struct and separately allocated field and value strings. A sorted set at scale has a hash-table entry and
a skip-list node per member, and each skip-list node has a variable number of level pointers plus spans. Each
allocation is rounded up to the allocator's size class (jemalloc uses classes of 8, 16, 32, 48, 64, 80… bytes), so
a 9-byte string costs 16.

Conversions happen inside the command that crosses a threshold. The `ZADD` that adds member 129 rebuilds the
listpack as a skip list on the spot, which costs O(N) once. They go one way only, because converting back on
every deletion would make workloads oscillate. A key that grew past a threshold and then shrank keeps its large
encoding until it's rewritten (for example, by reloading from an RDB snapshot, which re-encodes compactly).

Fragmentation is an allocator property. Redis frees objects in an unpredictable order, so pages end up partially
used. jemalloc exposes per-allocation hints that Redis's active defrag uses: it re-allocates a value into a better
spot and fixes the one pointer to it, a little at a time within a CPU budget (`active-defrag-cycle-min/max`). A
generic malloc offers no such hints, which is why the feature needs jemalloc.

## Common mistakes

- **Sizing from one encoding and running in another.** Test memory with realistic collection sizes. 128 versus
  129 members was 941 versus 11,692 bytes.
- **Raising listpack thresholds into the thousands** to save memory. Every access becomes a long linear scan, and
  a listpack bigger than a few KB defeats the cache-locality argument. Benchmark latency as well as memory.
- **Verbose key names on hundreds of millions of keys.** The name is stored once per key and doubled memory here.
- **Watching only `used_memory`.** The OOM killer looks at RSS. Monitor `mem_fragmentation_ratio` too.
- **Benchmarking memory on a laptop build and assuming production matches.** This course's macOS build uses libc
  malloc, and Linux production almost always uses jemalloc. Check `mem_allocator`.
- **Bucketing data that needs per-item TTLs** without planning for field TTLs (7.4+) or a separate expiry
  mechanism.

## Exercise

1. Store 1,000,000 user → "last seen" timestamps three ways: string keys, hash buckets of 100, and hash buckets of
   1,000 with `hash-max-listpack-entries` raised to 1024. Record memory and the average `HGET` latency for each
   (time 10,000 random lookups).
2. Find a sorted set size near the threshold in an application you know (or invent one). Measure `MEMORY USAGE`
   for 120, 128, 129 and 200 members, then set `zset-max-listpack-entries 256` and repeat. How does `ZRANK`
   latency change?
3. Grow a hash past 512 fields, delete back down to 10, and check its encoding. Then `DEBUG RELOAD` (lab only:
   saves and reloads the RDB) and check again.
4. On a Linux machine or container with an official Redis build, repeat the "delete every second key" experiment
   with `activedefrag yes` and watch `mem_fragmentation_ratio` over a minute.
