---
description: "redis-cli in depth — output modes, databases, -x, --pipe mass insertion, MONITOR, --bigkeys — and the RESP2/RESP3 wire protocol underneath it."
---

# 02 · redis-cli & the RESP Protocol

Every Redis client, whether it's `redis-cli`, redis-py, node-redis or Jedis, does the same job. It turns a
command like `SET city Porto` into bytes, sends them over TCP and parses the reply. The format of those bytes is
**RESP**, the Redis Serialization Protocol. It's simple enough to speak by hand, and it pays to have done that
once: it shows you why pipelining is so fast, what `(nil)` really is, and why a client library can return a
`float` from one server and a `str` from another.

All examples use the lab server on port 6379.

## Speaking RESP by hand

A request is an **array of bulk strings**. `*1` means "an array of one element", and `$4` means "a string of 4
bytes follows". Every line ends with `\r\n`. Here is a small Python script that opens a raw socket and sends
RESP:

```python
import socket, time

s = socket.create_connection(("127.0.0.1", 6379))
def send(raw):
    s.sendall(raw)
    time.sleep(0.05)
    print(repr(s.recv(65536)))

send(b"*1\r\n$4\r\nPING\r\n")
send(b"*3\r\n$3\r\nSET\r\n$4\r\ncity\r\n$5\r\nPorto\r\n")
send(b"*2\r\n$3\r\nGET\r\n$4\r\ncity\r\n")
send(b"*2\r\n$3\r\nGET\r\n$7\r\nmissing\r\n")
send(b"*2\r\n$4\r\nINCR\r\n$4\r\nhits\r\n")
send(b"*2\r\n$4\r\nINCR\r\n$4\r\ncity\r\n")
```

```text
b'+PONG\r\n'
b'+OK\r\n'
b'$5\r\nPorto\r\n'
b'$-1\r\n'
b':1\r\n'
b'-ERR value is not an integer or out of range\r\n'
```

The **first byte** of every reply tells the client what type follows:

| Prefix | RESP2 type | Example above |
|--------|------------|---------------|
| `+` | simple string (status) | `+PONG`, `+OK` |
| `-` | error | `-ERR value is not an integer…` |
| `:` | integer | `:1` from `INCR` |
| `$` | bulk string (length-prefixed, binary-safe) | `$5\r\nPorto` |
| `$-1` | null bulk string | `GET missing` |
| `*` | array | replies to `LRANGE`, `HGETALL`… |

Two details matter in practice. Bulk strings carry a length, so a value can contain `\r\n`, NUL bytes or a JPEG,
and nothing has to be escaped. And an error is a normal reply, not a dropped connection: the client stays
connected and the next command works.

Because the format is just bytes on a stream, you can send several commands in one write and read all the
replies afterwards:

```python
send(b"*4\r\n$5\r\nRPUSH\r\n$1\r\nq\r\n$1\r\na\r\n$1\r\nb\r\n"
     b"*4\r\n$6\r\nLRANGE\r\n$1\r\nq\r\n$1\r\n0\r\n$2\r\n-1\r\n")
```

```text
b':2\r\n*2\r\n$1\r\na\r\n$1\r\nb\r\n'
```

Both replies arrived in one read, in order. That is **pipelining**, and it's the single biggest performance
lever on the client side (measured below).

Redis also accepts **inline commands**, which are plain space-separated text, mainly so you can poke a server with
`telnet` or `nc`:

```python
send(b"EXISTS city hits nope\r\n")
```

```text
b':2\r\n'
```

Clients never use inline commands: they can't carry binary data or spaces inside arguments.

## RESP3: richer types after HELLO 3

RESP2 has no maps, no floats and no sets. `HGETALL` returns a flat array `[field, value, field, value]` and
`ZSCORE` returns a number as a *string*. **RESP3** (Redis 6+) adds real types, and a connection opts in with
`HELLO 3`:

```python
send(b"*2\r\n$5\r\nHELLO\r\n$1\r\n3\r\n")
```

```text
b'%7\r\n$6\r\nserver\r\n$5\r\nredis\r\n$7\r\nversion\r\n$6\r\n8.10.2\r\n$5\r\nproto\r\n:3\r\n
  $2\r\nid\r\n:16\r\n$4\r\nmode\r\n$10\r\nstandalone\r\n$4\r\nrole\r\n$6\r\nmaster\r\n ...'
```

(Wrapped for width.) `%7` is a **map** with seven entries. Compare the same commands on a RESP2 connection
and a RESP3 connection:

```text
RESP2  ZSCORE z m      ->  b'$3\r\n1.5\r\n'          bulk string "1.5"
RESP3  ZSCORE z m      ->  b',1.5\r\n'               double
RESP3  HGETALL h       ->  b'%1\r\n$1\r\nf\r\n$1\r\nv\r\n'   map
RESP3  SMEMBERS s      ->  b'~1\r\n$1\r\nx\r\n'       set
RESP3  GET missing     ->  b'_\r\n'                   null
```

RESP3 also adds booleans (`#t`), big numbers (`(`), verbatim strings (`=`) and **push** messages (`>`).
Push messages are out-of-band data that the server sends without being asked. They're how Pub/Sub messages and
client-side-cache invalidations can share a connection with normal commands (Level 3 · 06).

This difference reaches your application code. A client that negotiates RESP3 can hand you a native `float`
for a score and a dict for a hash, while a RESP2 client may give you bytes that you have to convert. Level 1 · 09
shows what redis-py 8 does about that.

## redis-cli in interactive mode

Run `redis-cli` with no command to get a prompt. It formats each reply type so you can tell them apart:

```text
127.0.0.1:6379> SET city Porto
OK
127.0.0.1:6379> GET city
"Porto"
127.0.0.1:6379> GET missing
(nil)
127.0.0.1:6379> INCR hits
(integer) 2
127.0.0.1:6379> INCR city
(error) ERR value is not an integer or out of range
127.0.0.1:6379> RPUSH q a b
(integer) 4
127.0.0.1:6379> LRANGE q 0 -1
1) "a"
2) "b"
3) "a"
4) "b"
127.0.0.1:6379> ZSCORE z m
"1.5"
```

`"Porto"` with quotes is a bulk string, `OK` without quotes is a status reply, and `(nil)` is the null bulk
string. (`RPUSH` returned 4 because the raw-socket demo had already pushed `a` and `b`; lists keep
duplicates.) Switch the same session to RESP3 and the formatting follows the new types:

```text
127.0.0.1:6379> HELLO 3
1# "server" => "redis"
2# "version" => "8.10.2"
3# "proto" => (integer) 3
...
127.0.0.1:6379> HGETALL h
1# "f" => "v"
127.0.0.1:6379> ZSCORE z m
(double) 1.5
```

You can also start in RESP3 with `redis-cli -3`. Other things worth knowing at the prompt:

- `HELP SET` prints the syntax, and the prompt shows grey hints as you type. In Redis 8.10 the `SET`
  hint already includes newer options like `IFEQ`.
- `SELECT 2` switches to logical database 2, and the prompt becomes `127.0.0.1:6379[2]>`.
- After `MULTI`, the prompt shows `(TX)` until `EXEC` or `DISCARD`.
- Command names are case-insensitive. Key names are not.

## redis-cli in scripts

Give `redis-cli` a command as arguments and it runs it and exits. When stdout is not a terminal it switches to
**raw** output, with no quotes, no `1)` numbering and no `(nil)`:

```bash
redis-cli RPUSH q a b c
redis-cli LRANGE q 0 -1
redis-cli --no-raw LRANGE q 0 -1
echo "[$(redis-cli GET nothere)]"
```

```text
3
a
b
c
1) "a"
2) "b"
3) "c"
[]
```

In raw mode a missing key and an empty string both print as an empty line, so a shell script can't tell them
apart. If the difference matters, check `EXISTS` first or use `--no-raw`. For machine-readable output there's
`--json`:

```bash
redis-cli HSET user:1 name Ana city Porto
redis-cli --json HGETALL user:1
```

```text
{"name":"Ana","city":"Porto"}
```

**Reading a value from stdin** with `-x` makes the last argument come from stdin, byte for byte:

```bash
printf 'line one\nline two\n' > note.txt
redis-cli -x SET note < note.txt
redis-cli --no-raw GET note
```

```text
OK
"line one\nline two\n"
```

**Databases.** A server has 16 numbered databases by default. `-n` picks one:

```bash
redis-cli -n 2 SET only-in-db2 yes
redis-cli GET only-in-db2
redis-cli -n 2 GET only-in-db2
redis-cli INFO keyspace
```

```text
OK

yes
# Keyspace
db0:keys=3,expires=0,avg_ttl=0,subexpiry=0
db2:keys=1,expires=0,avg_ttl=0,subexpiry=0
```

Numbered databases are namespaces, not isolation. They share one memory limit, one thread and one set of users,
and Redis Cluster supports only database 0. Prefer key prefixes (`app1:…`) or separate servers.

**Repeating a command** uses `-r` (count) and `-i` (interval in seconds):

```bash
redis-cli -r 3 -i 0.5 INCR tick
```

```text
1
2
3
```

## Round trips are the cost: three ways to load 100,000 keys

Here is a file of 100,000 `SET product:N value` commands, loaded three ways:

```bash
python3 -c "
for i in range(100000): print(f'SET product:{i} {i*3}')" > cmds.txt
```

**One `redis-cli` process per command** (only 2,000 of them, because the full set would take minutes):

```bash
time (head -2000 cmds.txt | while read c k v; do redis-cli SET $k $v >/dev/null; done)
```

```text
5.84s user 3.11s system 77% cpu 11.480 total
```

**One `redis-cli` process reading all 100,000 commands from stdin.** It sends one command, waits for the
reply, then sends the next:

```bash
time (redis-cli < cmds.txt > /dev/null)
```

```text
0.33s user 0.65s system 11% cpu 8.192 total
```

**`--pipe`**, which streams the commands without waiting for replies. It needs input already encoded as RESP:

```python
import sys
for i in range(100000):
    parts = [b"SET", f"product:{i}".encode(), str(i * 3).encode()]
    sys.stdout.buffer.write(b"*%d\r\n" % len(parts) +
        b"".join(b"$%d\r\n%s\r\n" % (len(p), p) for p in parts))
```

```bash
python3 gen_resp.py > cmds.resp
time (redis-cli --pipe < cmds.resp)
```

```text
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 100000
0.02s user 0.02s system 59% cpu 0.079 total
```

That's **8.2 seconds versus 0.079 seconds** for the same 100,000 writes, roughly a hundredfold, and the server
did identical work in both cases. The difference is the time spent waiting: one network round trip per command
versus a continuous stream. On a real network, where a round trip is often 0.5–1 ms rather than tens of
microseconds over loopback, the gap is larger. (Starting a process per command is slower still: 11.5 s for just
2,000 keys.) Every client library supports pipelining. Use it whenever you have more than a handful of
independent commands to send.

## Watching and inspecting a live server

**`MONITOR`** streams every command the server executes. Here it captures a short redis-py session:

```bash
redis-cli MONITOR
```

```text
OK
1791854878.277435 [0 [::1]:60703] "HELLO" "3"
1791854878.277941 [0 [::1]:60703] "CLIENT" "SETINFO" "LIB-NAME" "redis-py"
1791854878.278115 [0 [::1]:60703] "CLIENT" "SETINFO" "LIB-VER" "8.1.0"
1791854878.278321 [0 [::1]:60703] "SET" "session:9" "x" "EX" "60"
1791854878.278565 [0 [::1]:60703] "GET" "product:7"
1791854878.278817 [0 [::1]:60703] "INCRBY" "a" "1"
1791854878.278823 [0 [::1]:60703] "INCRBY" "b" "1"
```

There are three useful findings in seven lines. The library opened with `HELLO 3`, so it negotiated RESP3. It
labelled itself with `CLIENT SETINFO`, which makes it identifiable in `CLIENT LIST`. And its `incr()` method
actually sends `INCRBY key 1`. The last two `INCRBY` calls were pipelined, 6 µs apart. `MONITOR` is a debugging
tool, though. It costs the server real throughput, and it shows every argument, passwords and personal data
included. Don't leave it running in production.

**`--bigkeys`** scans the keyspace (with `SCAN`, so it doesn't block like `KEYS`) and reports the largest key of
each type:

```bash
redis-cli --bigkeys
```

```text
-------- summary -------

Sampled 100003 keys in the keyspace!
Total key length in bytes is 1288901 (avg len 12.89)

Biggest string found "product:80350" has 6 bytes

0 lists with 0 items (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
100003 strings with 562963 bytes (100.00% of keys, avg size 5.63)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

`--memkeys` does the same ranked by `MEMORY USAGE`, `--stat` prints a rolling one-line summary every second, and
`--latency` measures `PING` round trips continuously. Level 4 · 08 uses all of them for troubleshooting.

## How It Actually Works

Server-side, every connection has a **query buffer**. The event loop reads whatever bytes the socket has into
that buffer, and the parser pulls out as many *complete* commands as it can, leaving any partial command in
place until more bytes arrive. Then it executes those commands back to back and appends each reply to the
client's **output buffer**. That's why pipelining works without any special server mode. Redis never needed
the first reply to be read before it processed the second command; the client was simply choosing to wait.
`--pipe` and library pipelines stop waiting.

The prefix-and-length design is what makes the parser fast. To read a bulk string, the server reads the
`$<len>` line and then copies exactly `len` bytes. It never scans the payload for a terminator and never
unescapes anything. Replies work the same way on the client side.

RESP3 doesn't change command semantics. The server keeps a per-connection protocol version (set by `HELLO`)
and serializes the *same* internal result differently: a hash reply becomes a `%` map instead of a `*` array, a
score becomes a `,` double instead of a `$` string. Push messages (`>`) are tagged so that a client can pull them
out of the reply stream and route them to a callback. That's what lets one RESP3 connection receive Pub/Sub
messages and invalidation notices while still running ordinary commands. RESP2 needs a dedicated connection
for that.

`MONITOR` works by registering the connection as a monitor. After each command executes, the server formats it
and appends it to every monitor's output buffer. With many monitors or a busy server, that formatting work
happens on the one command thread, which is why it's expensive.

## Common mistakes

- **One round trip per command in a loop.** It's the most common reason for "Redis is slow". The server is
  idle and the client is waiting. Pipeline the loop, or use a multi-key command like `MSET`.
- **Parsing raw `redis-cli` output in scripts** and treating an empty line as "missing". Raw mode can't tell
  `(nil)` from `""`. Use `EXISTS`, `--no-raw` or `--json`.
- **Building `--pipe` input as inline text** with values that contain spaces or newlines. Generate real RESP,
  as shown above.
- **Leaving `MONITOR` attached** to a production server, or pasting its output, which contains every value and
  any `AUTH` arguments, into a ticket.
- **Using numbered databases for tenant isolation.** They share everything, and Cluster only has database 0.
- **Assuming the reply type.** A score is a string in RESP2 and a double in RESP3. Check what your client
  negotiated (`CLIENT LIST` shows `resp=2` or `resp=3`).

## Exercise

1. With `nc 127.0.0.1 6379` (or the Python socket script), send `SET bin "a\r\nb"` as a properly framed RESP
   array, then `STRLEN bin`. Why does the length come back as 4?
2. Send `HELLO 3` followed by `HGETALL user:1` and `SISMEMBER s x`, and identify every RESP3 type prefix in the
   raw reply.
3. Regenerate `cmds.resp` with 1,000,000 commands and time `--pipe`. Then time the same load with redis-py, once
   with a plain loop of `r.set()` and once with a pipeline that sends batches of 1,000. Work out the commands
   per second for each.
4. Run `MONITOR` in one terminal and a script that uses your favourite client library in another. What does the
   library send when it connects? Does it send `HELLO`, `CLIENT SETINFO` or `SELECT`?
