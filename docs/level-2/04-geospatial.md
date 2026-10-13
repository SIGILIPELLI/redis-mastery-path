---
description: "Redis geospatial indexes: GEOADD, GEOSEARCH by radius and box, nearest-N on 1M points, the geohash sorted-set design, and its accuracy limits."
---

# 04 · Geospatial Indexes

"Find the ten nearest drivers", "which stores are within 5 km?" and "how far is the user from the restaurant?"
are questions about points on a map, and Redis answers them with a handful of `GEO*` commands. There's no separate
geo type. A geospatial index is a **sorted set** whose scores encode each point's location, which lets everything
from Level 1 · 08 work on it too.

## Adding and reading points

Coordinates go **longitude first, then latitude**. That's the opposite of how most people say them and the
opposite of Google Maps' "lat, lng" display, so it's the most common source of geo bugs:

```text
127.0.0.1:6379> GEOADD cafes -9.1393 38.7223 lisbon-baixa -9.1427 38.7139 lisbon-cais -8.6291 41.1579 porto-ribeira -9.1500 38.7369 lisbon-saldanha
(integer) 4
127.0.0.1:6379> GEOPOS cafes lisbon-baixa
1) 1) "-9.139302670955658"
   2) "38.72230020357074"
```

`GEOPOS` doesn't return exactly what went in. The location is quantized (more on that below), and the difference
is tiny. For a random point in the benchmark later in this lesson, the stored and returned positions differed by
**2.6 cm**.

Valid latitudes stop short of the poles:

```text
127.0.0.1:6379> GEOADD cafes 0 89 north-pole
(error) ERR invalid longitude,latitude pair 0.000000,89.000000
```

The limit is ±85.05112878°, the same cutoff used by Web Mercator maps.

`GEOADD` takes the same `NX`/`XX`/`CH` flags as `ZADD`. `XX` updates the position of an existing member, which is
what a moving driver's location ping does:

```text
127.0.0.1:6379> GEOADD cafes XX -9.1395 38.7224 lisbon-baixa
(integer) 0
```

## Distances

```text
127.0.0.1:6379> GEODIST cafes lisbon-baixa porto-ribeira km
"274.3731"
127.0.0.1:6379> GEODIST cafes lisbon-baixa lisbon-cais m
"979.6791"
```

Units are `m`, `km`, `mi` and `ft`.

## Searching

`GEOSEARCH` finds members inside a circle (`BYRADIUS`) or a rectangle (`BYBOX`), centred either on a coordinate
(`FROMLONLAT`) or on an existing member (`FROMMEMBER`):

```text
127.0.0.1:6379> GEOSEARCH cafes FROMLONLAT -9.1400 38.7200 BYRADIUS 2 km ASC WITHDIST
1) 1) "lisbon-baixa"
   2) "0.2629"
2) 1) "lisbon-cais"
   2) "0.7177"
127.0.0.1:6379> GEOSEARCH cafes FROMMEMBER lisbon-baixa BYRADIUS 5 km ASC COUNT 2 WITHDIST WITHCOORD
1) 1) "lisbon-baixa"
   2) "0.0000"
   3) 1) "-9.139302670955658"
      2) "38.72230020357074"
2) 1) "lisbon-cais"
   2) "0.9797"
   3) 1) "-9.142698347568512"
      2) "38.71390013764869"
127.0.0.1:6379> GEOSEARCH cafes FROMLONLAT -9.14 38.72 BYBOX 4 4 km ASC
1) "lisbon-baixa"
2) "lisbon-cais"
3) "lisbon-saldanha"
```

- `ASC` sorts nearest first. Without it, the order isn't defined.
- `COUNT n` limits the results, which is how you get "nearest N".
- `WITHDIST` and `WITHCOORD` add the distance (in the query's unit) and the position.
- `FROMMEMBER` includes the member itself at distance 0. Skip it in your code if you want its "neighbours".
- `BYBOX 4 4 km` is a 4 km × 4 km box centred on the point. Saldanha, about 2 km north, falls inside the box but
  outside a 2 km circle.

`GEOSEARCHSTORE` saves the results into another key. With `STOREDIST`, the scores of the new sorted set are the
distances, so the result is a ready-sorted "by distance" list:

```text
127.0.0.1:6379> GEOSEARCHSTORE near:me cafes FROMLONLAT -9.14 38.72 BYRADIUS 3 km STOREDIST
(integer) 3
127.0.0.1:6379> ZRANGE near:me 0 -1 WITHSCORES
1) "lisbon-baixa"
2) "0.2629022054797619"
3) "lisbon-cais"
4) "0.7177408007183564"
5) "lisbon-saldanha"
6) "2.070324830252379"
```

(`GEORADIUS` and `GEORADIUSBYMEMBER` are the older commands that `GEOSEARCH` replaces. You'll still see them in
existing code.)

## It's a sorted set

```text
127.0.0.1:6379> TYPE cafes
zset
127.0.0.1:6379> ZSCORE cafes lisbon-baixa
"1966377783327353"
127.0.0.1:6379> ZRANGE cafes 0 -1
1) "lisbon-cais"
2) "lisbon-saldanha"
3) "lisbon-baixa"
4) "porto-ribeira"
127.0.0.1:6379> ZREM cafes lisbon-cais
(integer) 1
```

There's no `GEODEL`. You remove a point with `ZREM`, count with `ZCARD`, and expire the whole index with `EXPIRE`.
The score is a 52-bit **geohash** of the location stored as an integer. Points that are near each other usually
have nearby scores, which is what makes searching fast. `GEOHASH` returns the standard base32 text form, usable
with other geohash tools:

```text
127.0.0.1:6379> GEOHASH cafes lisbon-baixa porto-ribeira
1) "eycs210vwz0"
2) "ez3f5ss6pk0"
```

## A million drivers

Load 1,000,000 random points across mainland Portugal and time a few searches around central Lisbon:

```python
import redis, random, time
r = redis.Redis(decode_responses=True)
random.seed(11)
pts = [(random.uniform(-9.5, -6.2), random.uniform(37.0, 42.1)) for _ in range(1_000_000)]
p = r.pipeline(transaction=False)
for i in range(0, len(pts), 5000):
    args = []
    for j, (lon, lat) in enumerate(pts[i:i+5000], start=i):
        args += [lon, lat, f"driver:{j}"]
    p.geoadd("drivers", args); p.execute()
print("members", r.zcard("drivers"), "encoding", r.object("encoding", "drivers"),
      "memory", r.memory_usage("drivers"))

def t(label, fn, n=200):
    s = time.perf_counter()
    for _ in range(n): res = fn()
    print(f"{label:52} {(time.perf_counter()-s)/n*1e6:8.1f} us -> {len(res)} results")

q = dict(longitude=-9.14, latitude=38.72, unit="km")
t("GEOSEARCH 1 km, ASC COUNT 10", lambda: r.geosearch("drivers", radius=1, sort="ASC", count=10, **q))
t("GEOSEARCH 10 km, ASC COUNT 10", lambda: r.geosearch("drivers", radius=10, sort="ASC", count=10, **q))
t("GEOSEARCH 10 km, ASC COUNT 10 ANY", lambda: r.geosearch("drivers", radius=10, sort="ASC", count=10, any=True, **q))
t("GEOSEARCH 10 km, all results", lambda: r.geosearch("drivers", radius=10, **q), n=20)
```

```text
members 1000000 encoding skiplist memory 77722402
GEOSEARCH 1 km, ASC COUNT 10                            217.5 us -> 10 results
GEOSEARCH 10 km, ASC COUNT 10                           840.8 us -> 10 results
GEOSEARCH 10 km, ASC COUNT 10 ANY                       190.6 us -> 10 results
GEOSEARCH 10 km, all results                            939.0 us -> 1799 results
```

The interesting line is the second one. "Nearest 10 within 10 km" took **841 µs** even though it returned only 10
results. To guarantee the ten *nearest*, Redis has to collect and sort every candidate in the radius (1,799 of
them) before it can keep ten. Adding **`ANY`** lets Redis stop as soon as it has found ten matches, which took the
time back down to 191 µs. The catch is that those ten are *some* matches within 10 km, sorted among themselves,
and not necessarily the ten nearest.

The practical pattern for "nearest N" is therefore to **start with a small radius and widen it** only if you don't
get enough results (1 km, then 3 km, then 10 km). In dense areas the small query succeeds immediately, and in
sparse areas the wide one has few candidates anyway.

## How accurate are the distances?

Redis models the Earth as a **sphere**. Compared with geodesic distances on the WGS84 ellipsoid (computed with the
`geographiclib` package) for the same stored positions:

```text
lisbon->porto: Redis   274.373 km  WGS84 ellipsoid   273.925 km  diff +0.164%
lisbon->tokyo: Redis 11145.939 km  WGS84 ellipsoid 11166.662 km  diff -0.186%
```

An error of about 0.2% (450 m over 274 km, 21 km over 11,000 km) is irrelevant for "drivers near me" and wrong for
surveying, aviation or anything legally binding. The search radius itself carries the same approximation. A point
right at the edge of a radius may land on either side of it.

## How It Actually Works

`GEOADD` interleaves the bits of the quantized longitude and latitude into one 52-bit integer, a **geohash**, and
stores it as the member's sorted-set score. Twenty-six bits per axis divide the world into cells of roughly half a
metre, which is where the centimetre-level `GEOPOS` difference comes from. A 52-bit integer also fits exactly in a
double's 53-bit mantissa, so the score loses nothing.

Interleaving bits creates a Z-order curve. Points that share a long geohash prefix are close together, and a
prefix corresponds to a rectangular cell. To search a radius, Redis picks a geohash precision whose cells are
about the size of the radius, computes the cell containing the centre plus its **8 neighbours**, and turns each
cell into a score range. Each range is an O(log N) seek in the skip list followed by a scan of the members in that
cell. Each candidate's exact distance is then computed with the haversine formula on a sphere of radius 6,372,797.6
m, and candidates outside the circle or box are dropped. That's why the cost depends on how many points fall in
the 9 cells, not on the total size of the index, and why a 10 km search in a dense city has to examine thousands
of candidates.

Since the index is just a sorted set, it inherits everything else: replication, persistence, `MULTI`, Lua, and the
memory profile of a large sorted set (about 78 bytes per point here).

## Common mistakes

- **Latitude first.** It's `GEOADD key longitude latitude member`, every time.
- **Assuming "nearest N" is cheap with a big radius.** Without `ANY`, Redis sorts every candidate in the radius.
  Widen progressively.
- **Treating distances as survey-grade.** The spherical model was 0.16–0.19% off here.
- **Forgetting stale positions.** A driver who went offline stays in the index. Store a last-seen time and remove
  members, or keep one index per time bucket and expire old buckets.
- **One global index in Cluster** for a hot use case. A sorted set lives on a single node. Shard by region (one key
  per city) if one node can't keep up.
- **Expecting polygons.** Redis core supports circles and boxes. For polygons, or for combining geo with other
  filters, the query engine (Level 3 · 09) has geo fields.

## Exercise

1. Build a "drivers near me" service. Drivers report positions every few seconds (`GEOADD XX`, with `NX` for new
   drivers), and riders ask for the nearest 5 within an expanding radius (1, 3, 10 km). Track `last-seen` in a
   separate sorted set and remove drivers silent for more than 60 s.
2. Measure how the 10 km `ASC COUNT 10` query time changes as you go from 100,000 to 3,000,000 points in the
   same area, with and without `ANY`.
3. Verify the ±85.05112878° latitude limit experimentally. What's the largest latitude `GEOADD` accepts, and what
   does `GEOPOS` return for it?
4. Find two points whose `GEOSEARCH BYRADIUS` result disagrees with an exact ellipsoidal calculation (one inside by
   one measure and outside by the other). How far from the radius boundary must a point be for this to happen?
