---
description: "Redis Query Engine on 50k documents: TEXT, TAG, NUMERIC and GEO indexes, a negation trap, aggregations, and HNSW recall at default settings."
---

# 09 · Query Engine: Secondary Indexes & Vector Search

Everything so far has been looked up by key. The **query engine** (the search module shipped with Redis 8, loaded by
the lab config) adds **secondary indexes** over hashes and JSON documents: full-text search, exact-match tags,
numeric and geo ranges, aggregations, and vector similarity search. You declare an index over a key prefix, and Redis
keeps it up to date as documents are written.

This lesson indexes 50,000 synthetic product documents and 50,000 synthetic vectors. Synthetic data is fine for
learning how the engine behaves. Relevance on real text and real embeddings depends on your data and model, which
weren't part of this experiment.

## Creating an index

Products are JSON documents like `{"name": "wireless wooden lamp", "category": "garden", "price": 42.5, "rating":
4.1, "in_stock": true, "store": "-9.14,38.72"}` under keys `product:0` … `product:49999`. The index declares which
fields to index and how:

```python
r.execute_command("FT.CREATE", "idx:products", "ON", "JSON", "PREFIX", "1", "product:", "SCHEMA",
    "$.name", "AS", "name", "TEXT",
    "$.category", "AS", "category", "TAG",
    "$.price", "AS", "price", "NUMERIC", "SORTABLE",
    "$.rating", "AS", "rating", "NUMERIC",
    "$.in_stock", "AS", "in_stock", "TAG",
    "$.store", "AS", "store", "GEO")
```

```text
indexed 50000 docs in 6.98s (background indexing of existing keys)
```

Creating an index over existing data **indexes it in the background**. `FT.INFO`'s `indexing` field stays true until
it's done, and queries during that time see partial results. New writes to matching keys are indexed synchronously
as part of the write. The index used about 10.7 MB (`total_index_memory_sz_mb`) on top of the documents.

Field types matter:

- **TEXT** is tokenized, stemmed and ranked: "lamps" matches "lamp". Use it for human language.
- **TAG** is exact-match, for categories, IDs, flags and status values. Query with `{value}`.
- **NUMERIC** supports ranges. Add `SORTABLE` to sort by it cheaply.
- **GEO** supports radius queries over `"lon,lat"` strings.

## Querying

```text
127.0.0.1:6379> FT.SEARCH idx:products "wireless lamp" LIMIT 0 2 RETURN 2 name price
1) (integer) 1156
2) "product:13001"
3) 1) "name"
   2) "wireless wireless lamp"
   3) "price"
   4) "283.03"
...
127.0.0.1:6379> FT.SEARCH idx:products "@category:{garden} @price:[10 20]" SORTBY price ASC LIMIT 0 3 RETURN 2 name price
1) (integer) 217
2) "product:24624"
3) 1) "price"
   2) "10.05"
   3) "name"
   4) "ergonomic wireless drone"
...
127.0.0.1:6379> FT.SEARCH idx:products "@name:(tent|backpack) @in_stock:{true} @rating:[4.5 +inf]" LIMIT 0 0
1) (integer) 1087
127.0.0.1:6379> FT.SEARCH idx:products "@store:[-9.14 38.72 5 km] @category:{outdoor}" LIMIT 0 2 RETURN 2 name store
1) (integer) 2
2) "product:26644"
3) 1) "name"
   2) "waterproof wooden lamp"
   3) "store"
   4) "-9.11684,38.72966"
...
```

The first element of each reply is the **total** number of matches, followed by one page of documents. `LIMIT 0 0`
is a cheap way to just count. Terms separated by spaces are ANDed, `|` is OR, `@field:` scopes a term to a field, and
`[min max]` is a numeric range (with `(` for exclusive and `+inf`/`-inf` for open ends).

### A negation trap

This query is meant to be "kettles that aren't toys":

```text
127.0.0.1:6379> FT.SEARCH idx:products "-@category:{toys} kettl*" LIMIT 0 1 RETURN 1 name
1) (integer) 48988
2) "product:2545"
3) 1) "name"
   2) "compact wireless tent"
```

It matched 48,988 documents, including a tent. `FT.EXPLAIN` shows how the query was parsed:

```text
127.0.0.1:6379> FT.EXPLAIN idx:products "-@category:{toys} kettl*"
"NOT{\n  INTERSECT {\n    TAG:@category {\n      toys\n    }\n    PREFIX{kettl*}\n  }\n}\n"
```

The `-` applied to **everything after it**: NOT (toys AND kettle). That's every document except toy kettles. Putting
the positive term first, or parenthesizing the negation, gives the intended result:

```text
127.0.0.1:6379> FT.SEARCH idx:products "kettl* -@category:{toys}" LIMIT 0 1 RETURN 1 name
1) (integer) 4059
2) "product:46219"
3) 1) "name"
   2) "foldable wireless kettle"
127.0.0.1:6379> FT.SEARCH idx:products "(-@category:{toys}) kettl*" LIMIT 0 0
1) (integer) 4059
```

On this server `FT.CONFIG GET DEFAULT_DIALECT` returned `1`, and adding `DIALECT 2` to the original query still
returned 48,988. When a query returns a surprising count, run `FT.EXPLAIN` before anything else.

### Aggregations

`FT.AGGREGATE` runs a pipeline over the matches: group, reduce, sort and apply expressions:

```text
127.0.0.1:6379> FT.AGGREGATE idx:products "*" GROUPBY 1 @category REDUCE COUNT 0 AS n REDUCE AVG 1 @price AS avg_price SORTBY 2 @n DESC
1) (integer) 5
2) 1) "category"
   2) "outdoor"
   3) "n"
   4) "10071"
   5) "avg_price"
   6) "251.423819879"
...
```

### What the index buys you

The same question, "garden products priced 10–20", answered by the index and by scanning every document from
Python:

```text
FT.SEARCH: 217 matches in 12.4 ms | SCAN + JSON.GET + filter in Python: 217 matches in 6.6 s
```

The answers are the same, and the index is about 500× faster. That's the whole point of a secondary index.

## Vector search

A **vector index** answers "which stored vectors are most similar to this one?", the core of semantic search,
recommendations and retrieval-augmented generation. Here 50,000 synthetic 64-dimensional vectors (50 clusters plus
noise, L2-normalized) are stored as `FLOAT32` bytes in hashes and indexed twice: **FLAT** (exact brute force) and
**HNSW** (an approximate graph index):

```python
import time, numpy as np, redis
from redis.commands.search.query import Query
r = redis.Redis()
DIM, N = 64, 50_000
rng = np.random.default_rng(1)
centers = rng.normal(size=(50, DIM))
vecs = centers[rng.integers(0, 50, N)] + 0.3 * rng.normal(size=(N, DIM))
vecs = (vecs / np.linalg.norm(vecs, axis=1, keepdims=True)).astype(np.float32)

p = r.pipeline(transaction=False)
for i, v in enumerate(vecs):
    p.hset(f"doc:{i}", mapping={"emb": v.tobytes(), "lang": "en" if i % 4 else "pt"})
    if i % 5000 == 4999: p.execute()
p.execute()

for name, algo, extra in (("idx:vec:flat", "FLAT", []),
                          ("idx:vec:hnsw", "HNSW", ["M", "16", "EF_CONSTRUCTION", "200"])):
    r.execute_command("FT.CREATE", name, "ON", "HASH", "PREFIX", "1", "doc:", "SCHEMA",
                      "lang", "TAG", "emb", "VECTOR", algo, 6 + len(extra),
                      "TYPE", "FLOAT32", "DIM", DIM, "DISTANCE_METRIC", "COSINE", *extra)

def knn(index, q, k=10, filt="*"):
    qry = (Query(f"({filt})=>[KNN {k} @emb $v AS score]")
           .sort_by("score").return_fields("score").dialect(2).paging(0, k))
    return [d.id for d in r.ft(index).search(qry, query_params={"v": q.astype(np.float32).tobytes()}).docs]

# The rest of the script waits for indexing to finish, then times knn() for 200 noisy query vectors on each
# index, computes recall@10 against the FLAT results, cross-checks FLAT against NumPy (vecs @ q), and runs
# 50 hybrid queries with filt="@lang:{pt}".
```

```text
FLAT index built over 50000 vectors in 8.3s, vector index memory 14.9 MB
HNSW index built over 50000 vectors in 9.6s, vector index memory 27.7 MB
idx:vec:flat    1.03 ms/query  recall@10 vs exact 1.000
idx:vec:hnsw    0.33 ms/query  recall@10 vs exact 0.684
FLAT agrees with numpy brute force on 20 of 20 queries
hybrid (KNN within lang=pt): 0.45 ms/query, all results pt: True
```

FLAT was exact (it matched an independent NumPy brute-force search on 20 of 20 queries) at about 1 ms per query.
HNSW was 3× faster, but with the default query settings it found only **68% of the true top 10**. That's a quality
problem you'd never notice without measuring against an exact baseline.

The knob is **`EF_RUNTIME`**, the size of the candidate list HNSW explores per query. It can be set per query
(`KNN 10 @emb $v EF_RUNTIME 100 AS score`):

```text
HNSW EF_RUNTIME=default  0.34 ms/query  recall@10 0.684
HNSW EF_RUNTIME=50       0.35 ms/query  recall@10 0.966
HNSW EF_RUNTIME=100      0.36 ms/query  recall@10 0.991
HNSW EF_RUNTIME=200      0.41 ms/query  recall@10 0.994
HNSW EF_RUNTIME=500      0.58 ms/query  recall@10 1.000
```

At 100, recall was 0.991 for almost the same latency. On this data, the default was simply too low for a top-10 query.
Measure recall on your own vectors and set `EF_RUNTIME` (or the index default) deliberately. Also note the memory:
HNSW's graph cost almost twice FLAT's raw vectors.

**Hybrid queries** combine a filter with KNN, such as `(@lang:{pt})=>[KNN 10 …]`, and every result honoured the filter.
At 50,000 vectors FLAT is perfectly usable. HNSW earns its memory and recall tuning at millions of vectors, where
brute force grows linearly.

### Vector sets

Redis 8 also has a built-in **vector set** type (no module needed), a key that holds named vectors with an HNSW
index of its own:

```text
127.0.0.1:6379> VADD colors VALUES 3 1 0 0 red
(integer) 1
127.0.0.1:6379> VADD colors VALUES 3 0.9 0.1 0 orange
(integer) 1
127.0.0.1:6379> VADD colors VALUES 3 0 0 1 blue
(integer) 1
127.0.0.1:6379> VSIM colors VALUES 3 1 0.05 0 WITHSCORES COUNT 2
1) "red"
2) "0.9993762075901031"
3) "orange"
4) "0.9989067316055298"
127.0.0.1:6379> TYPE colors
vectorset
```

It's like a sorted set for similarity: simple, one key per collection, with no schema or document fields. Use the
query engine when you need filters and other fields alongside the vectors.

## How It Actually Works

An index is built from **keyspace notifications inside the server**. The module registers for changes to keys under
the declared prefixes, and each `JSON.SET` or `HSET` on `product:*` re-extracts the indexed fields and updates the
index structures in the same command. That's why the index is consistent with the data immediately after each write,
and also why indexing adds cost to every write.

TEXT fields go into an **inverted index**: for each (stemmed) term, a compressed list of document IDs with
frequencies and positions, used for relevance ranking (the scoring function is selectable with `SCORER`). TAG fields are inverted indexes without
tokenization or stemming. NUMERIC fields use a range tree, and GEO is stored as geohash-encoded numerics, like
`GEOADD`. A query is parsed into a tree of operators (`INTERSECT`, `UNION`, `NOT`, `PREFIX`… as `FT.EXPLAIN`
showed), and each node iterates document IDs in order, which is how intersections stay cheap.

**HNSW** builds a multi-layer proximity graph. Each vector links to about `M` neighbours per layer, and upper layers are
sparse "highways". A search descends from the top layer greedily, then explores the bottom layer while keeping a
candidate list of size `EF_RUNTIME`. A small list finishes fast but can miss true neighbours that aren't reachable
greedily. That's the 0.684 recall. A bigger list explores more and converges on the exact answer. FLAT just computes
the distance to every vector, which is exact and O(N).

In Redis Cluster, each shard indexes its own keys, and a coordinator fans queries out and merges results (in Redis's
own distributions and managed offerings; check what your deployment supports).

## Common mistakes

- **TEXT where you meant TAG.** Status values get stemmed and tokenized, and `@status:open` matches "opening".
- **Negation precedence.** `-a b` means NOT (a AND b). Check with `FT.EXPLAIN`.
- **Querying during initial indexing** and trusting the counts.
- **Indexing every field "just in case".** Each indexed field costs memory and slows every write.
- **Shipping HNSW with untuned `EF_RUNTIME`.** Recall was 0.68 here. Measure against FLAT or brute force.
- **Mismatched vector bytes**: wrong dtype (float64 sent to a FLOAT32 field), wrong dimension, or vectors that aren't
  normalized when the metric assumes it.
- **Using the query engine as the source of truth for counts in reports** without understanding its tokenization.

## Exercise

1. Add a `SORTABLE` TAG field `brand` to the product index (with `FT.ALTER`), backfill it, and write a query for
   "in-stock garden items under 50, sorted by rating, page 2 of 10 results".
2. Use `FT.AGGREGATE` to compute, per category, the number of products rated ≥ 4.5 and their median price
   (`REDUCE QUANTILE`).
3. Rebuild the HNSW index with `M 32` and `EF_CONSTRUCTION 400`. Record build time, memory and recall at
   `EF_RUNTIME` 10, 50 and 100.
4. Generate real embeddings for 2,000 sentences with any embedding model you have access to, index them, and compare
   FLAT and HNSW recall on 100 queries. Is the default `EF_RUNTIME` adequate for your data?
