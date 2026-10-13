---
description: "Redis bitmaps and bitfields: active-user bitmaps, BITOP, packed counters with overflow control, the sparse-offset trap, and a BITCOUNT bug."
---

# 02 · Bitmaps & Bitfields

A Redis string is a byte array, and a set of commands treats it as an array of **bits**. That gives you a bitmap:
one bit per user, per day, per feature flag. A bitmap for 10 million users fits in 1.25 MB, and bitwise AND/OR
across days runs on the server. `BITFIELD` goes further and packs many small integers (8-bit counters, 4-bit
ratings) into one string.

Bitmaps are not a separate type. `TYPE` reports `string`, and `GET` returns the raw bytes. They're a way of
*using* strings.

## Setting and reading bits

Mark users 5, 9 and 42 active on one day, and 9, 42 and 77 the next:

```text
127.0.0.1:6379> SETBIT active:2026-10-12 5 1
(integer) 0
127.0.0.1:6379> SETBIT active:2026-10-12 9 1
(integer) 0
127.0.0.1:6379> SETBIT active:2026-10-12 42 1
(integer) 0
127.0.0.1:6379> SETBIT active:2026-10-13 9 1
(integer) 0
127.0.0.1:6379> SETBIT active:2026-10-13 42 1
(integer) 0
127.0.0.1:6379> SETBIT active:2026-10-13 77 1
(integer) 0
127.0.0.1:6379> GETBIT active:2026-10-13 9
(integer) 1
127.0.0.1:6379> GETBIT active:2026-10-13 5
(integer) 0
127.0.0.1:6379> BITCOUNT active:2026-10-13
(integer) 3
127.0.0.1:6379> STRLEN active:2026-10-13
(integer) 10
```

`SETBIT` returns the bit's **previous** value. That's useful as "was this user already counted today?" in the same
round trip. The string grows to cover the highest offset set: bit 77 needs 10 bytes, and new bytes are zero.

Bit 0 is the **most significant** bit of byte 0:

```text
127.0.0.1:6379> SETBIT flags 0 1
(integer) 0
127.0.0.1:6379> SETBIT flags 7 1
(integer) 0
127.0.0.1:6379> GET flags
"\x81"
```

That's `1000 0001`. This matters as soon as another program reads the bytes, for example when you `GET` a bitmap
and decode it in Python.

## Bitwise operations across keys

`BITOP` combines bitmaps and stores the result:

```text
127.0.0.1:6379> BITOP AND active:both active:2026-10-12 active:2026-10-13
(integer) 10
127.0.0.1:6379> BITCOUNT active:both
(integer) 2
127.0.0.1:6379> BITOP OR active:either active:2026-10-12 active:2026-10-13
(integer) 10
127.0.0.1:6379> BITCOUNT active:either
(integer) 4
127.0.0.1:6379> BITOP DIFF active:churned active:2026-10-12 active:2026-10-13
(integer) 10
127.0.0.1:6379> BITCOUNT active:churned
(integer) 1
```

Two users were active both days (9, 42), four on either day, and one (user 5) was active yesterday but not today.
`AND`, `OR`, `XOR` and `NOT` have existed since Redis 2.6. **`DIFF`** (in the first key but none of the others),
`DIFF1`, `ANDOR` and `ONE` (bits set in exactly one input) are recent Redis 8 additions. They ran on this 8.10.2
server, but older servers and other forks may reject them, so check `HELP BITOP`.

Weekly actives are an `OR` over seven daily keys, and "active every day this week" is an `AND`. Each costs O(size
of the bitmaps), done in C on the server.

`BITPOS` finds the first set or clear bit:

```text
127.0.0.1:6379> BITPOS active:2026-10-13 1
(integer) 9
127.0.0.1:6379> BITPOS active:2026-10-13 0
(integer) 0
```

`BITPOS key 0` is a quick way to find the first free slot in an allocation bitmap. `BITCOUNT` and `BITPOS` take
an optional range, in bytes by default or in bits with the `BIT` keyword:

```text
127.0.0.1:6379> BITCOUNT active:2026-10-13 0 7 BIT
(integer) 0
```

## Bitmap vs set at scale

Here are 3 million active users out of 10 million, stored both as a bitmap and as a set:

```python
import redis, random, time
r = redis.Redis()
random.seed(1)
active = random.sample(range(10_000_000), 3_000_000)
p = r.pipeline(transaction=False)
for i, uid in enumerate(active):
    p.setbit("dau:bitmap", uid, 1)
    if i % 20000 == 0: p.execute()
p.execute()
for i in range(0, len(active), 10000):
    r.sadd("dau:set", *active[i:i+10000])
print("bitmap:", r.memory_usage("dau:bitmap"), "bytes;", "set:", r.memory_usage("dau:set"), "bytes")
```

```text
bitmap: 2244637 bytes; set: 124998402 bytes
```

The bitmap was **56× smaller**: 2.2 MB versus 125 MB. (Its `STRLEN` is exactly 1,250,000 bytes. `MEMORY USAGE`
is higher because the string grew in steps as `SETBIT` extended it, and the allocation kept spare capacity.) A
bitmap's size depends on the **highest ID**, not on how many bits are set, so it suits dense integer IDs. A set's
size depends on the **count**.

### The sparse-offset trap

What if IDs aren't dense? One `SETBIT` at a large offset:

```python
t = time.perf_counter(); r.setbit("sparse", 2**31, 1); el = time.perf_counter() - t
print(f"SETBIT at offset 2^31 took {el*1000:.0f} ms; key is now {r.strlen('sparse')} bytes, "
      f"used_memory={r.info('memory')['used_memory_human']}")
```

```text
SETBIT at offset 2^31 took 72 ms; key is now 268435457 bytes, used_memory=370.64M
```

**One bit cost 256 MB** and a 72 ms stall while Redis allocated and zeroed it. Using 64-bit IDs, hashed IDs or
UUIDs as offsets is a memory bomb. The maximum offset is 2^32 − 1 (a 512 MB string). Map sparse IDs to a dense
range first, or use a set, or a HyperLogLog if an approximate count will do (next lesson).

## A wrong answer from BITCOUNT on this build

Counting the 3-million-user bitmap gave a surprise:

```python
t = time.perf_counter(); n = r.bitcount("dau:bitmap")
print(f"BITCOUNT -> {n} in {(time.perf_counter()-t)*1000:.2f} ms")
```

```text
BITCOUNT -> 902848 in 0.24 ms
```

That should be 3,000,000. Counting the same bytes in Python agreed with the input, not with Redis:

```python
b = r.get("dau:bitmap")
print("python popcount:", int.from_bytes(b, "big").bit_count(), "BITCOUNT:", r.bitcount("dau:bitmap"))
```

```text
python popcount: 3000000 BITCOUNT: 902848
```

Testing strings of known content narrowed it down:

```text
0x01 x 2,097,151 bytes:    2097151      (correct)
0x01 x 2,097,152 bytes:    2097152      (correct)
0x01 x 3,000,000 bytes:     902848      expected 3000000; 3000000 mod 2^21 = 902848
0xff x 262,151 bytes:           56      expected 2097208
0xff x 1,250,000 bytes:    1611392      expected 10000000; 10000000 mod 2^21 = 1611392
```

On this server (the Homebrew bottle of **Redis 8.10.2 on Apple Silicon**), every total above 2,097,152 (2^21)
that we tried came back as the true count **modulo 2^21**, whole-key or with an explicit `0 -1` range. Smaller
totals were exact. The same tests on **Valkey 9.1.2** on the same machine returned the correct values. We
couldn't determine whether this is specific to this build or affects other platforms. If you rely on `BITCOUNT`
for large counts, test your own server with a known string, as above. Where it's wrong, a chunked count is
correct, because each chunk stays far below the limit:

```python
def bitcount_chunked(key, chunk=131072):          # 131,072 bytes = 1,048,576 bits per call
    n = r.strlen(key); total = 0
    for start in range(0, n, chunk):
        total += r.bitcount(key, start, min(start + chunk, n) - 1)
    return total
```

```text
chunked: 10000000
dau chunked: 3000000
```

This is a good reminder that course numbers, benchmark numbers and database results are all worth a sanity check
against an independent count.

## BITFIELD: packed integers

`BITFIELD` reads and writes integers of any width from 1 to 64 bits (`u` unsigned up to 63 bits, `i` signed) at
any offset. `#n` means "the n-th field of this width", so `u8 #1` is bits 8–15:

```text
127.0.0.1:6379> BITFIELD counters SET u8 #0 250 GET u8 #0
1) (integer) 0
2) (integer) 250
127.0.0.1:6379> BITFIELD counters INCRBY u8 #0 10
1) (integer) 4
```

`SET` returns the old value. `250 + 10` in an unsigned 8-bit field **wrapped** to 4, which is the default overflow
behaviour. You choose the behaviour per operation with `OVERFLOW`:

```text
127.0.0.1:6379> BITFIELD counters OVERFLOW SAT INCRBY u8 #1 300
1) (integer) 255
127.0.0.1:6379> BITFIELD counters OVERFLOW FAIL INCRBY u8 #0 250
1) (integer) 254
127.0.0.1:6379> BITFIELD counters OVERFLOW FAIL INCRBY u8 #0 10
1) (nil)
127.0.0.1:6379> BITFIELD counters GET u8 #0
1) (integer) 254
127.0.0.1:6379> BITFIELD counters OVERFLOW WRAP INCRBY u8 #0 10
1) (integer) 8
```

- `SAT` saturates at the type's limit (255).
- `FAIL` refuses the increment and returns `(nil)`, leaving the value unchanged (254 stays 254).
- `WRAP` wraps around (254 + 10 = 264 → 8).

Field offsets with `#` are multiplied by the field width, so mixing widths over the same bytes gets confusing
fast. `i16 #1` is bits 16–31, not bits 8–23:

```text
127.0.0.1:6379> BITFIELD counters INCRBY i16 #1 -5 GET i16 #1
1) (integer) -5
2) (integer) -5
127.0.0.1:6379> BITFIELD_RO counters GET u8 #0 GET u8 #1
1) (integer) 254
2) (integer) 255
127.0.0.1:6379> STRLEN counters
(integer) 4
```

`BITFIELD_RO` is the read-only variant: it can run on replicas and is allowed by read-only ACLs. All the
subcommands in one `BITFIELD` call execute atomically, in order.

A typical use is many small counters for one entity: per-hour counts (24 × u16 = 48 bytes per day), per-level
progress in a game, or 4-bit ratings for 10,000 items in 5 KB. Choose the width so overflow can't happen in
practice, and still set `OVERFLOW SAT` or `FAIL` so that if it does happen, it's visible.

## How It Actually Works

A bitmap is an ordinary SDS string. `SETBIT key offset` computes `byte = offset >> 3` and `bit = 7 - (offset & 7)`
(hence most-significant-bit first), grows the string with zero bytes if `byte` is past the end, and flips one bit.
Growth goes through the normal string-growth path, which over-allocates to make the next extension cheap. That's
why `MEMORY USAGE` exceeded `STRLEN`.

`BITCOUNT` is a population count over a byte range. Redis's implementation processes many bytes per step (word-wise
popcount with lookup tables or CPU popcount instructions, depending on the build and platform) and adds up the
results. `BITOP` loops over the inputs word by word and treats a shorter input as if it were padded with zeros.
These are tight C loops, so counting a 1.25 MB bitmap took 0.24 ms. They're still O(N) in the byte length, and N
is decided by the largest offset, not by how many bits are set.

`BITFIELD` reads the bytes that cover the field, assembles an integer of the requested width and signedness,
applies the operation with overflow checking in 64-bit arithmetic, and writes the bits back.

## Common mistakes

- **Sparse or huge offsets.** One bit at 2^31 allocated 256 MB. Map IDs to a dense range, or use another structure.
- **Assuming least-significant-bit order** when decoding bitmap bytes in client code. Bit 0 is the MSB of byte 0.
- **Trusting one counting path without a cross-check.** On this build `BITCOUNT` was wrong above 2^21. Verify with a
  known string, and count in chunks if needed.
- **Default `WRAP` overflow** on counters that must not silently wrap. Say `OVERFLOW SAT` or `FAIL`.
- **Mixing field widths with `#` offsets** over the same bytes. Keep a documented layout per key.
- **Running `BITOP` over months of daily bitmaps on the request path.** Precompute weekly and monthly rollups.

## Exercise

1. Build daily active-user bitmaps for 30 days of synthetic data (100,000 users, about 20% active per day). Compute
   DAU for each day, MAU (OR of all 30) and "active every day of the last 7" (AND). Verify each against Python
   sets.
2. Run the `BITCOUNT` test strings from this lesson on your own Redis build. Report your version, platform and
   whether the results are exact.
3. Store per-hour page views for one page and one day in a single key with `BITFIELD` (24 × `u16`, `OVERFLOW SAT`).
   Write functions `record(hour)` and `histogram()`, and compare memory with a 24-field hash.
4. Use `BITPOS key 0` to implement an ID allocator for 1,024 slots: allocate (find a free slot and set it) and free.
   Two clients can find the same free slot at the same time. Make the allocation safe using `SETBIT`'s return
   value.
