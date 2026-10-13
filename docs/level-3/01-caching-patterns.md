---
description: "Caching with Redis: cache-aside at 97.7% hits, a stale-write race that both DEL and SET lose, a versioned fix, write-through and write-behind."
---

# 01 · Caching Patterns: Cache-Aside to Write-Behind

Caching is the job Redis is hired for most often, and the one most often done subtly wrong. Putting values in Redis
is easy. The hard parts are deciding **who** writes the cache, **when** it's invalidated, and what happens when
reads and writes **race**. This lesson builds a cache in front of a real database (SQLite, with 5 ms of artificial
latency so it behaves like a database across a network), measures what it buys you, and then reproduces the race
that makes the textbook pattern serve stale data for minutes.

## Cache-aside (lazy loading)

The application checks the cache first. On a miss it reads the database and stores the result with a TTL:

```python
import json, random, sqlite3, statistics, time
import redis

r = redis.Redis(decode_responses=True)
db = sqlite3.connect("shop.db", check_same_thread=False)
# ... table `products` with 10,000 rows ...

DB_LATENCY = 0.005          # pretend the real database is 5 ms away

def db_get_product(pid):
    time.sleep(DB_LATENCY)
    row = db.execute("SELECT id, name, price_cents, stock FROM products WHERE id=?", (pid,)).fetchone()
    return None if row is None else dict(zip(("id", "name", "price_cents", "stock"), row))

def get_product(pid, ttl=300):
    key = f"cache:v1:product:{pid}"
    cached = r.get(key)
    if cached is not None:
        return json.loads(cached)                    # hit (may be the cached "null")
    product = db_get_product(pid)                    # miss: go to the source of truth
    r.set(key, json.dumps(product), ex=ttl if product else 30)   # cache misses briefly too
    return product
```

Run 5,000 requests with long-tailed popularity (a few products are very popular, most rarely requested), first
straight to the database and then through the cache. The hit ratio comes from the server's `keyspace_hits` and
`keyspace_misses` counters:

```text
no cache    : mean  7.06 ms  p50  7.10 ms  p99 10.46 ms
cache-aside : mean  0.33 ms  p50  0.12 ms  p99  8.04 ms  hit ratio 97.7%
cached keys: 113
```

- With a **97.7%** hit ratio, the mean latency fell from 7.06 ms to 0.33 ms, about 20×.
- The **p99 barely moved** (10.5 → 8.0 ms). The slowest 1% are the misses, which still pay for the database *plus*
  a Redis round trip. Caches improve averages far more than tails. If your SLO is on p99, the cold path still
  matters.
- Only 113 products were ever requested, so a cache sized for a fraction of the catalogue served nearly everything.
  This is why caching works at all: real access patterns are skewed.

Details in that small function that matter in production:

- **A version segment in the key** (`v1`). When the cached format changes, deploy `v2`. Old entries are simply never
  read again and expire on their own.
- **A TTL on everything.** It's the safety net for every invalidation bug below.
- **Negative caching.** A product that doesn't exist is cached as `null` for 30 s. Otherwise a flood of requests for
  a missing ID (a bot, or a broken link) goes straight to the database every time.
- **JSON, not pickle.** Never unpickle data from a shared cache. Anyone who can write the key can then run code in
  your process.

## The race that serves stale data

The standard advice is "on update, write the database, then delete the cache key". Here are two requests,
interleaved with thread events so the interleaving is deterministic:

1. A **reader** misses the cache and reads the price from the database: 1000.
2. A **writer** updates the price to 1500 and invalidates the cache.
3. The reader, which was slow (GC pause, network hiccup), now writes what it read, 1000, into the cache.

```python
def scenario(writer_strategy):
    db_set_price(1000); r.delete(KEY)
    read_db_done, writer_done = threading.Event(), threading.Event()
    def reader():                       # a cache miss that is slow to write back
        price = db_price()              # reads OLD price 1000
        read_db_done.set(); writer_done.wait()
        r.set(KEY, json.dumps({"price_cents": price}), ex=300)
    def writer():
        read_db_done.wait()
        db_set_price(1500)              # the update
        writer_strategy()
        writer_done.set()
    ...

print("update DB, then DEL cache   -> db=%s cache=%s ttl=%s" % scenario(lambda: r.delete(KEY)))
print("update DB, then SET cache   -> db=%s cache=%s ttl=%s" %
      scenario(lambda: r.set(KEY, json.dumps({"price_cents": 1500}), ex=300)))
```

```text
update DB, then DEL cache   -> db=1500 cache=1000 ttl=300
update DB, then SET cache   -> db=1500 cache=1000 ttl=300
```

**Both strategies end with the old price cached for the full 5 minutes.** Deleting doesn't help, because the delete
happens before the stale write-back. Writing the new value doesn't help either, because the stale write-back
overwrites it. The window is small, but at thousands of requests per second it gets hit, and nothing ever corrects
it except the TTL.

### A fix: versioned write-back

Make the write-back conditional. Writers bump a version counter after changing the database. A reader notes the
version **before** reading the database, and only writes to the cache if it's still the same, checked atomically in
a script:

```python
VER = "cache:ver:product:7"
SET_IF_VERSION = r.register_script("""
if (redis.call('GET', KEYS[2]) or '0') == ARGV[1] then
  return redis.call('SET', KEYS[1], ARGV[2], 'EX', ARGV[3])
end
return false
""")

def reader():
    ver = r.get(VER) or "0"
    price = db_price()
    ...
    SET_IF_VERSION(keys=[KEY, VER], args=[ver, json.dumps({"price_cents": price}), 300])

def writer():
    db_set_price(1500); r.incr(VER); r.delete(KEY)
```

```text
versioned write-back        -> db=1500 cache=None reader-wrote=None
```

The stale write-back was refused (`reader-wrote=None`) and the cache is empty, so the next read loads 1500. Other
approaches exist: a short "do not cache" lease set by the writer, or a TTL short enough that occasional staleness
is acceptable. What matters is that you *choose* one knowingly. For many caches (product descriptions, avatars) a
few minutes of staleness is fine and the TTL is the whole strategy. For prices, permissions and balances it isn't.

## Write-through

In **write-through**, every write goes to the database and then the cache in the same code path, so the cache is
always populated for recently written data:

```python
def update_product(pid, **changes):
    db_update(pid, changes)                          # 1. source of truth first
    product = db_get_product(pid)                    # 2. read back what was committed
    r.set(f"cache:v1:product:{pid}", json.dumps(product), ex=300)   # 3. refresh the cache
```

It avoids the first-read miss after a write, but it doesn't remove the race above (a concurrent slow reader can
still overwrite step 3), and it caches data that may never be read. It's often combined with cache-aside for reads.
Never write the cache *first*: if the database write then fails, the cache holds a value that never existed.

## Write-behind (write-back)

In **write-behind**, the application writes to Redis and a background job writes to the database later in batches.
It suits high-volume, loss-tolerant updates like view counters, "last seen" times and analytics. Here 100,000 page
views are recorded with one `HINCRBY` each, and one flush moves them to SQLite:

```python
def record_view(page):                     # hot path: one Redis command, no DB write
    r.hincrby("views:pending", page, 1)

def flush():                               # runs every few seconds in a background job
    p = r.pipeline(transaction=True)       # atomically take the pending counts and reset
    p.hgetall("views:pending"); p.delete("views:pending")
    pending, _ = p.execute()
    db.executemany("INSERT INTO page_views VALUES (?, ?) ON CONFLICT(page) DO UPDATE "
                   "SET views = views + excluded.views",
                   [(k, int(v)) for k, v in pending.items()])
    db.commit()
    return len(pending)
```

```text
100,000 views recorded in 14.40s
flushed 50 rows in 6.6 ms
db total: (100000, 50)
```

The database received **50 row updates instead of 100,000**, and the totals match. (The 14.4 s is 100,000 sequential
round trips from one Python thread. Pipelining would cut it, as Level 1 · 02 showed.) The `MULTI` makes "read and
reset" atomic, so views recorded between the two commands aren't lost. The risk is the window between flushes: if
Redis loses data (no persistence, or a failover with replication lag, Level 4), those counts are gone. If the flush
crashes after `EXEC` but before `commit`, they're gone too. Only use write-behind for data where that's acceptable,
or make the hand-off durable (a stream with a consumer group instead of a hash).

## Choosing

| Pattern | Who fills the cache | Staleness risk | Good for |
|---|---|---|---|
| Cache-aside + TTL | readers, on miss | up to TTL after a write (race) | most read-heavy data |
| Cache-aside + versioned write-back | readers, conditionally | until next read | data where stale is harmful |
| Write-through | writers | race still possible | read-after-write heavy data |
| Write-behind | app writes Redis, job writes DB | DB lags; loss on Redis failure | counters, analytics |

## How It Actually Works

A cache is a second copy of data with no transaction spanning both copies. Every pattern above is a different answer
to "in which order do the two copies change, and who is allowed to write the second one?". The stale-cache race
exists because the cache write is based on a database read that happened **earlier**. By the time the write
lands, it reflects a past state. Deletion-based invalidation can't stop it, because nothing ties the write-back to
the version of the data that was read. The version counter supplies that tie: it's a logical clock for the row, and
the script compares and sets atomically on the single Redis thread, so no writer's increment can slip between the
comparison and the `SET`.

The hit ratio comes from `INFO stats`: every read command that finds a key increments `keyspace_hits`, and every one
that doesn't increments `keyspace_misses`. Taking the difference before and after a run measures that run. In
production, graph `hits / (hits + misses)` per instance. A sudden drop usually means a deploy changed key names, a
flush happened, or the cache is evicting (next lesson).

## Common mistakes

- **No TTL** "because we invalidate on write". Every invalidation bug then becomes permanent.
- **Trusting delete-on-write to prevent staleness.** The race above defeats it.
- **Writing the cache before the database.** A failed DB write leaves a phantom value.
- **Caching per-user or per-request data under shared keys.** One user sees another's data. Put the user in the key,
  or don't cache it.
- **Unbounded key cardinality** (caching every search query string forever). Use TTLs and `maxmemory` with an
  eviction policy.
- **Pickle in a shared cache.** It's a remote code execution risk.
- **Measuring averages only.** Look at p99, where the misses live.

## Exercise

1. Add a `stats()` helper that reports the hit ratio for the last minute using `INFO stats` deltas, and run the
   benchmark with TTLs of 1 s, 10 s and 300 s. Plot hit ratio against TTL.
2. Reproduce the stale-write race without thread events: run 20 reader threads and 1 writer that bumps the price every
   10 ms for 10 s, and count how many times the cache disagrees with the database once the writer stops. Then apply
   the versioned write-back and count again.
3. Change the write-behind counter to use a stream and a consumer group so a crash between `EXEC` and `commit` loses
   nothing. What does the flusher acknowledge, and when?
4. Implement negative caching for a "user by email" lookup, and explain what goes wrong if the negative entry's TTL
   is 1 hour and the user signs up 5 minutes later.
