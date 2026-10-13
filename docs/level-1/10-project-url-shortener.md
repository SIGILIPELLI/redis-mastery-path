---
description: "Level 1 project: a Redis-only URL shortener with atomic code claims, exact click counts, daily uniques, recent clicks and a top-links board."
---

# 10 · Project — URL Shortener with Click Analytics

This project pulls Level 1 together. You'll build a URL shortener whose only storage is Redis, with click
analytics on top, and test it under concurrency. It uses every type from this level, and each choice of type is
deliberate:

| Need | Structure | Why this one |
|------|-----------|--------------|
| Next numeric id | string + `INCR` | atomic, gap-tolerant counter |
| A link's data | hash `short:link:<code>` | one object, per-field updates, `HINCRBY` clicks |
| Claim a code or alias once | `HSETNX` on the link hash | first writer wins, atomically |
| Same URL → same code | string `short:by-target:<sha256>` + `SET NX` | dedupe without scanning |
| Last 20 clicks | list + `LPUSH`/`LTRIM` | capped, newest first |
| Unique visitors per day | set per code per day, with TTL | `SADD` dedupes, `SCARD` counts |
| Top links today | sorted set per day | `ZINCRBY` + `ZRANGE … REV` |

The finished module is about 130 lines. It has 13 tests, and the first version had a real bug that one of the
tests found. That story is below.

## Setup

```bash
python3 -m venv .venv && .venv/bin/pip install redis hiredis pytest
redis-server --port 6379 --save '' --daemonize yes   # or your lab server
```

The tests use **database 15** and flush it before and after each test. Don't point them at a database you care
about.

## The code

`shortener.py`:

```python
"""A URL shortener backed only by Redis (Level 1 project)."""
import hashlib
import string
import time
from datetime import datetime, timezone

import redis

ALPHABET = string.digits + string.ascii_letters          # base62
RECENT_CLICKS = 20
DAY_TTL = 8 * 24 * 3600                                    # keep per-day keys for a week


def base62(n: int) -> str:
    if n == 0:
        return ALPHABET[0]
    out = []
    while n:
        n, rem = divmod(n, 62)
        out.append(ALPHABET[rem])
    return "".join(reversed(out))


class AliasTaken(Exception):
    pass


class Shortener:
    def __init__(self, r: redis.Redis, prefix: str = "short"):
        self.r = r
        self.p = prefix

    # key helpers -----------------------------------------------------------
    def _link(self, code):      return f"{self.p}:link:{code}"
    def _by_target(self, url):  return f"{self.p}:by-target:{hashlib.sha256(url.encode()).hexdigest()[:32]}"
    def _recent(self, code):    return f"{self.p}:recent:{code}"
    def _visitors(self, code, day): return f"{self.p}:visitors:{code}:{day}"
    def _top(self, day):        return f"{self.p}:top:{day}"

    @staticmethod
    def _today(now=None):
        return datetime.fromtimestamp(now or time.time(), tz=timezone.utc).strftime("%Y-%m-%d")

    # writes ----------------------------------------------------------------
    def shorten(self, url: str, owner: str = "anon", ttl: int | None = None) -> str:
        """Return a code for url. The same url (without ttl) always maps to the same code."""
        if not url.startswith(("http://", "https://")):
            raise ValueError("only http(s) URLs")
        if ttl is None:
            existing = self.r.get(self._by_target(url))
            if existing:
                return existing
        while True:                                   # skip codes already claimed as aliases
            code = base62(self.r.incr(f"{self.p}:next-id") + 100_000)   # skip the very short codes
            if self._create(code, url, owner, ttl):
                break
        if ttl is None:
            # first writer wins; a concurrent writer of the same URL reuses its code
            if not self.r.set(self._by_target(url), code, nx=True):
                winner = self.r.get(self._by_target(url))
                self.r.unlink(self._link(code))
                return winner
        return code

    def shorten_alias(self, alias: str, url: str, owner: str = "anon") -> str:
        if not alias.isalnum() or not 3 <= len(alias) <= 32:
            raise ValueError("alias must be 3-32 letters/digits")
        # HSETNX creates the hash only if the field is absent: an atomic "claim"
        if not self.r.hsetnx(self._link(alias), "target", url):
            raise AliasTaken(alias)
        self.r.hset(self._link(alias), mapping={"owner": owner, "created": int(time.time()), "clicks": 0})
        return alias

    def _create(self, code, url, owner, ttl) -> bool:
        # claim the code first: HSETNX fails if an alias (or anything) already owns it
        if not self.r.hsetnx(self._link(code), "target", url):
            return False
        pipe = self.r.pipeline(transaction=True)
        pipe.hset(self._link(code), mapping={"owner": owner, "created": int(time.time()), "clicks": 0})
        if ttl:
            pipe.expire(self._link(code), ttl)
        pipe.execute()
        return True

    # reads -----------------------------------------------------------------
    def resolve(self, code: str, visitor: str = "", referrer: str = "-", now: float | None = None) -> str | None:
        target = self.r.hget(self._link(code), "target")
        if target is None:
            return None
        day = self._today(now)
        pipe = self.r.pipeline(transaction=False)
        pipe.hincrby(self._link(code), "clicks", 1)
        pipe.lpush(self._recent(code), f"{int(now or time.time())}|{referrer}")
        pipe.ltrim(self._recent(code), 0, RECENT_CLICKS - 1)
        if visitor:
            pipe.sadd(self._visitors(code, day), visitor)
            pipe.expire(self._visitors(code, day), DAY_TTL)
        pipe.zincrby(self._top(day), 1, code)
        pipe.expire(self._top(day), DAY_TTL)
        pipe.execute()
        return target

    def stats(self, code: str, now: float | None = None) -> dict | None:
        day = self._today(now)
        pipe = self.r.pipeline(transaction=False)
        pipe.hgetall(self._link(code))
        pipe.ttl(self._link(code))
        pipe.scard(self._visitors(code, day))
        pipe.lrange(self._recent(code), 0, 4)
        link, ttl, uniques, recent = pipe.execute()
        if not link:
            return None
        return {"target": link["target"], "clicks": int(link["clicks"]), "ttl": ttl,
                "unique_today": uniques, "recent": recent}

    def top_today(self, n: int = 5, now: float | None = None):
        return self.r.zrange(self._top(self._today(now)), 0, n - 1, desc=True, withscores=True)

    def delete(self, code: str, owner: str) -> bool:
        link = self.r.hgetall(self._link(code))
        if not link or link.get("owner") != owner:
            return False
        pipe = self.r.pipeline(transaction=True)
        pipe.unlink(self._link(code), self._recent(code))
        if self.r.get(self._by_target(link["target"])) == code:
            pipe.unlink(self._by_target(link["target"]))
        pipe.execute()
        return True
```

### Design notes

**Codes from a counter.** `INCR short:next-id` hands out unique integers with no coordination between app
servers. Adding 100,000 before base62-encoding keeps codes at three characters or more (the first is `q0V`), so
nobody can guess the first few links by trying `1`, `2`, `3`. Sequential codes are still *enumerable*, though.
If links can be private, use random codes (`secrets.token_urlsafe`) claimed with the same `HSETNX` and retried on
collision.

**Claiming with `HSETNX`.** A custom alias like `launch` must belong to whoever asked first. `HSETNX link:launch
target <url>` creates the field only if it's absent, and creating the field creates the hash. So the
check-and-create is a single atomic command, and twenty concurrent claims produce exactly one winner (see
`test_alias_race`).

**Same URL, same code.** The reverse index key is a SHA-256 of the URL, which keeps keys short and avoids
special characters. Two requests shortening the same new URL at the same moment can both miss the index and both
create a code. The `SET … NX` decides which one wins, and the loser deletes its own code and returns the
winner's. The race is resolved rather than prevented, and `test_concurrent_same_url_single_code` checks that 20
threads get one code.

**Click recording is one round trip.** `resolve` sends six or seven commands in one non-transactional pipeline:
increment clicks, push and trim the recent list, add the visitor to today's set, bump today's top-links board,
and refresh TTLs. None of them depends on another's result, so there's no need for `MULTI`. Each command is atomic
by itself, which is all the click counter needs. (`test_clicks_are_exact_under_concurrency`: 8 threads × 250
clicks = exactly 2,000.)

**Bounded memory.** Recent clicks are trimmed to 20. Per-day sets and boards expire after 8 days. Only the link
hashes live forever, unless created with a TTL.

## The tests

`test_shortener.py`:

```python
import threading, time
import pytest, redis
from shortener import Shortener, AliasTaken, base62

@pytest.fixture
def s():
    r = redis.Redis(db=15, decode_responses=True)
    r.flushdb()
    yield Shortener(r)
    r.flushdb()

def test_base62():
    assert [base62(n) for n in (0, 61, 62, 100_001)] == ["0", "Z", "10", "q0V"]

def test_shorten_and_resolve(s):
    code = s.shorten("https://redis.io/docs/")
    assert s.resolve(code) == "https://redis.io/docs/"
    assert s.resolve("nope") is None

def test_same_url_same_code(s):
    assert s.shorten("https://example.com/a") == s.shorten("https://example.com/a")
    assert s.shorten("https://example.com/a") != s.shorten("https://example.com/b")

def test_rejects_non_http(s):
    with pytest.raises(ValueError):
        s.shorten("javascript:alert(1)")

def test_alias_claimed_once(s):
    assert s.shorten_alias("launch", "https://example.com/launch") == "launch"
    with pytest.raises(AliasTaken):
        s.shorten_alias("launch", "https://evil.example/")
    assert s.resolve("launch") == "https://example.com/launch"

def test_alias_race(s):
    wins, errors = [], []
    def claim(i):
        try:
            s.shorten_alias("promo", f"https://example.com/{i}"); wins.append(i)
        except AliasTaken:
            errors.append(i)
    ts = [threading.Thread(target=claim, args=(i,)) for i in range(20)]
    [t.start() for t in ts]; [t.join() for t in ts]
    assert len(wins) == 1 and len(errors) == 19
    assert s.resolve("promo") == f"https://example.com/{wins[0]}"

def test_concurrent_same_url_single_code(s):
    codes = []
    def go():
        codes.append(s.shorten("https://example.com/hot"))
    ts = [threading.Thread(target=go) for _ in range(20)]
    [t.start() for t in ts]; [t.join() for t in ts]
    assert len(set(codes)) == 1
    assert s.resolve(codes[0]) == "https://example.com/hot"

def test_clicks_are_exact_under_concurrency(s):
    code = s.shorten("https://example.com/popular")
    def hit():
        for _ in range(250): s.resolve(code, visitor="v")
    ts = [threading.Thread(target=hit) for _ in range(8)]
    [t.start() for t in ts]; [t.join() for t in ts]
    assert s.stats(code)["clicks"] == 2000

def test_stats_uniques_and_recent_capped(s):
    code = s.shorten("https://example.com/x")
    for i in range(30):
        s.resolve(code, visitor=f"user{i % 7}", referrer=f"ref{i}")
    st = s.stats(code)
    assert st["clicks"] == 30 and st["unique_today"] == 7
    assert len(st["recent"]) == 5 and st["recent"][0].endswith("|ref29")
    assert s.r.llen(s._recent(code)) == 20

def test_top_today(s):
    a = s.shorten("https://example.com/1"); b = s.shorten("https://example.com/2")
    for _ in range(3): s.resolve(a)
    for _ in range(5): s.resolve(b)
    assert s.top_today(2) == [(b, 5.0), (a, 3.0)]

def test_expiring_link(s):
    code = s.shorten("https://example.com/temp", ttl=1)
    assert s.resolve(code) == "https://example.com/temp"
    time.sleep(1.2)
    assert s.resolve(code) is None

def test_delete_only_by_owner(s):
    code = s.shorten("https://example.com/mine", owner="ana")
    assert s.delete(code, owner="ben") is False
    assert s.delete(code, owner="ana") is True
    assert s.resolve(code) is None
    assert s.shorten("https://example.com/mine") != code

def test_generated_code_never_overwrites_alias(s):
    s.shorten_alias("q0V", "https://example.com/alias-owner")      # the next generated code
    code = s.shorten("https://example.com/generated")
    assert s.resolve("q0V") == "https://example.com/alias-owner"
    assert code != "q0V" and s.resolve(code) == "https://example.com/generated"
```

```bash
.venv/bin/pytest -v
```

```text
test_shortener.py::test_base62 PASSED                                    [  7%]
test_shortener.py::test_shorten_and_resolve PASSED                       [ 15%]
test_shortener.py::test_same_url_same_code PASSED                        [ 23%]
test_shortener.py::test_rejects_non_http PASSED                          [ 30%]
test_shortener.py::test_alias_claimed_once PASSED                        [ 38%]
test_shortener.py::test_alias_race PASSED                                [ 46%]
test_shortener.py::test_concurrent_same_url_single_code PASSED           [ 53%]
test_shortener.py::test_clicks_are_exact_under_concurrency PASSED        [ 61%]
test_shortener.py::test_stats_uniques_and_recent_capped PASSED           [ 69%]
test_shortener.py::test_top_today PASSED                                 [ 76%]
test_shortener.py::test_expiring_link PASSED                             [ 84%]
test_shortener.py::test_delete_only_by_owner PASSED                      [ 92%]
test_shortener.py::test_generated_code_never_overwrites_alias PASSED     [100%]

============================== 13 passed in 1.65s ==============================
```

## The bug the last test found

The first version of `_create` wrote the new link with a plain `HSET` in a pipeline:

```python
def _create(self, code, url, owner, ttl):
    pipe = self.r.pipeline(transaction=True)
    pipe.hset(self._link(code), mapping={"target": url, "owner": owner,
                                         "created": int(time.time()), "clicks": 0})
    ...
```

Generated codes and custom aliases share one namespace, `short:link:<code>`. So what happens if someone claims
the alias `q0V` *before* the counter reaches it? A test for exactly that:

```python
def test_generated_code_never_overwrites_alias(s):
    s.shorten_alias("q0V", "https://example.com/alias-owner")      # the next generated code
    code = s.shorten("https://example.com/generated")
    assert s.resolve("q0V") == "https://example.com/alias-owner"
    assert code != "q0V" and s.resolve(code) == "https://example.com/generated"
```

```text
>       assert s.resolve("q0V") == "https://example.com/alias-owner"
E       AssertionError: assert 'https://exam...com/generated' == 'https://exam...m/alias-owner'
1 failed, 12 passed in 1.64s
```

The generated link **silently overwrote** someone's alias, because `HSET` overwrites. The fix is the same idea
used for aliases: claim with `HSETNX` first and take the next id if the claim fails. That's the `while True` loop in
`shorten` and the `bool` return of `_create` in the code above. The broader lesson for Redis design is to ask, for
every write, whether it can overwrite something it shouldn't. Redis will never complain about it.

(One more honest note: the first draft of `test_base62` expected `"q0T"` for 100,001. The arithmetic was wrong, not the code.
100,001 = 26·62² + 0·62 + 57, and digit 57 in this alphabet is `V`. The test caught the test.)

## Looking at the data

A short demo with prefix `demo`:

```python
import redis
from shortener import Shortener
s = Shortener(redis.Redis(decode_responses=True), prefix="demo")
code = s.shorten("https://redis.io/docs/latest/develop/data-types/", owner="ana")
print("code:", code)
for v, ref in [("u1", "twitter"), ("u2", "news"), ("u1", "news"), ("u3", "-")]:
    s.resolve(code, visitor=v, referrer=ref)
print(s.stats(code))
for k in sorted(s.r.scan_iter("demo:*")):
    print(f"{k:42} {s.r.type(k)}")
```

```text
code: q0V
{'target': 'https://redis.io/docs/latest/develop/data-types/', 'clicks': 4, 'ttl': -1, 'unique_today': 3, 'recent': ['1791855697|-', '1791855697|news', '1791855697|news', '1791855697|twitter']}
demo:by-target:19e9d250eb6c557366f8f89453d311de string
demo:link:q0V                              hash
demo:next-id                               string
demo:recent:q0V                            list
demo:top:2026-10-13                        zset
demo:visitors:q0V:2026-10-13               set
```

Four clicks from three distinct visitors, the newest click first, and six keys across five types, each named so
that `SCAN MATCH demo:*:q0V*` or `demo:top:*` finds a whole family.

## How It Actually Works

Each request touches only O(1) or O(log N) operations: `HGET`, `HINCRBY`, `LPUSH`/`LTRIM` (O(1) when trimming
one element off a short list), `SADD`, `ZINCRBY` (O(log N) in today's board). So resolve latency stays flat as the
number of links grows. The only O(N) operation is `stats`'s `LRANGE 0 4`, and N is 5.

Correctness rests on one rule: **every decision that must be unique happens inside a single Redis command.**
"Is this alias free?" and "take it" are one `HSETNX`. "Does this URL already have a code?" and "record mine" are
one `SET NX`. "Add a click" is one `HINCRBY`. Because Redis runs commands one at a time, these can't interleave
with other clients. The places where the code reads, decides in Python, then writes (`delete` checks the owner,
then unlinks) are exactly the places that *could* race. For `delete` the worst case is harmless: deleting a link
that was just deleted. Level 2 · 08–09 shows how to make read-decide-write atomic when it isn't harmless.

The daily keys use UTC dates, so "today" changes at midnight UTC for every server. Their 8-day TTL is refreshed on
each click, so a key expires 8 days after its *last* activity. That's a few redundant `EXPIRE` calls in exchange
for not needing a cleanup job.

## Common mistakes

- **Read-then-write claims** (`EXISTS` then `HSET`). Two clients both see "free". Claim in one command.
- **Overwrites across shared namespaces.** Generated codes and aliases collided in the first version. Look for
  them whenever two code paths write the same key pattern.
- **Counting uniques in an ever-growing set.** Use per-period keys with TTLs, or HyperLogLog (Level 2 · 03) for
  big sites.
- **Using `KEYS` for the "top links" page.** Maintain the sorted set as you go instead.
- **Trusting user input as a redirect target.** The `http(s)` check is the minimum; a real service also needs
  abuse and malware checks.

## Exercise

1. Add `short:links:<owner>` (a sorted set of codes scored by creation time) so a user can list their links,
   newest first, 10 per page. Keep it consistent in `shorten`, `shorten_alias` and `delete`.
2. Replace sequential codes with 7-character random codes claimed via `HSETNX`. Write a test that forces a
   collision (by seeding the random generator, or by pre-claiming the code) and checks that the retry works.
3. Add a per-link daily click histogram using a hash `short:hourly:<code>:<date>` with one field per hour, and
   expose it from `stats`.
4. Benchmark `resolve` with 1,000,000 links and 100,000 resolves from 8 threads. Report requests per second and
   the server's `used_memory` per link.
