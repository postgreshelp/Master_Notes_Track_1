# PostgreSQL Connections, Processes & pgBouncer — Session Notes

This document captures key concepts from a live lab session covering PostgreSQL backend process architecture, connection management, pg_hba.conf rules, process states, timeout controls, safe process termination techniques, and a ConnectLab live demo showing HikariCP pool behavior under load. Each section includes actual terminal output observed during the session, followed by all questions discussed.

---

## Table of Contents

- [1. PostgreSQL Process Architecture](#1-postgresql-process-architecture)
- [2. pg\_hba.conf — Connection Filtering](#2-pg_hbaconf--connection-filtering)
- [3. kill -9 — Why It's a Crime](#3-kill--9--why-its-a-crime)
- [4. Safe Ways to Terminate Sessions](#4-safe-ways-to-terminate-sessions)
- [5. Backend Process States](#5-backend-process-states)
- [6. Session & Statement Timeout Controls](#6-session--statement-timeout-controls)
- [7. ConnectLab Live Demo — HikariCP Pool Behavior Under Load](#7-connectlab-live-demo--hikarcp-pool-behavior-under-load)
- [8. Summary of Error / Termination Messages](#8-summary-of-error--termination-messages)
- [9. Questions Discussed in This Session](#9-questions-discussed-in-this-session)

---

## 1. PostgreSQL Process Architecture

Every PostgreSQL instance runs a **postmaster** process that manages all background workers and spawns a new backend process for each client connection. With 100 direct connections, you get 100 backend processes.

```
/usr/pgsql-18/bin/postgres -D /u01/pgsql/18   ← postmaster (PID 5651)
```

Background processes spawned by the postmaster:

```
postgres    5652    5651   postgres: logger
postgres    5653    5651   postgres: io worker 0
postgres    5654    5651   postgres: io worker 1
postgres    5655    5651   postgres: io worker 2
postgres    5656    5651   postgres: checkpointer
postgres    5657    5651   postgres: background writer
postgres    5659    5651   postgres: walwriter
postgres    5660    5651   postgres: autovacuum launcher
postgres    5661    5651   postgres: logical replication launcher
```

Each background process has the postmaster as its parent (PPID = 5651).

---

## 2. pg_hba.conf — Connection Filtering

`pg_hba.conf` rules are evaluated **top to bottom**. The first matching rule wins.

```
# TYPE  DATABASE    USER   ADDRESS      METHOD
host    connectlab  all    0.0.0.0/0    reject
host    all         all    0.0.0.0/0    md5
```

Attempting to connect to `connectlab` from an external IP hits the first rule and is rejected before even reaching authentication:

```
psql: error: connection to server at "192.168.44.129", port 5432 failed:
FATAL:  pg_hba.conf rejects connection for host "192.168.44.129",
        user "postgres", database "connectlab", no encryption
```

The second rule (`md5` for all databases) never gets evaluated for `connectlab` because the `reject` rule matched first.

---

## 3. `kill -9` on a PostgreSQL Background Process — Why It's a Crime

Sending `SIGKILL` directly to any PostgreSQL background process bypasses PostgreSQL's internal shutdown protocol. The postmaster detects the abnormal exit, terminates all other active backends, and triggers crash recovery.

**Experiment: killing the background writer (PID 5657)**

```bash
kill -9 5657
```

**Postmaster log response:**

```
LOG:  background writer process (PID 5657) was terminated by signal 9: Killed
LOG:  terminating any other active server processes
LOG:  all server processes terminated; reinitializing
LOG:  database system was interrupted; last known up at 2026-03-20 07:23:37 IST
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  redo starts at 2/6DF1F260
LOG:  redo done at 2/6DF37CF8
LOG:  checkpoint complete: ...
LOG:  database system is ready to accept connections
```

Any session that was active at the time of the kill receives this warning and is terminated:

```
WARNING:  terminating connection because of crash of another server process
DETAIL:  The postmaster has commanded this server process to roll back the
         current transaction and exit, because another server process exited
         abnormally and possibly corrupted shared memory.
```

> ⚠️ PostgreSQL recovered automatically here, but in production this causes in-flight transaction rollbacks, potential shared memory corruption, and a full crash recovery cycle. **Never `kill -9` a PostgreSQL process.**

---

## 4. Safe Ways to Terminate Sessions

PostgreSQL provides two SQL functions for gracefully managing sessions without touching the OS.

### `pg_cancel_backend(pid)` — Cancel the current query, keep the connection

```sql
SELECT pg_cancel_backend('9948');
-- t

-- On the target session:
ERROR:  canceling statement due to user request
```

The session stays connected. Only the running query is cancelled.

### `pg_terminate_backend(pid)` — Disconnect the session entirely

```sql
SELECT pg_terminate_backend('9948');
-- t

-- On the target session:
FATAL:  terminating connection due to administrator command
-- Connection lost. Attempting reset: Succeeded.
```

The session is disconnected but PostgreSQL reconnects the client automatically (in `psql`). No crash recovery is triggered.

---

## 5. Backend Process States

Every backend visible in `ps -ef | grep postgres` is in one of these states (also visible in `pg_stat_activity`):

| State | Meaning |
|---|---|
| `idle` | Connected, not executing anything |
| `idle in transaction` | Inside an open transaction, waiting for next command |
| `active` | Currently executing a query |

---

## 6. Session & Statement Timeout Controls

All four timeout parameters default to `0` (disabled). They are set in `postgresql.conf` or at the session level.

```ini
statement_timeout                   = 0   # Kill query after N ms
transaction_timeout                 = 0   # Kill transaction after N ms
idle_in_transaction_session_timeout = 0   # Kill idle-in-transaction sessions
idle_session_timeout                = 0   # Kill idle sessions
```

### `idle_session_timeout` — Kills sessions sitting idle too long

```ini
idle_session_timeout = 5s
```

```
FATAL:  terminating connection due to idle-session timeout
-- Connection lost. Attempting reset: Succeeded.
```

### `idle_in_transaction_session_timeout` — Kills sessions holding open transactions

```ini
idle_in_transaction_session_timeout = 10s
```

```sql
UPDATE emp SET sal=200 WHERE id=1;
-- UPDATE 1
-- (10 seconds pass without COMMIT)
FATAL:  terminating connection due to idle-in-transaction timeout
```

> ⚠️ This is critical in production — an open transaction holds row-level locks and prevents autovacuum from cleaning dead tuples on affected tables.

---

## 7. ConnectLab Live Demo — HikariCP Pool Behavior Under Load

This section documents a live demo using **ConnectLab** (`connectlab-1.0.0.jar`) — a Spring Boot application that visualises HikariCP connection pool behaviour in real time. The goal was to prove, using `ps -ef`, exactly what PostgreSQL sees when pool pressure changes.

### 7.1 Pool Configuration (Pool Settings Tuner)

ConnectLab supports **live tuning** via HikariCP's MXBean interface — pool parameters take effect immediately without restarting the application.

| Parameter | Value |
|---|---|
| `maximum-pool-size` | **10** |
| `minimum-idle` | 2 |
| `connection-timeout` | 20000 ms |
| `idle-timeout` | 30000 ms |
| `max-lifetime` | 1800000 ms |
| `leak-detection-threshold` | 15000 ms |

---
![Pool Settings Tuner](images/6.jpg)

### 7.2 Step 1 — Baseline: App Started, No Load

```bash
ps -ef | grep postgres
```

```
postgres   35533       1   0 11:21 ?   /usr/pgsql-18/bin/postgres -D /u01/pgsql/18
postgres   35534   35533   0 11:21 ?   postgres: logger
postgres   35535   35533   0 11:21 ?   postgres: io worker 0
postgres   35536   35533   0 11:21 ?   postgres: io worker 1
postgres   35537   35533   0 11:21 ?   postgres: io worker 2
postgres   35538   35533   0 11:21 ?   postgres: checkpointer
postgres   35539   35533   0 11:21 ?   postgres: background writer
postgres   35541   35533   0 11:21 ?   postgres: walwriter
postgres   35542   35533   0 11:21 ?   postgres: autovacuum launcher
postgres   35543   35533   0 11:21 ?   postgres: logical replication launcher
```

Only background workers visible. No client backend processes — HikariCP has not yet opened its minimum-idle connections.

---

### 7.3 Step 2 — Light Spike: 5 Concurrent Connections

**Spike Simulator fired with 5 connections, hold time 5 seconds, query: `SELECT pg_sleep(n)`**

**Live Pool State:**

| Metric | Value |
|---|---|
| ACTIVE | **5** |
| IDLE | 0 |
| WAITING | 0 |
| TOTAL | 5 |
| Pool Pressure | **50%** |

Pool is half utilised. No queuing. All 5 requests reached PostgreSQL immediately.

---

### 7.4 Step 3 — Heavy Spike: 20 Connections Against pool_size=10

**Spike Simulator fired with 20 connections, same hold time and query.**

**Live Pool State:**

| Metric | Value |
|---|---|
| ACTIVE | **10** |
| IDLE | 0 |
| WAITING | **10** |
| TOTAL | 10 |
| Pool Pressure | **100%** 🔴 |

Pool is fully saturated. The 10 excess threads are **blocked inside HikariCP** waiting for a connection to free up — PostgreSQL never saw them.

**`ps -ef` captured during this spike:**

```
postgres   35342    3686  15 11:19 pts/1   java -jar connectlab-1.0.0.jar

postgres   35649   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55728) SELECT
postgres   35650   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55738) SELECT
postgres   35651   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55750) SELECT
postgres   35669   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(52918) SELECT
postgres   35670   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(52930) SELECT
postgres   35674   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57846) SELECT
postgres   35675   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57860) SELECT
postgres   35704   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57862) SELECT
postgres   35710   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57864) SELECT
postgres   35711   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57870) SELECT
```

**Exactly 10 backend processes** — matching `maximum-pool-size`. The other 10 threads waiting in HikariCP are invisible to PostgreSQL entirely.

---

### 7.5 Step 4 — After Spike Settles

```
postgres   35649   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55728) idle
postgres   35650   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55738) idle
postgres   35651   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(55750) idle
postgres   35669   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(52918) idle
postgres   35670   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(52930) SELECT
postgres   35674   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57846) idle
postgres   35675   35533   0 11:22 ?   postgres: postgres connectlab 127.0.0.1(57860) idle
```

Load subsided. Same 10 backends, most back to `idle` — HikariCP is holding them open (minimum-idle pool behaviour). One backend still shows `SELECT` mid-execution. No new processes were created, none were destroyed.

---

### 7.6 Key Takeaway

> **PostgreSQL saw a maximum of 10 backend processes regardless of 20 concurrent requests.**
> HikariCP absorbed the overflow silently. The 10 waiting threads never appeared in `ps`.

```
20 app threads
    │
    ├── 10 → HikariCP pool → 10 PostgreSQL backends  (visible in ps)
    └── 10 → blocked inside HikariCP                 (invisible to PostgreSQL)
```

This is the direct, observable proof of why connection pooling exists.

---

## 8. Summary of Error / Termination Messages

| Message | Trigger |
|---|---|
| `ERROR: canceling statement due to user request` | `pg_cancel_backend()` |
| `FATAL: terminating connection due to administrator command` | `pg_terminate_backend()` |
| `WARNING: terminating connection because of crash of another server process` | `kill -9` on any backend |
| `FATAL: terminating connection due to idle-session timeout` | `idle_session_timeout` exceeded |
| `FATAL: terminating connection due to idle-in-transaction timeout` | `idle_in_transaction_session_timeout` exceeded |

---

## 9. Questions Discussed in This Session

**Q1. Every session creates a new process — so 100 connections = 100 processes in `ps`?**
Yes. PostgreSQL uses a process-per-connection model. Each client connection forks a new backend process from the postmaster. This is why connection pooling exists.

**Q2. If pgBouncer is in session mode, does `ps` show `pool_size` backends forever?**
Not forever — they persist as long as clients are connected or until `server_idle_timeout` elapses after a client disconnects. `pool_size` is the ceiling; you will never see more backends than that value regardless of how many clients connect.

**Q3. Those 10 idle backends in `ps` — will they go away on their own?**
Only if `idle_session_timeout` is set. With the default of `0`, they sit indefinitely as long as the client (e.g., HikariCP pool) holds the connection open. TCP keepalive (`tcp_keepalive_time = 7200s` by default) only helps with silently dead clients, not live idle ones.

**Q4. What does `tcp_keepalives_idle = 0` mean in PostgreSQL?**
It means PostgreSQL defers to the OS default, which on Linux is `7200` seconds (`cat /proc/sys/net/ipv4/tcp_keepalive_time`). A dead client won't be detected for approximately 2 hours 11 minutes under default settings.

**Q5. Is `kill -9` on a PostgreSQL process safe?**
No — it is the one thing you should never do. PostgreSQL treats any abnormal process exit as a potential shared memory corruption event, terminates all active sessions, and triggers crash recovery. Use `pg_terminate_backend()` from SQL instead.

**Q6. What is the difference between `pg_cancel_backend()` and `pg_terminate_backend()`?**
`pg_cancel_backend()` sends `SIGINT` — cancels the current query but leaves the connection alive. `pg_terminate_backend()` sends `SIGTERM` — disconnects the session entirely. In production, prefer cancel first; terminate only when the session must be removed.

**Q7. What is the `idle in transaction` state and why is it dangerous?**
A session is `idle in transaction` when it has started a transaction (`BEGIN`) and issued at least one statement but has not yet committed or rolled back. This state holds row-level locks and blocks autovacuum from reclaiming dead tuples on affected tables. `idle_in_transaction_session_timeout` is the safety net for this.

**Q8. If HikariCP pool is set to 20, does the 21st request get blocked at the client level?**
Yes — exactly. The 21st thread blocks inside HikariCP waiting for a connection to free up, controlled by `connectionTimeout` (default 30 seconds). PostgreSQL never sees it at all. The ConnectLab spike demo with 20 requests against `pool_size=10` proved this live — `ps` showed exactly 10 backends while 10 threads waited silently inside Java.

**Q9. Does having 50 idle sessions sitting in `ps` hurt performance?**
No — memory is approximately 1.9 MB per backend, so 50 idle sessions is around 95 MB total, which is negligible. The only secondary effect is that each backend occupies a slot in PostgreSQL's ProcArray, which is scanned during snapshot acquisition for MVCC. At 50 backends this overhead is immeasurable; it only becomes relevant above ~500 connections.

**Q10. With pgBouncer `pool_size=10` and two databases configured, do I get 10 shared connections or 20 total?**
20 total. `pool_size` applies **per (database, user) pair**. Each unique combination gets its own independent pool. Verify live with:
```sql
psql -p 6432 pgbouncer -c "SHOW POOLS"
```

**Q11. pgbench showed TPS doubled and latency halved through pgBouncer, but initial connection time was higher — is that expected?**
Yes, completely expected. pgBouncer accepted 800 client connections and handled auth negotiation for each, which adds overhead to the initial connection phase. But actual query execution was far more efficient because only `pool_size` backends were doing real work. In production, connections are established once and reused, so the higher initial connection time is a one-time cost that does not matter.
