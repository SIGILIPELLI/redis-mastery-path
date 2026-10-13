---
description: "The Redis JSON type: JSONPath reads, atomic nested updates, MERGE and NX, and when it beats a JSON string or costs twice the memory."
---

# 08 · JSON Documents

Level 1 stored objects as hashes (flat) or as JSON strings (nested, but opaque to Redis). The **JSON type**, shipped
as a module with Redis 8 (load it with the lab config from Level 1 · 01), stores a parsed document tree. You can
read and update parts of it by path, atomically, without fetching and rewriting the whole thing. It's also what the
query engine in the next lesson indexes.

## Writing and reading by path

```text
127.0.0.1:6379> JSON.SET order:1001 $ '{"id":1001,"customer":{"name":"Ana","tier":"gold"},"items":[{"sku":"pen","qty":2,"price_cents":150},{"sku":"ink","qty":1,"price_cents":400}],"status":"open","notes":null}'
OK
127.0.0.1:6379> JSON.GET order:1001 $.customer.name
"[\"Ana\"]"
127.0.0.1:6379> JSON.GET order:1001 $.items[*].sku
"[\"pen\",\"ink\"]"
127.0.0.1:6379> JSON.GET order:1001 $..price_cents
"[150,400]"
127.0.0.1:6379> JSON.GET order:1001 '$.items[?(@.qty>1)].sku'
"[\"pen\"]"
```

Paths starting with `$` are **JSONPath**: `.field`, `[*]` for all elements, `..` for recursive descent, and
`[?(...)]` filters. A JSONPath can match many places, so results always come back as a **JSON array of matches**, even
for a single value. A path that matches nothing returns an empty array, not an error:

```text
127.0.0.1:6379> JSON.GET order:1001 $.nosuch
"[]"
```

The older "legacy" path syntax (no leading `$`) returns a single value instead. It still works, but use `$` paths in
new code:

```text
127.0.0.1:6379> JSON.GET order:1001 .customer.name
"\"Ana\""
```

## Updating parts of a document

Each of these changes one part of the document in place, as a single atomic command:

```text
127.0.0.1:6379> JSON.NUMINCRBY order:1001 '$.items[?(@.sku=="pen")].qty' 3
"[5]"
127.0.0.1:6379> JSON.ARRAPPEND order:1001 $.items '{"sku":"pad","qty":1,"price_cents":250}'
1) (integer) 3
127.0.0.1:6379> JSON.SET order:1001 $.status '"paid"'
OK
127.0.0.1:6379> JSON.SET order:1001 $.paid_at '"2026-10-13T09:00:00Z"' NX
OK
127.0.0.1:6379> JSON.SET order:1001 $.paid_at '"later"' NX
(nil)
127.0.0.1:6379> JSON.DEL order:1001 '$.items[?(@.sku=="ink")]'
(integer) 1
```

- The value argument of `JSON.SET` is JSON, so a string needs its own quotes: `'"paid"'`.
- `NX`/`XX` work per path. `NX` refused to overwrite `paid_at`, which makes a natural "set once" timestamp.
- A filter in the path (`[?(@.sku=="pen")]`) lets you update an array element **by content**, not by index, which is
  safer when another client may have reordered or appended to the array.

`JSON.MERGE` applies an RFC 7386 merge patch. Objects merge recursively, and a `null` **deletes** a field:

```text
127.0.0.1:6379> JSON.MERGE order:1001 $ '{"customer":{"tier":"platinum"},"notes":null}'
OK
127.0.0.1:6379> JSON.GET order:1001
"{\"id\":1001,\"customer\":{\"name\":\"Ana\",\"tier\":\"platinum\"},\"items\":[{\"sku\":\"pen\",\"qty\":5,\"price_cents\":150},{\"sku\":\"pad\",\"qty\":1,\"price_cents\":250}],\"status\":\"paid\",\"paid_at\":\"2026-10-13T09:00:00Z\"}"
```

Two behaviours to watch for, because neither is reported as an error:

```text
127.0.0.1:6379> JSON.NUMINCRBY order:1001 $.status 1
"[null]"
127.0.0.1:6379> JSON.SET order:1001 $.a.b.c 1
(nil)
```

Incrementing a string field returned `[null]` for that match and changed nothing. Setting a path whose parents don't
exist returned `(nil)` and created nothing: `JSON.SET` only creates the last segment of a path. Check replies.

## Why not just store a JSON string?

Here are 8 threads each incrementing a nested `views` counter 250 times, once by `GET` → `json.loads` → modify → `SET`
on a string, and once with `JSON.NUMINCRBY`:

```python
def via_string():
    for _ in range(250):
        d = json.loads(r.get("doc:str")); d["views"] += 1; r.set("doc:str", json.dumps(d))
def via_json():
    for _ in range(250):
        r.json().numincrby("doc:json", "$.views", 1)
```

```text
8 threads x 250 increments -> string+json.loads: 397 | JSON.NUMINCRBY: 2000
```

The string version **lost 80% of the updates**. That's the read-modify-write race from Level 1 · 03, now on a nested
field. The JSON type got exactly 2,000.

Reading one field from a large document (20,000 events, 945 KB of JSON text):

```text
GET whole string + json.loads, read owner      7.976 ms
JSON.GET big $.owner                           0.178 ms
```

That's 45× faster, because only the matched value crosses the network and only it gets serialized.

The cost is **memory**. Here are 100,000 small order documents stored both ways:

```text
JSON string (SET)               303 bytes/doc
JSON type (JSON.SET)            607 bytes/doc
size of big doc as text: 944927 bytes; MEMORY USAGE as JSON type: 1702264
```

The parsed tree took about **twice** the memory of the text, both for small documents and for the large one. Each
node is a typed value with its own allocation and, for objects, a key table.

| | JSON string | Hash | JSON type |
|---|---|---|---|
| Nested data | yes | no | yes |
| Atomic update of one nested field | no (needs WATCH/Lua) | top-level fields only | yes |
| Read one field of a big doc | fetch everything | `HGET` | path |
| Memory | smallest | small (listpack) | about 2× the text here |
| Indexable by the query engine | no | yes | yes |

Use the JSON type for nested documents that are updated partially or queried (next lesson). Keep JSON strings for
blobs you always read and write whole, such as cached API responses.

## How It Actually Works

`JSON.SET` parses the text once and stores a tree of typed nodes (object, array, string, number, boolean, null) as
the key's value. Object members are kept in insertion order, which is why `JSON.OBJKEYS` returned the keys in the order
they were written. A JSONPath is compiled and evaluated against that tree, producing the list of matching nodes.
Reads serialize only those nodes. Writes modify them in place. Because the whole command runs on Redis's single
command thread, a `NUMINCRBY` or `ARRAPPEND` is atomic exactly like `HINCRBY`.

The tree is serialized into RDB and AOF in the module's own format, replicated by sending the command as an effect, and
counted by `MEMORY USAGE` through a module callback. That callback is how the 1.7 MB figure for the big document was
measured.

## Common mistakes

- **Forgetting that `$` paths return arrays.** Unwrap `[0]` in client code, or use a single-value path deliberately.
- **Unquoted string values** in `JSON.SET` (`paid` instead of `'"paid"'`) are rejected as invalid JSON.
- **Assuming silent replies mean success.** `[null]` and `(nil)` mean nothing happened.
- **Read-modify-write on JSON strings** under concurrency. Use the JSON type's commands, or a script.
- **Storing huge documents** (megabytes) that many clients update. Every command on the key runs on one thread. Split
  them.
- **Using the JSON type for write-once blobs.** You pay double memory for features you don't use.

## Exercise

1. Model a shopping cart as a JSON document. Write the commands for adding an item (or incrementing its quantity if
   already present: try `NUMINCRBY` with a filter, then `ARRAPPEND` if it matched nothing), removing an item by SKU, and
   applying a coupon to `$.discount` only once (`NX`). Which steps need a script to be atomic together?
2. Measure memory for 100,000 documents as a JSON string, a JSON type and a hash (flattening `customer.name` to
   `customer_name`). Include the listpack threshold from Level 2 · 01 in your reasoning.
3. Time `JSON.GET` of one field against `GET` + parse for documents of 1 KB, 10 KB, 100 KB and 1 MB. Where does the
   JSON type start to pay off?
4. Use `JSON.MERGE` with nested `null`s to delete two fields in different sub-objects in one command, and verify the
   result.
