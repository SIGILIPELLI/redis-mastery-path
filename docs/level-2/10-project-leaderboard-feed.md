---
description: "Level 2 project: a weekly leaderboard with first-come tie-breaks, an activity stream, HyperLogLog counts, Pub/Sub alerts and a badge worker."
---

# 10 · Project — Game Leaderboard & Activity Feed

This project combines Level 2 into one feature set a real game backend needs:

- a **weekly leaderboard** that keeps each player's best score, where ties go to whoever got there **first**;
- "**players around me**" and top-N queries;
- an **activity stream** of every submission, trimmed automatically;
- an approximate count of **distinct players this week**;
- a **Pub/Sub** announcement when someone enters the top 10;
- a **badge worker** that reads the stream through a consumer group and never double-awards, even after a crash.

The write path, a score submission, touches four structures. It has to be atomic and decide things ("did this beat
your best?", "what's your rank now?") based on current data. That's a job for a **Redis Function**.

## The score encoding

Sorted sets break ties alphabetically (Level 1 · 08). To make equal points rank by submission time, pack both into
one score:

```text
score = points × 2^20 + (2^20 − 1 − seconds_since_week_start)
```

A week is 604,800 seconds, which is less than 2^20 (1,048,576). So the time part never overflows into the points
part. With equal points, an earlier submission leaves a larger remainder and ranks higher. Doubles hold integers
exactly up to 2^53, so points can go up to 2^33 (about 8.6 billion) before precision is lost. `ZADD GT` keeps the
higher composite, which gives "personal best" semantics. A later submission with equal points *doesn't* replace the
earlier one, because its composite is smaller.

## The function library

`leaderboard.lua`:

```lua
#!lua name=leaderboard

-- score layout: points * 2^20 + (2^20 - 1 - seconds since week start)
-- a week is 604,800 s < 2^20, so equal points rank the EARLIER submission higher
local SHIFT = 1048576

local function submit(keys, args)
  local board, activity, players = keys[1], keys[2], keys[3]
  local player, points = args[1], tonumber(args[2])
  local now_ms, week_start_ms = tonumber(args[3]), tonumber(args[4])
  local maxlen = args[5]
  if not points or points < 0 or points ~= math.floor(points) then
    return redis.error_reply('ERR points must be a non-negative integer')
  end
  local secs = math.floor((now_ms - week_start_ms) / 1000)
  if secs < 0 or secs >= SHIFT then
    return redis.error_reply('ERR timestamp outside this week')
  end
  local composite = points * SHIFT + (SHIFT - 1 - secs)
  local improved = redis.call('ZADD', board, 'GT', 'CH', composite, player)
  local rank = redis.call('ZREVRANK', board, player)
  redis.call('PFADD', players, player)
  redis.call('XADD', activity, 'MAXLEN', '~', maxlen, '*',
             'player', player, 'points', points, 'best', tostring(improved == 1), 'rank', rank + 1)
  if improved == 1 and rank < 10 then
    redis.call('PUBLISH', board .. ':top10', player .. ' is now #' .. (rank + 1))
  end
  return {improved, rank + 1}
end

redis.register_function('lb_submit', submit)
```

Notes:

- **All three keys come in through `KEYS`** and share the hash tag `{<game>}`, so in Cluster they live in one slot
  and the function can run there (Level 4 · 05).
- Validation comes first, and the writes come after, so a bad input can't leave a partial update (Level 2 · 09).
- `ZADD GT CH` returns 1 only if the score was **changed**, which is exactly "improved your best".
- The activity entry records whether it was a best and the rank at the time. The stream is trimmed with
  `MAXLEN ~`, so it costs nothing extra to keep bounded.
- `PUBLISH` inside a function is fine. Like any write, it's propagated as an effect.

## The Python side

`game.py`:

```python
"""Weekly game leaderboard + activity feed (Level 2 project)."""
import datetime as dt
import time
from pathlib import Path

import redis

SHIFT = 1 << 20
LIB = Path(__file__).with_name("leaderboard.lua").read_text()


def week_start(now: float) -> dt.datetime:
    d = dt.datetime.fromtimestamp(now, tz=dt.timezone.utc)
    monday = d - dt.timedelta(days=d.weekday())
    return monday.replace(hour=0, minute=0, second=0, microsecond=0)


def decode(composite: float) -> int:
    return int(composite) // SHIFT


class Leaderboard:
    def __init__(self, r: redis.Redis, game: str = "arena", feed_len: int = 1000):
        self.r, self.game, self.feed_len = r, game, feed_len
        r.function_load(LIB, replace=True)

    def _keys(self, now):
        ws = week_start(now)
        tag = ws.strftime("%G-W%V")                      # ISO week, e.g. 2026-W42
        base = f"lb:{{{self.game}}}"                     # hash tag keeps all keys in one slot
        return ws, (f"{base}:board:{tag}", f"{base}:activity", f"{base}:players:{tag}")

    def submit(self, player: str, points: int, now: float | None = None):
        now = time.time() if now is None else now
        ws, keys = self._keys(now)
        improved, rank = self.r.fcall("lb_submit", 3, *keys, player, points,
                                      int(now * 1000), int(ws.timestamp() * 1000), self.feed_len)
        board = keys[0]
        self.r.expire(board, 35 * 24 * 3600, nx=True)    # keep ~5 weeks of boards
        self.r.expire(keys[2], 35 * 24 * 3600, nx=True)
        return bool(improved), rank

    def top(self, n=10, now=None):
        _, (board, _, _) = self._keys(time.time() if now is None else now)
        rows = self.r.zrange(board, 0, n - 1, desc=True, withscores=True)
        return [(i + 1, p, decode(s)) for i, (p, s) in enumerate(rows)]

    def around(self, player, k=2, now=None):
        _, (board, _, _) = self._keys(time.time() if now is None else now)
        rank = self.r.zrevrank(board, player)
        if rank is None:
            return []
        start = max(0, rank - k)
        rows = self.r.zrange(board, start, rank + k, desc=True, withscores=True)
        return [(start + i + 1, p, decode(s)) for i, (p, s) in enumerate(rows)]

    def players_this_week(self, now=None):
        _, (_, _, players) = self._keys(time.time() if now is None else now)
        return self.r.pfcount(players)

    def feed(self, n=5):
        _, (_, activity, _) = self._keys(time.time())
        return [f for _, f in self.r.xrevrange(activity, count=n)]
```

Boards are per ISO week (`2026-W42`), so a "weekly reset" is just a new key. Old boards keep a 35-day TTL, set once
with `EXPIRE … NX`, so history is available for a few weeks and then disappears without a cleanup job.

`achievements.py`, the badge worker:

```python
"""Consumer-group worker: awards badges from the activity stream, at-least-once, idempotently."""
import redis

BADGES = [(1000, "bronze"), (5000, "silver"), (10000, "gold")]


def ensure_group(r, stream, group="badges"):
    try:
        r.xgroup_create(stream, group, id="0", mkstream=True)
    except redis.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise


def process_batch(r, stream, consumer, group="badges", count=50, min_idle_ms=30000):
    _, msgs, _ = r.xautoclaim(stream, group, consumer, min_idle_time=min_idle_ms, start_id="0-0", count=count)
    if not msgs:
        res = r.xreadgroup(group, consumer, {stream: ">"}, count=count, block=100)
        msgs = res[0][1] if res else []
    awarded = 0
    for msg_id, f in msgs:
        pts = int(f["points"])
        for threshold, badge in BADGES:
            if pts >= threshold:
                # SADD returns 1 only the first time: re-delivery never double-awards
                awarded += r.sadd(f"badges:{f['player']}", badge)
        r.xack(stream, group, msg_id)
    return len(msgs), awarded
```

The worker follows the recovery loop from Level 2 · 06: claim stale entries first, then read new ones. Badges are
stored in a set, so `SADD`'s return value tells you whether this delivery actually awarded anything new. A message
delivered twice (after a crash) awards nothing the second time. **At-least-once delivery plus an idempotent
handler gives effectively-once results.**

## Tests

`test_game.py` pins time to a fixed instant (Tuesday 2026-10-13 02:00 UTC, ISO week 2026-W42) so ranks and week keys
are deterministic:

```python
import threading, time
import pytest, redis
from game import Leaderboard, week_start
from achievements import ensure_group, process_batch

NOW = 1791856800.0          # a fixed instant (Tue 2026-10-13 UTC) so tests are deterministic

@pytest.fixture
def lb():
    r = redis.Redis(db=0, decode_responses=True)
    for k in r.scan_iter("lb:{test}*"): r.unlink(k)
    for k in r.scan_iter("badges:*"): r.unlink(k)
    yield Leaderboard(r, game="test", feed_len=100)
    for k in r.scan_iter("lb:{test}*"): r.unlink(k)

def test_personal_best_only_goes_up(lb):
    assert lb.submit("ana", 500, NOW) == (True, 1)
    assert lb.submit("ana", 300, NOW + 1) == (False, 1)
    assert lb.submit("ana", 900, NOW + 2) == (True, 1)
    assert lb.top(1, NOW) == [(1, "ana", 900)]

def test_ties_go_to_whoever_got_there_first(lb):
    lb.submit("late", 700, NOW + 50)
    lb.submit("early", 700, NOW + 10)
    lb.submit("ben", 650, NOW)
    assert [p for _, p, _ in lb.top(3, NOW)] == ["early", "late", "ben"]

def test_equal_score_later_does_not_replace_earlier(lb):
    lb.submit("ana", 700, NOW)
    improved, _ = lb.submit("ana", 700, NOW + 100)
    assert improved is False

def test_rejects_bad_points(lb):
    with pytest.raises(redis.ResponseError, match="non-negative integer"):
        lb.submit("ana", -5, NOW)

def test_around_me(lb):
    for i in range(20):
        lb.submit(f"p{i:02}", i * 10, NOW)
    around = lb.around("p10", k=2, now=NOW)
    assert [(rank, p) for rank, p, _ in around] == [(8, "p12"), (9, "p11"), (10, "p10"), (11, "p09"), (12, "p08")]
    assert lb.around("nobody", now=NOW) == []

def test_new_week_new_board(lb):
    lb.submit("ana", 100, NOW)
    next_week = week_start(NOW).timestamp() + 7 * 24 * 3600 + 5
    lb.submit("ben", 50, next_week)
    assert [p for _, p, _ in lb.top(5, NOW)] == ["ana"]
    assert [p for _, p, _ in lb.top(5, next_week)] == ["ben"]

def test_concurrent_submissions_keep_max(lb):
    def player(i):
        for s in range(0, 2000, 7):
            lb.submit(f"c{i}", s, NOW)
    ts = [threading.Thread(target=player, args=(i,)) for i in range(8)]
    [t.start() for t in ts]; [t.join() for t in ts]
    top = lb.top(8, NOW)
    assert all(score == 1995 for _, _, score in top) and len(top) == 8

def test_feed_and_uniques(lb):
    lb.submit("ana", 10, NOW); lb.submit("ben", 20, NOW); lb.submit("ana", 5, NOW)
    feed = lb.feed(3)
    assert [(f["player"], f["points"], f["best"]) for f in feed] == [("ana", "5", "false"), ("ben", "20", "true"), ("ana", "10", "true")]
    assert lb.players_this_week(NOW) == 2

def test_feed_is_trimmed(lb):
    for i in range(1000):
        lb.submit(f"x{i}", i, NOW)
    assert lb.r.xlen("lb:{test}:activity") < 1000

def test_top10_announcements(lb):
    p = lb.r.pubsub(); p.subscribe("lb:{test}:board:2026-W42:top10"); time.sleep(0.1)
    lb.submit("ana", 100, NOW)
    msgs = []
    end = time.time() + 1
    while time.time() < end:
        m = p.get_message(timeout=0.1)
        if m and m["type"] == "message": msgs.append(m["data"])
    assert msgs == ["ana is now #1"]

def test_badges_awarded_once_even_if_redelivered(lb):
    stream = "lb:{test}:activity"
    lb.submit("ana", 1200, NOW); lb.submit("ana", 6000, NOW + 1)
    ensure_group(lb.r, stream)
    # consumer "crashy" reads but never acks
    lb.r.xreadgroup("badges", "crashy", {stream: ">"}, count=10)
    n, awarded = process_batch(lb.r, stream, "w1", min_idle_ms=0)      # reclaims crashy's entries
    n2, awarded2 = process_batch(lb.r, stream, "w1", min_idle_ms=0)
    assert (n, awarded) == (2, 2) and awarded2 == 0
    assert lb.r.smembers("badges:ana") == {"bronze", "silver"}
    assert lb.r.xpending(stream, "badges")["pending"] == 0
```

```text
test_game.py::test_personal_best_only_goes_up PASSED                     [  9%]
test_game.py::test_ties_go_to_whoever_got_there_first PASSED             [ 18%]
test_game.py::test_equal_score_later_does_not_replace_earlier PASSED     [ 27%]
test_game.py::test_rejects_bad_points PASSED                             [ 36%]
test_game.py::test_around_me PASSED                                      [ 45%]
test_game.py::test_new_week_new_board PASSED                             [ 54%]
test_game.py::test_concurrent_submissions_keep_max PASSED                [ 63%]
test_game.py::test_feed_and_uniques PASSED                               [ 72%]
test_game.py::test_feed_is_trimmed PASSED                                [ 81%]
test_game.py::test_top10_announcements PASSED                            [ 90%]
test_game.py::test_badges_awarded_once_even_if_redelivered PASSED        [100%]

============================== 11 passed in 2.22s ==============================
```

The concurrency test has 8 threads each submitting rising scores up to 1,995 for its own player, and the board ends
with all eight at exactly 1,995. The badge test leaves two entries pending under a "crashy" consumer, reclaims them
with `XAUTOCLAIM`, processes them twice, and checks that the second pass awards nothing and nothing is left pending.

## Running it with load

`demo.py` submits 40,000 scores for 50,000 possible players from 8 threads, with long-tailed point values:

```python
import random, threading, time
import redis
from game import Leaderboard
r = redis.Redis(decode_responses=True)
for k in r.scan_iter("lb:{demo}*"): r.unlink(k)
lb = Leaderboard(r, game="demo")
random.seed(5)
players = [f"player{i}" for i in range(50_000)]
def worker(seed):
    rnd = random.Random(seed); c = Leaderboard(redis.Redis(decode_responses=True), game="demo")
    for _ in range(5000):
        c.submit(rnd.choice(players), int(rnd.paretovariate(1.5) * 100))
t = time.perf_counter()
ts = [threading.Thread(target=worker, args=(i,)) for i in range(8)]
[x.start() for x in ts]; [x.join() for x in ts]
el = time.perf_counter() - t
print(f"40,000 submissions from 8 threads in {el:.2f}s ({40000/el:,.0f}/s)")
for row in lb.top(5): print("top", row)
me = lb.top(500)[-1][1]
for row in lb.around(me, k=1): print("around", row)
print("players this week ~", lb.players_this_week())
print("feed:", lb.feed(2))
board = [k for k in r.scan_iter("lb:{demo}:board:*")][0]
print(board, "members", r.zcard(board), "memory", r.memory_usage(board), "| activity XLEN", r.xlen("lb:{demo}:activity"))
```

```text
40,000 submissions from 8 threads in 6.20s (6,447/s)
top (1, 'player31559', 80816)
top (2, 'player16280', 64898)
top (3, 'player25408', 55269)
top (4, 'player45704', 49827)
top (5, 'player44481', 40786)
around (499, 'player19634', 1930)
around (500, 'player18416', 1928)
around (501, 'player13286', 1927)
players this week ~ 27562
feed: [{'player': 'player25101', 'points': '108', 'best': 'false', 'rank': '2514'}, {'player': 'player18754', 'points': '102', 'best': 'false', 'rank': '8834'}]
lb:{demo}:board:2026-W42 members 27559 memory 2173259 | activity XLEN 1000
```

- About 6,500 submissions per second from one Python process with 8 threads. Each submission is three round trips
  (the `FCALL` and two `EXPIRE … NX`), and Python's thread overhead dominates. The server time per call is far
  smaller. Folding the `EXPIRE`s into the function would make it one round trip (Exercise 2).
- The HyperLogLog estimated **27,562** players this week. The board, which is exact, has **27,559** members.
- The week's board for about 27,600 players uses 2.2 MB. The activity stream stayed at its 1,000-entry target.

## How It Actually Works

Each submission is one `FCALL`, executed atomically on the server. Inside it: `ZADD GT CH` (O(log N)), `ZREVRANK`
(O(log N), using the skip-list spans from Level 1 · 08), `PFADD` (O(1)), `XADD` with approximate trimming (amortized
O(1)) and, rarely, a `PUBLISH`. No other client can see the board updated without the activity entry, or read a
rank that doesn't match the score just written.

Ranks are read **after** the `ZADD`, inside the same function, so the rank in the activity entry and in the return
value is exactly the rank the submission produced. A client doing `ZADD` and then `ZREVRANK` in two round trips
could report a rank that someone else's submission had already changed.

The week tag is computed by the client from its clock, and the `secs` bound check in the function rejects
timestamps outside the week. If app servers' clocks disagree near midnight on Sunday, a submission can land on the
"wrong" week. Using the Redis server's time (`redis.call('TIME')`) inside the function would remove that
dependency, at the cost of making tests harder to pin. That's a trade-off worth knowing you made.

## Common mistakes

- **Separate `ZADD` and `ZREVRANK` round trips** to report "your new rank". It's racy. Compute it in the same atomic
  unit.
- **Alphabetical tie-breaking** by accident. Encode the tie-breaker into the score, and check the arithmetic against
  2^53.
- **A forever-growing activity log.** Trim on every `XADD`.
- **Non-idempotent consumers.** A redelivered "award gold badge" must not award twice.
- **Weekly resets by deleting the board** with a cron job. Rotate key names, and let TTLs clean up.
- **Keys without a shared hash tag** in a multi-key function. It works on one node and fails in Cluster.

## Exercise

1. Add per-country boards (`board:<week>:<country>`) updated in the same `FCALL`, and a `top_in_country` method.
   What changes about the keys you pass, and what does that mean for Cluster?
2. Move the two `EXPIRE … NX` calls into the function and measure the throughput change with `demo.py`.
3. Use the server clock (`redis.call('TIME')`) inside the function instead of client timestamps. How do you keep the
   tests deterministic now?
4. Add an all-time board that tracks the best score ever, updated with the same function. What's the size of the
   all-time board after a year if 2 million distinct players have played? Estimate it from this lesson's
   bytes-per-member, then measure it.
