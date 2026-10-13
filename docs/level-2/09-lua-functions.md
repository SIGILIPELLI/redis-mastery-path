---
description: "Server-side logic with Lua: EVAL and EVALSHA, KEYS vs ARGV, conversion traps, a transfer 7x faster than WATCH, BUSY scripts, and Functions."
---

# 09 · Lua Scripts & Redis Functions

`WATCH` let the last lesson do read-decide-write correctly, but under contention it spent 2.6 failed attempts for
every success. Redis's other tool takes the opposite approach: instead of bringing data to the client to make a
decision, it sends the decision to the data. A **Lua script** runs inside the server, reads and writes keys with
`redis.call`, and executes **atomically**, because nothing else runs while it does. There are no retries and no
round trips between the read and the write. **Redis Functions** (7.0+) are the same idea packaged as named,
persistent libraries.

## EVAL basics

```text
127.0.0.1:6379> EVAL "return 'hello from Lua'" 0
"hello from Lua"
```

The `0` is the number of key names that follow. Key names go in `KEYS`, and everything else goes in `ARGV`. Here's
a conditional stock reservation, the check-then-act problem solved in one command:

```text
127.0.0.1:6379> SET stock:pen 3
OK
127.0.0.1:6379> EVAL "local n = tonumber(redis.call('GET', KEYS[1])) if n >= tonumber(ARGV[1]) then return redis.call('DECRBY', KEYS[1], ARGV[1]) else return -1 end" 1 stock:pen 2
(integer) 1
127.0.0.1:6379> EVAL "local n = tonumber(redis.call('GET', KEYS[1])) if n >= tonumber(ARGV[1]) then return redis.call('DECRBY', KEYS[1], ARGV[1]) else return -1 end" 1 stock:pen 2
(integer) -1
```

The first call reserved 2 and left 1. The second saw 1 < 2 and refused. No other client could act between the `GET`
and the `DECRBY`.

**Always pass key names through `KEYS`**, even though the script could build names from `ARGV`. Redis Cluster uses
`KEYS` to route the script to the node that owns those keys and to check that they share a slot, and ACLs use them
to check key permissions. A script that touches keys it didn't declare works on a single server and fails or
misbehaves in Cluster.

## Type conversion traps

Values cross the boundary between Lua and Redis with conversions that bite:

```text
127.0.0.1:6379> EVAL "return {1, 2.9, 'three', true, false, nil, 'after nil'}" 0
1) (integer) 1
2) (integer) 2
3) "three"
4) (integer) 1
5) (nil)
```

- Lua numbers become **integers, truncated**: `2.9` came back as `2`. Return `tostring(x)` to keep decimals.
- `true` becomes 1, and `false` becomes a null reply.
- A Lua table stops at the first `nil`, so `'after nil'` was silently dropped.
- Going the other way, `redis.call('GET', missing)` returns `false` (not `nil`), which is why scripts write
  `redis.call('GET', k) or '0'`.

Errors are returned with a table that has an `err` field, or with `redis.error_reply`:

```text
127.0.0.1:6379> EVAL "return {err='custom failure'}" 0
(error) custom failure
```

## No rollback here either

A script is atomic in the sense that nothing interleaves with it. It is **not** atomic in the sense of "all or
nothing" if it fails halfway:

```text
127.0.0.1:6379> SET word hello
OK
127.0.0.1:6379> EVAL "redis.call('SET', KEYS[1], 'changed'); redis.call('INCR', KEYS[2]); return 'done'" 2 a word
(error) ERR value is not an integer or out of range script: ae517de5ce89a18c8dd8aa50fd582c7861aa7909, on @user_script:1.
127.0.0.1:6379> GET a
"changed"
```

The `SET` before the failing `INCR` stayed. `redis.call` raises the error and aborts the script, while
`redis.pcall` returns it as a value so the script can handle it:

```text
127.0.0.1:6379> EVAL "local ok = redis.pcall('INCR', KEYS[1]); if type(ok) == 'table' and ok.err then return 'caught: ' .. ok.err end; return ok" 1 word
"caught: ERR value is not an integer or out of range"
```

The robust pattern is **validate first, then write**. Do every read and check at the top of the script and return
early if anything is wrong, so the writes at the bottom can't fail for data reasons.

## EVALSHA and the script cache

Sending the full script text on every call wastes bandwidth. `SCRIPT LOAD` caches a script and returns its SHA-1,
and `EVALSHA` runs it by hash:

```text
127.0.0.1:6379> SCRIPT LOAD "return redis.call('INCRBY', KEYS[1], ARGV[1])"
"8cd00688c05c46bde4a2e60658ef20a2e5c0b248"
127.0.0.1:6379> EVALSHA 8cd00688c05c46bde4a2e60658ef20a2e5c0b248 1 counter 5
(integer) 5
127.0.0.1:6379> EVALSHA 4bc6f6e5b2bdbcbf0b18d1d5f9a31eb38e18f4b2 1 counter 5
(error) NOSCRIPT No matching script. Please use EVAL.
```

The script cache is **not persistent**. After `SAVE`, a restart and a check:

```bash
redis-cli SAVE
redis-cli SHUTDOWN NOSAVE
redis-server ~/redis-lab/redis.conf      # loads dump.rdb
redis-cli SCRIPT EXISTS 8cd00688c05c46bde4a2e60658ef20a2e5c0b248
```

```text
0
```

The cache was empty. It's also empty on a freshly promoted replica and after `SCRIPT FLUSH`. Clients must handle
`NOSCRIPT` by falling back to `EVAL`. redis-py's `register_script` does exactly that:

```python
TRANSFER = r.register_script("""
local bal = tonumber(redis.call('GET', KEYS[1]))
local amt = tonumber(ARGV[1])
if bal < amt then return 0 end
redis.call('DECRBY', KEYS[1], amt)
redis.call('INCRBY', KEYS[2], amt)
return 1
""")

def transfer_lua(c, src, dst, amt):
    return TRANSFER(keys=[src, dst], args=[amt], client=c) == 1
```

## The transfer benchmark, again

Same workload as Level 2 · 08 (16 threads, 2,000 random transfers each, 5 accounts of 100), timed with `WATCH`
and with the script:

```text
WATCH  12.08s  total=500 min=22 transfers=21874 retries=56784
Lua     1.70s  total=500 min=6 transfers=25179 retries=0
```

Both kept the total at 500 and never overdrew an account. The script was **7× faster**, with zero retries and a
single round trip per transfer. It also completed more transfers. Fewer were refused for insufficient funds,
because each decision saw the current balance instead of a balance that was stale by the time `EXEC` ran. Under
contention, scripts win. With rare conflicts and logic that needs client-side data, `WATCH` is fine.

## The cost of atomicity: a slow script blocks everyone

While a script runs, the server runs nothing else. Here's a script with an infinite loop, using a 1-second
`busy-reply-threshold` (the default is 5000 ms) so the demo stays short:

```python
r.config_set("busy-reply-threshold", 1000)
# thread 1: c.eval("local i = 0 while true do i = i + 1 end", 0)
# thread 2, 0.2 s later:
other.get("anything")
```

```text
other client after 0.8s: BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.
SCRIPT KILL -> True
script client after 1.0s: Script killed by user with SCRIPT KILL... script: 3fcf51a9b854b8f66cb8685600b0d7df968f4398, on @user_script:1.
```

After the threshold, Redis starts answering other clients with `BUSY` and allows `SCRIPT KILL`. But if the script has
already **written** anything, killing it would leave a half-applied change, so Redis refuses:

```text
SCRIPT KILL on a script that wrote -> UNKILLABLE Sorry the script already executed write commands against the
dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
```

That's not a figure of speech. The lab server had to be shut down with `SHUTDOWN NOSAVE` and restarted, losing
everything since its last save. Keep scripts short and bounded: no unbounded loops, and no `KEYS`-style scans over
large collections inside a script.

## Redis Functions: named, persistent libraries

Functions fix the script cache's weaknesses. A **library** is loaded once with `FUNCTION LOAD`, it's stored in the
dataset (so it's persisted in RDB/AOF and replicated), and its functions are called by name with `FCALL`:

```lua
#!lua name=bank

local function transfer(keys, args)
  local amt = tonumber(args[1])
  if not amt or amt <= 0 then
    return redis.error_reply('ERR amount must be a positive number')
  end
  local bal = tonumber(redis.call('GET', keys[1]) or '0')
  if bal < amt then return 0 end
  redis.call('DECRBY', keys[1], amt)
  redis.call('INCRBY', keys[2], amt)
  return 1
end

local function balance(keys, args)
  return tonumber(redis.call('GET', keys[1]) or '0')
end

redis.register_function('transfer', transfer)
redis.register_function{function_name='balance', callback=balance, flags={'no-writes'}}
```

```bash
redis-cli -x FUNCTION LOAD < bank.lua
```

```text
bank
```

```text
127.0.0.1:6379> MSET acct:ana 100 acct:ben 0
OK
127.0.0.1:6379> FCALL transfer 2 acct:ana acct:ben 30
(integer) 1
127.0.0.1:6379> FCALL transfer 2 acct:ana acct:ben 500
(integer) 0
127.0.0.1:6379> FCALL transfer 2 acct:ana acct:ben -5
(error) ERR amount must be a positive number
127.0.0.1:6379> FCALL_RO balance 1 acct:ben
(integer) 30
127.0.0.1:6379> FCALL_RO transfer 2 acct:ana acct:ben 1
(error) ERR Can not execute a script with write flag using *_ro command.
```

- The `no-writes` flag allows `FCALL_RO`, which can run on read-only replicas. The server refused to run the
  writing function in read-only mode.
- Loading the same library again fails (`ERR Library 'bank' already exists`). Deploy new versions with
  `FUNCTION LOAD REPLACE`.
- After the same save-and-restart that emptied the script cache, `FUNCTION LIST` still showed `bank`.

Treat function libraries like database migrations. Keep them in version control and load them with `REPLACE` as
part of deployment. Every node, including replicas created later, gets them through replication and persistence.

## How It Actually Works

Redis embeds a Lua 5.1 interpreter. `EVAL` hashes the script body with SHA-1, compiles it into a function named
after the hash (that's the `f_<sha>` the cache holds), and calls it with `KEYS` and `ARGV` set as globals.
`redis.call` goes through the same command dispatcher a client uses, but within the same execution, so there's no
event-loop iteration between calls and no other client runs. Scripts run in a sandbox: global variables can't be
created, and libraries like `os` and `io` aren't available, which keeps scripts deterministic and safe.

Replication used to send the script itself to replicas, which required scripts to be deterministic. Since Redis
7.0, only **effects** replication is used: the write commands a script performed are wrapped in `MULTI`/`EXEC` and
sent to replicas and the AOF. That's why a script can safely use `TIME` or random numbers, and also why its effects,
not its logic, are what's replayed.

During a long script, the server periodically checks the elapsed time from a hook inside the Lua VM. After
`busy-reply-threshold`, it starts processing other connections' input just enough to answer them with `BUSY` and
to accept `SCRIPT KILL`/`SHUTDOWN NOSAVE`. `SCRIPT KILL` raises an error inside the VM, but only if the script
hasn't written yet, because there's no undo log to roll a write back.

Functions use the same VM and dispatcher. The difference is lifecycle: the library code is stored in the server's
function registry, which is serialized into RDB and AOF and replicated with `FUNCTION LOAD`. Flags declared at
registration (`no-writes`, `allow-oom`, `allow-stale`, …) let the server decide before execution whether a call
is allowed in the current context.

## Common mistakes

- **Building key names inside the script** instead of passing them in `KEYS`. It breaks Cluster routing and ACLs.
- **Returning floats** and getting truncated integers. Return strings.
- **Writing first, validating later.** Errors mid-script leave partial writes. Validate at the top.
- **Long or unbounded scripts.** They block everything, and once they've written they can't be killed.
- **Relying on the script cache surviving.** Handle `NOSCRIPT`, or use Functions.
- **Hot-reloading functions without `REPLACE`**, or editing them by hand on one node in production.
- **Using Lua to "speed up" ordinary commands.** Pipelining already removes round trips. Use scripts for
  atomic decisions.

## Exercise

1. Write the delayed-job claim from Level 1 · 08 as a script: atomically take up to N members of `jobs:delayed`
   whose score ≤ now, remove them, and push them onto `jobs:ready`. Run two pollers and prove no job is taken twice.
2. Write an idempotent append that works on any Redis version: `SET idem:<request-id> 1 NX EX 600` and `XADD` only
   if the `SET` succeeded, returning the previously stored stream ID on retries (store it in the idem key).
3. Package exercises 1 and 2 as a Functions library `jobs` with sensible flags, load it, restart the server, and
   show it's still there.
4. Measure the overhead of `EVALSHA` versus a plain `INCR` with redis-benchmark or a Python loop. When is the script
   overhead worth paying?
