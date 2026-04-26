If you love self-hosting your own database, or you are naturally curious, you will enjoy this piece. In today's write-up, we will discuss how to scale PostgreSQL throughput with PgBouncer.

My name is Ilori Stephen Adejuwon, and I am a software engineer who loves self-hosting, and all the pain that comes with it.

## The Problem

PostgreSQL uses a process-per-connection model. This means every client connection gets a dedicated backend process to handle that session's lifecycle. This design has benefits: process isolation reduces blast radius if one backend fails.

So what is the trade-off? Creating and managing a large number of processes is expensive, and there is a hard ceiling to how many useful concurrent connections you can run before context switching and memory pressure become painful.

## The Solution

Before PgBouncer, there were several attempts to reduce this overhead, and they usually looked like this:

- **Client-side pooling:** In a microservices setup, if you run `N` app instances each with `POOL_SIZE`, your database sees roughly `N * POOL_SIZE` potential connections.
- **Application-level multiplexing:** I used to work at a startup where all DB requests routed through a single internal service. It worked, but it added another component that could become a bottleneck.
- **Pgpool-II:** Powerful, but operationally heavier. Many teams find PgBouncer simpler when the primary goal is lightweight connection pooling.

Among these, one tool stood out: PgBouncer. It became the de facto standard for lightweight PostgreSQL connection pooling, and it complements PostgreSQL really good too.

## What Is PgBouncer?

PgBouncer is a lightweight connection pooler for PostgreSQL. It sits in front of your database, accepts many client connections, and multiplexes them over a smaller number of server connections to PostgreSQL.

The memory footprint per client connection is small (typically only a few KB). It is not a process-per-connection or thread-per-connection model; PgBouncer is a single-process, event-loop based service (built on `libevent`).

One of my favorite features is that you can expose logical database names to clients, then map those names to physical PostgreSQL databases in the PgBouncer config.

## Installing PgBouncer

There are several ways to install PgBouncer. One of the easiest is installing from the PostgreSQL APT repository on Debian/Ubuntu. It is usually more convenient than building from source.

```bash
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install -y wget gnupg2 lsb-release ca-certificates
$ echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
| sudo tee /etc/apt/sources.list.d/pgdg.list
$ wget -qO - https://www.postgresql.org/media/keys/ACCC4CF8.asc \
| sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
$ sudo apt update && sudo apt install -y pgbouncer
```

If the steps above succeed, PgBouncer should be installed. You can verify with:

```bash
$ pgbouncer --version
```

## Configuring PgBouncer

Configuring PgBouncer is straightforward when you install from your distribution package repository. I did not find source installs as smooth, so yes, this is my bias toward packaged installs.

The main files to care about are usually in `/etc/pgbouncer`:

- `userlist.txt`: Stores users allowed to authenticate through PgBouncer.

```bash
"username" "password"
```

The second field can be plaintext, an MD5 hash (`md5...`), or a SCRAM secret (depending on your `auth_type` and setup). Keep this file protected with strict permissions.

- `pgbouncer.ini`: Main PgBouncer configuration. You can either edit it directly or keep local overrides in a separate file and include them.

## Creating Your Own PgBouncer Configuration

```bash
[databases]
# Map logical name `wallet_db` used by clients to the real Postgres database `wallets`.
wallet_db = host=127.0.0.1 port=5432 dbname=wallets user=diablo
# Map logical name `transfer_db` used by clients to the real Postgres database `transfers`.
transfer_db = host=127.0.0.1 port=5432 dbname=transfers user=diablo

[pgbouncer]
# Network interface PgBouncer listens on; `0.0.0.0` allows remote clients.
listen_addr = 0.0.0.0
# TCP port clients use to connect to PgBouncer.
listen_port = 6432

# Authentication
# Authentication method PgBouncer uses when validating users.
auth_type = scram-sha-256
# File containing allowed users and their auth secrets.
auth_file = /etc/pgbouncer/userlist.txt

# Pool Mode
# Reuse server connections at transaction boundaries (good default for stateless apps).
pool_mode = transaction

# Max Connections: Client -> PgBouncer
# Maximum concurrent client connections accepted by PgBouncer.
max_client_conn = 200
# Default number of server connections kept per database/user pool.
default_pool_size = 20
# Minimum number of server connections PgBouncer tries to keep ready.
min_pool_size = 5
# Extra temporary server connections allowed when the pool is under pressure.
reserve_pool_size = 5

# Reset Queries
# Query run when a server connection is released to clean session state.
server_reset_query = DISCARD ALL
# Seconds between health checks on idle server connections.
server_check_delay = 30
# Query used to verify server connection health.
server_check_query = SELECT 1

# Max Connections: PgBouncer -> PostgreSQL
# Hard cap of server connections PgBouncer opens to one database.
max_db_connections = 40
# Hard cap of server connections per database user.
max_user_connections = 40

# Timeouts: Client -> PgBouncer
# Max seconds allowed for client authentication to complete.
client_login_timeout = 5
# Max seconds a client query can run before timeout.
query_timeout = 60
# Disconnect client if idle for this many seconds.
client_idle_timeout = 300

# Timeouts: PgBouncer -> PostgreSQL
# Wait before retrying a failed server login.
server_login_retry = 5
# Max seconds allowed to establish a connection to PostgreSQL.
server_connect_timeout = 5
# Close idle server connections after this many seconds.
server_idle_timeout = 300
# Recycle server connections after this many seconds to avoid very old sessions.
server_lifetime = 3600

# Transaction safety
# Drop clients that leave a transaction idle for too long.
idle_transaction_timeout = 30

# Logging
# Log each accepted client connection.
log_connections = 1
# Log each client disconnection.
log_disconnections = 1
# Log internal pooler errors.
log_pooler_errors = 1

# PID file location for service/process management.
pidfile = /var/run/pgbouncer/pgbouncer.pid
# PgBouncer admin-console users (for SHOW, RELOAD, PAUSE, etc.).
admin_users = pgbouncer_admin
# Users allowed to run read-only stats commands.
stats_users = pgbouncer_stats
```

Notes:

- `pool_mode = transaction` is a great default for stateless application queries.
- If your app relies on session-level features (for example temp tables or `SET` state), test carefully or use `session` pool mode.
- Ensure users listed in `admin_users` and `stats_users` exist in `userlist.txt`.

## Testing Your PgBouncer Configuration

After writing your config, do not jump straight into production. Run a quick verification checklist first.

1. Confirm the service is up.

```bash
$ sudo systemctl restart pgbouncer
$ sudo systemctl status pgbouncer --no-pager
```

2. Confirm PgBouncer is listening on the expected port.

```bash
$ ss -ltnp | rg 6432
```

3. Test a normal client connection through PgBouncer.

```bash
$ psql "host=127.0.0.1 port=6432 dbname=wallet_db user=diablo sslmode=disable"
wallet_db=> SELECT now();
wallet_db=> \q
```

4. Connect to the PgBouncer admin console and inspect pools.

```bash
$ psql "host=127.0.0.1 port=6432 dbname=pgbouncer user=pgbouncer_admin"
pgbouncer=> SHOW POOLS;
pgbouncer=> SHOW STATS;
pgbouncer=> SHOW CLIENTS;
pgbouncer=> SHOW SERVERS;
```

Things to look for while testing:

- `cl_waiting` should not keep growing forever under normal load.
- `sv_active` and `sv_idle` should fluctuate in a healthy range, not stay pinned at hard limits.
- Query latency should be stable when many clients connect at once.
- PgBouncer logs should not show recurring auth failures or pooler errors.

If you want a simple load test, use `pgbench` against port `6432` and compare it to direct Postgres traffic.

## Things to Watch Out For

### 1) Pool mode and session semantics

`pool_mode = transaction` is fast, but session state is not guaranteed to stick to one backend. Be careful if your app depends on:

- temporary tables,
- session-level `SET` commands,
- server-side prepared statements that assume session affinity.

If your application relies heavily on these patterns, validate carefully or switch to `pool_mode = session`. The transaction mode in particular broke my laravel application, and I had to switch to session mode for that instance, and this is because laravel uses prepared statements which isn't guranteed to work in the transaction mode.

### 2) Connection limit math

Make sure PgBouncer limits and PostgreSQL limits agree with each other.

- `max_client_conn` can be high (many logical clients),
- but backend limits (`max_db_connections`, `max_user_connections`) must fit under Postgres `max_connections`,
- and you still need headroom for admin access, migrations, and maintenance jobs.

### 3) Authentication mismatches

`auth_type`, values in `userlist.txt`, and PostgreSQL user credentials must be compatible. If one piece is out of sync, login failures can be confusing.

### 4) Timeouts that are too strict

Aggressive timeout values may look good on paper but can cause disconnects in real traffic spikes or under slow network conditions.

### 5) Missing observability

Always keep an eye on:

- PgBouncer logs (`journalctl -u pgbouncer -f`),
- `SHOW STATS;` and `SHOW POOLS;`,
- application error rates and connection wait times.

## Conclusion

PgBouncer is one of the simplest high-impact upgrades you can make for PostgreSQL at scale. It helps your database handle many client connections without opening one backend process per client. Deployment topology also matters: while you can run PgBouncer on a separate VM, you lose some latency and efficiency benefits. In most setups, it is best to run PgBouncer as close to PostgreSQL as possible, ideally on the same VM or host.

A practical rollout strategy is:

1. start with `pool_mode = transaction`,
2. test your application's session behavior,
3. observe pool stats under load,
4. tune limits and timeouts gradually.

If you do those four things well, you will get most of PgBouncer's benefits without the common operational surprises. I hope you enjoyed this piece as much as I enjoyed writing it. Please subscribe so you can get alerts for my next write-up.