---
description: "Securing Redis: protected mode, least-privilege ACL users, ACL DRYRUN and ACL LOG, and mutual TLS with certificates Python 3.14 accepts."
---

# 07 · Security: ACLs, Protected Mode & TLS

A default Redis server trusts anyone who can open a TCP connection to it, with full power to read every key, delete
everything, rewrite its configuration and load modules. Redis servers exposed to the internet are routinely found and
abused. This lesson closes the doors in layers: network exposure, authentication, per-user permissions and encryption
in transit.

## Layer 1: who can connect at all

A server started with no configuration listens on all interfaces (`bind * -::*`), but **protected mode** is on. With
no password set for the default user and no explicit `bind`, it accepts connections only from loopback. Connecting
from the machine's LAN address (`192.168.97.73`) to a server on port 6490:

```text
DENIED Redis is running in protected mode because protected mode is enabled and no password is set for the default
user. In this mode connections are only accepted from the loopback interface. If you want to connect from external
computers to Redis you may adopt one of the following solutions: 1) Just disable protected mode … 4) Set up an
authentication password for the default user. …
```

That's a safety net, not a security model. The moment you set a `bind` address or a password, protected mode steps
aside and your configuration is all that stands between the server and the network. With `--bind 127.0.0.1`, the LAN
address doesn't even accept the TCP connection:

```text
Could not connect to Redis at 192.168.97.73:6490: setsockopt(TCP_NODELAY): Invalid argument
```

(That odd error text is how this build of `redis-cli` on macOS reports a refused connection.) In production: bind to
private interfaces only, firewall the port (and the cluster bus port, +10000) to the hosts that need it, and never
expose Redis to the internet, even with a password.

## Layer 2: users and permissions (ACLs)

Since Redis 6, authentication is per **user**, and each user has rules about which commands, keys and Pub/Sub channels
it may use. A fresh server has one user:

```text
127.0.0.1:6490> ACL LIST
1) "user default on nopass sanitize-payload ~* &* +@all"
```

`nopass` with `+@all` on `~*` (all keys) and `&*` (all channels) means anyone can do anything. Create users per
application, by the principle of least privilege:

```text
127.0.0.1:6490> ACL SETUSER cache-app on >s3cret-cache-pw ~cache:* resetchannels -@all +@read +@write +@keyspace -@dangerous
OK
127.0.0.1:6490> ACL SETUSER reporter on >s3cret-report-pw ~* resetchannels -@all +@read -@slow +info|stats
OK
127.0.0.1:6490> ACL SETUSER admin on >s3cret-admin-pw ~* &* +@all
OK
127.0.0.1:6490> ACL SETUSER default off
OK
```

Reading the rules:

- `on` / `off` enables or disables the user. `>password` adds a password, which is stored as a SHA-256 hash, as
  `ACL LIST` shows (`#9a3039b1…`).
- `~cache:*` limits **keys** to that pattern. `resetchannels` removes all Pub/Sub channel access.
- `-@all +@read +@write +@keyspace -@dangerous` starts from nothing, then adds **command categories**, then removes the
  dangerous ones. `ACL CAT` lists the categories, and `ACL CAT dangerous` lists 60 commands on this version, including
  `flushdb`, `keys`, `save`, `debug`, `module|load`, `client|kill`, `acl|setuser`, `replicaof` and `info`.
- `+info|stats` allows one **subcommand argument** (`INFO stats`) and nothing else from `INFO`.
- Disabling `default` means unauthenticated clients can do nothing.

Before rolling rules out, check them without running anything with **`ACL DRYRUN`**:

```text
127.0.0.1:6490> ACL DRYRUN cache-app SET cache:product:1 x
OK
127.0.0.1:6490> ACL DRYRUN cache-app SET session:1 x
"User cache-app has no permissions to access the 'session:1' key"
127.0.0.1:6490> ACL DRYRUN cache-app FLUSHALL
"User cache-app has no permissions to run the 'flushall' command"
127.0.0.1:6490> ACL DRYRUN reporter KEYS *
"User reporter has no permissions to run the 'keys' command"
127.0.0.1:6490> ACL DRYRUN reporter INFO stats
OK
127.0.0.1:6490> ACL DRYRUN reporter INFO server
"User reporter has no permissions to run the 'info' command"
```

Then as the application:

```text
$ redis-cli -p 6490 GET x
NOAUTH Authentication required.

127.0.0.1:6490> AUTH cache-app wrong-password
(error) WRONGPASS invalid username-password pair or user is disabled.
127.0.0.1:6490> AUTH cache-app s3cret-cache-pw
OK
127.0.0.1:6490> ACL WHOAMI
(error) NOPERM User cache-app has no permissions to run the 'acl|whoami' command
127.0.0.1:6490> SET cache:product:1 widget
OK
127.0.0.1:6490> GET session:42
(error) NOPERM No permissions to access a key
127.0.0.1:6490> FLUSHALL
(error) NOPERM User cache-app has no permissions to run the 'flushall' command
127.0.0.1:6490> CONFIG SET maxmemory 1mb
(error) NOPERM User cache-app has no permissions to run the 'config|set' command
```

Note `ACL WHOAMI` being refused. It's in the `@connection` category, which `-@all` removed. Start-from-nothing rules
need care: client libraries may run `HELLO`, `CLIENT SETINFO`, `SELECT` or `PING` when they connect. Add `+@connection`
(or the specific commands) if your library fails to connect with a narrowly scoped user.

Every denial is recorded in **`ACL LOG`** (here read as `admin`), with the user, the command or key, and the client's
address. It's the first place to look when an application breaks after an ACL change, and a good thing to alert on:

```text
1)  1) "count"
    2) (integer) 1
    3) "reason"
    4) "command"
    5) "context"
    6) "toplevel"
    7) "object"
    8) "config|set"
    9) "username"
   10) "cache-app"
   ...
   14) "client-info"
       "id=6 addr=127.0.0.1:57076 … cmd=config|set user=cache-app …"
```

Persist users with an **ACL file** (`aclfile /etc/redis/users.acl`, then `ACL SAVE`/`ACL LOAD`) or `user` lines in
`redis.conf`. Don't leave them only in memory, where a restart would lose them. Replicas, Sentinels and cluster nodes
need users too: `masteruser`/`masterauth` for replication, `sentinel auth-user`/`auth-pass` for Sentinel.

The old approach of `rename-command FLUSHALL ""` still exists, but ACLs replace it: they're per user, visible and
auditable.

## Layer 3: encryption in transit (TLS)

Passwords and data cross the network in clear text unless you enable TLS. The Homebrew build includes TLS support.
First, a private CA and certificates for the server and one client:

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 30 -subj "/CN=redis-lab-ca" \
  -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -addext "subjectKeyIdentifier=hash" -out ca.crt

openssl genrsa -out redis.key 2048
openssl req -new -key redis.key -subj "/CN=localhost" -out redis.csr
printf "subjectAltName=DNS:localhost,IP:127.0.0.1\nauthorityKeyIdentifier=keyid,issuer\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature,keyEncipherment\nextendedKeyUsage=serverAuth\n" > server.ext
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 30 -sha256 -extfile server.ext -out redis.crt

openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=cache-app" -out client.csr
printf "authorityKeyIdentifier=keyid,issuer\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature,keyEncipherment\nextendedKeyUsage=clientAuth\n" > client.ext
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 30 -sha256 -extfile client.ext -out client.crt
```

Those extension lines matter. **The first version of these certificates was minimal**, and `redis-cli` accepted it,
but Python 3.14 rejected it twice:

```text
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: Missing Authority Key Identifier (_ssl.c:1082)
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: CA cert does not include key usage extension (_ssl.c:1082)
```

Recent Python versions verify certificates in OpenSSL's **strict X.509 mode** by default, which requires extensions
that many quick-start tutorials leave out. If one client connects and another doesn't, compare their TLS
libraries' strictness before blaming the server.

Start a TLS-only server (`--port 0` disables the plain-text port):

```bash
redis-server --port 0 --tls-port 6491 --tls-cert-file redis.crt --tls-key-file redis.key --tls-ca-cert-file ca.crt
```

```text
* Ready to accept connections tls
```

```text
$ redis-cli -p 6491 PING
Error: Connection reset by peer
$ redis-cli --tls --cacert ca.crt -p 6491 PING
Error: Server closed the connection
$ redis-cli --tls --cacert ca.crt --cert client.crt --key client.key -p 6491 PING
PONG
```

A plain-text client is refused. A TLS client **without a client certificate** is refused too, because
`tls-auth-clients` defaults to `yes` (mutual TLS). Set it to `optional` or `no` if clients should authenticate with
ACL passwords only. From Python:

```python
r = redis.Redis(port=6491, ssl=True, ssl_ca_certs="ca.crt", ssl_certfile="client.crt", ssl_keyfile="client.key")
print("python over TLS:", r.ping())
```

```text
python over TLS: True
no client cert: ConnectionError Error while reading from localhost:6491 : (1, '[SSL: TLSV13_ALERT_CERTIFICATE_REQUIRED] tlsv13 alert certifica…
```

TLS has a cost. `redis-benchmark` with 50 clients doing `GET`:

```text
GET plain: 164203.61 req/s  TLS: 121654.50 req/s
```

That's about 26% less throughput on this laptop, from encryption and the TLS record layer on Redis's main thread.
Long-lived pooled connections avoid repeating the handshake. Use TLS whenever traffic crosses a network you don't fully
control, and budget for the cost. Replication and cluster bus traffic can be encrypted too (`tls-replication yes`,
`tls-cluster yes`).

## How It Actually Works

On every command, Redis checks the connection's authenticated user. First, is the command (or the subcommand, or for
rules like `+info|stats`, the first argument) allowed by the user's command bitmap? Then the command's **key
specifications** (metadata saying which arguments are keys, and whether each is read or written) are checked against
the user's key patterns. Patterns can be split as `%R~` (read) and `%W~` (write) for finer control. That's why ACLs
need no per-command special cases, and why scripts and Functions must declare their keys (Level 2 · 09): the server
checks `KEYS` before running them, and checks again on every `redis.call`.

Passwords are stored only as SHA-256 hashes, so `ACL LIST` doesn't reveal them. Denials go into a bounded in-memory
log (`acllog-max-len`).

TLS is implemented with OpenSSL inside the connection layer. After the handshake, the same RESP bytes flow over an
encrypted channel. `tls-auth-clients yes` makes the server request and verify a client certificate against
`tls-ca-cert-file` during the handshake, before any command can be sent. By default the certificate's identity isn't
mapped to an ACL user. This 8.10 build also has a `tls-auth-clients-user` setting (`off` by default) for
authenticating clients as an ACL user derived from their certificate, but it wasn't tested for this lesson. Usually
mTLS and ACL passwords are combined.

## Common mistakes

- **Exposing Redis to the internet.** Bind privately and firewall it, password or not.
- **Only setting `requirepass`.** Every client then shares one all-powerful identity. Use ACL users per application.
- **Leaving `default` on with `nopass`** after creating users.
- **`+@all` for applications.** Grant categories and key patterns, and remove `@dangerous`.
- **Users only in memory.** Use an ACL file and `ACL SAVE`.
- **Minimal certificates** that one TLS stack accepts and another (Python 3.13+) rejects. Include proper extensions.
- **Plain-text replication** across networks when client traffic is encrypted.
- **Ignoring `ACL LOG`.** Denials are often the first sign of a misconfiguration or an intrusion attempt.

## Exercise

1. Create a `queue-worker` user that can only use streams under `jobs:*` (`XREADGROUP`, `XACK`, `XAUTOCLAIM`,
   `XADD`), and verify with `ACL DRYRUN` that it can't read `cache:*` keys or run `FLUSHDB`.
2. Give `reporter` read-only access to `cache:*` via `%R~cache:*` and write access to `reports:*` only. Test both
   directions.
3. Run a primary and replica where replication requires a dedicated `replicator` user. Which permissions does that user
   need? Use `ACL LOG` on the primary to find out by trial and error.
4. Measure TLS overhead with `redis-benchmark` at 1, 10 and 100 clients and with pipelining (`-P 16`). When is the
   overhead smallest?
