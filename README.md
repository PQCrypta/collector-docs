# pqcrypta-collector

Async metrics collector, log ingestion engine, and intelligence layer for PQCrypta infrastructure. Single-threaded Rust binary that scrapes system, process, application, and database metrics on configurable intervals, ingests logs from 13 sources with structured parsing, writes everything to PostgreSQL with batched inserts, performs time-series aggregation and retention, and runs statistical anomaly detection with SLO tracking and actionable recommendations.

## Architecture

```
                    ┌─────────────────────────────┐
                    │         main loop            │
                    │   tokio::select! event hub   │
                    └──────┬──────────────────┬────┘
                           │                  │
         ┌─────────────────┼──────────┐───────┼──────────┐
         ▼            ▼    ▼          ▼       ▼          ▼
    sys_tick      app_tick log_tick intel_tick agg_tick watchdog
     (10s)         (10s)   (15s)    (5min)    (1hr)    (30s)
         │            │      │        │         │        │
         ▼            ▼      ▼        ▼         ▼        ▼
   ┌──────────┐ ┌────────┐ ┌─────┐ ┌────────┐ ┌──────┐ ┌─────────┐
   │  sysinfo  │ │HTTP+PG │ │logs │ │baselines│ │rollups│ │staleness│
   │  /proc    │ │scrape  │ │files│ │anomalies│ │retain │ │health   │
   └─────┬────┘ └────┬───┘ │jrnl │ │SLOs     │ │log agg│ └────┬────┘
         │           │     └──┬──┘ │recs     │ └───┬──┘      │
         │           │        │    │log anlys│     │          │
         ▼           ▼        ▼    └────┬───┘     ▼          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │              MetricWriter + LogIngester (batched)             │
   │  10 metric buffers + log batch INSERT on flush                │
   └──────────────────────────┬───────────────────────────────────┘
                              ▼
                        ┌───────────┐
                        │ PostgreSQL │
                        └───────────┘
```

### Tick intervals

| Tick | Interval | Responsibility |
|------|----------|----------------|
| `sys_tick` | 10s | CPU, memory, load, disk, network via `sysinfo` + `/proc` |
| `app_tick` | 10s | API metrics (HTTP scrape port 3003), proxy metrics (HTTP scrape port 8082), DB stats (direct pg_stat queries) |
| `heartbeat_tick` | 5s | Lightweight liveness heartbeat (single INSERT) |
| `log_tick` | 15s | Log ingestion from 13 file and journal sources, batch INSERT into `log_entries` |
| `watchdog_tick` | 30s | Staleness detection, table health, process health, API error rate, DB response time |
| `pg_extended_tick` | 5min | Per-table, per-index, IO, replication, statement stats |
| `intel_tick` | 5min | Intelligence layer: baselines, anomaly detection, SLO computation, recommendations, log pattern analysis, error spike detection, security event detection |
| `agg_tick` | 1hr | Hourly/daily rollups, retention cleanup (raw 7d, hourly 90d, daily 2yr), baseline recomputation, SLO computation, log metric aggregation, log data cleanup |

The fast-path ticks (sys, app, heartbeat) are designed for negligible resource impact:
- `sysinfo` reads from `/proc` (kernel shared memory, no disk I/O)
- HTTP scrapes are localhost loopback (~0.1ms round-trip)
- DB stat catalog reads use PostgreSQL shared memory
- Batched INSERTs amortize write overhead

## Modules

### `src/metrics/`

**`system.rs`** — Collects host-level metrics via `sysinfo` crate: CPU usage per-core and aggregate, memory (total/used/available/swap), load averages (1/5/15 min), disk usage and IO per mount, network bytes/packets per interface.

**`process.rs`** — Per-process metrics from `/proc/{pid}/stat` and `/proc/{pid}/fd`. Tracks CPU percentage (delta-based calculation between samples), RSS bytes, file descriptor count, thread count, and process state. Filters to a configurable list of watched process names.

**`app.rs`** — Application-level metric collection with two strategies:
- **HTTP scrape**: Fetches JSON from API (`/metrics`) and proxy (`/metrics`) endpoints. Parses `ApiMetrics` (request counts, latency percentiles, error rates, active connections, cache stats, DB response time) and `ProxyMetrics` (connection counts, TLS handshake stats, rate limiting counters, upstream latency percentiles).
- **Direct PG queries**: Executes against `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_statio_user_tables`, `pg_stat_replication`, `pg_stat_statements`, `pg_stat_bgwriter`, `pg_stat_wal`, `pg_stat_activity` (wait events/locks). Collects 6 tiers of database metrics: connection pool stats, per-table stats (live/dead tuples, seq/idx scans, modifications), per-index stats (scans, reads, fetches, bloat), IO stats (heap/index/toast block reads/hits), replication lag, and statement-level stats (calls, total/mean time, rows).

### `src/db/`

**`writer.rs`** — `MetricWriter` with 10 typed `Vec` buffers (system, process, API, proxy, DB, table, index, IO, replication, statement). Each `push_*` method appends to the buffer. `flush()` builds bulk `INSERT` statements with `$1..$N` parameter binding and executes them in a single round-trip per table. Buffers are cleared after successful flush.

**`retention.rs`** — Time-series aggregation and cleanup:
- **Hourly rollups**: Aggregates raw rows from the past hour into `*_hourly` tables using `AVG`, `MAX`, `MIN`. Covers system, process, API, DB, table, and IO metrics. Uses `ON CONFLICT (bucket) DO UPDATE` for idempotent upserts.
  - **Cumulative counter handling**: API metrics `total_requests` and `failed_requests` are cumulative counters — the hourly rollup uses `GREATEST(max(col) - min(col), 0)` to compute per-hour deltas instead of `sum()`, which would produce inflated counts from cumulative snapshots.
  - **DB response time**: Includes `avg(db_response_ms)` in the API hourly rollup for baseline tracking.
- **Daily rollups**: Aggregates hourly rows from the past day into `*_daily` tables.
- **Retention cleanup**: Deletes raw data older than 7 days, hourly data older than 90 days, daily data older than 2 years. Also cleans resolved alerts older than 30 days and old insights/recommendations.

### `src/intelligence.rs`

Statistical intelligence engine that runs every 5 minutes:

**Baselines** — Computes 7-day and 30-day rolling mean and stddev for 24 global metrics across 5 domains plus dynamic per-table `dead_tup_ratio`. Stores in `collector.baselines` with `ON CONFLICT` upsert. Requires minimum 6 samples before establishing a baseline.

| Domain | Metrics |
|--------|---------|
| system | `cpu_system`, `load_1`, `mem_used_pct` |
| api | `p95_ms`, `rps`, `error_rate_pct`, `p99_ms`, `db_response_ms` |
| db | `cache_hit_ratio`, `active_conn`, `deadlocks`, `slow_queries`, `waiting_conn`, `blks_read`, `wal_bytes` |
| proxy | `latency_p95_ms`, `request_server_errors`, `latency_p50_ms`, `conn_active`, `handshake_failures`, `rl_requests_blocked` |
| logs | `error_count`, `warn_count`, `total_count`, `error_rate_pct` |
| table (dynamic) | `dead_tup_ratio` per table (computed separately for each table with sufficient history) |

**Anomaly detection** — For each baselined metric, fetches the latest raw value and computes:
- Z-score: `(value - mean) / stddev`
- Drift percentage: `(value - mean) / mean * 100`
- Severity: `critical` if |z| > 3, `warn` if |z| > 2, `info` otherwise
- Direction: `spike` or `drop` based on sign

Includes per-table anomaly detection for dead tuple ratio baselines. Deduplicates insights within 60 minutes.

**Lower-is-better suppression** — For metrics where a decrease is an improvement (not an anomaly), large negative drifts (>50%) are suppressed:
- `logs/error_count`, `logs/warn_count`, `logs/error_rate_pct`
- `db/slow_queries`, `db/deadlocks`
- `proxy/request_server_errors`, `proxy/handshake_failures`, `proxy/rl_requests_blocked`
- `table/dead_tup_ratio` (a drop means vacuum worked)

**Cross-domain correlation** — When 2+ anomalies from different domains co-occur in the same detection cycle, a `correlation` insight is generated linking them (e.g., API latency spike + DB cache drop + proxy error increase noted as a single correlated event). Rate-limited to one correlation insight per 30 minutes.

**SLO tracking** — Three SLOs with 30-day sliding window:
- `api_uptime`: Target 99.9% — counts periods where error rate exceeds 1%
- `api_latency_p95`: Target 200ms — counts periods where p95 > target
- `db_cache_hit`: Target 99% — counts periods where cache hit ratio < target

Computes error budget as `violations / allowed_violations * 100`. Generates `slo_violation` insights when SLOs are not met. Skipped for the first 10 minutes after collector restart to avoid counting the deployment gap as a violation.

**Recommendations** — Rule-based checks generating actionable recommendations with auto-cleanup when conditions normalize:

| Category | Target | Trigger | Severity |
|----------|--------|---------|----------|
| vacuum | `{schema}.{table}` | Dead tuples > 20% (warn), > 50% (crit) | warn/critical |
| system | cpu | CPU > 80% (warn), > 95% (crit) | warn/critical |
| system | memory | Memory > 85% (warn), > 95% (crit) | warn/critical |
| system | load | Load avg > 4.0 | warn |
| system | disk:{mount} | Disk > 85% (warn), > 95% (crit) | warn/critical |
| system | swap | Swap > 50% (info), > 80% (warn) | info/warn |
| api | db_response_time | DB response > 50ms (warn), > 200ms (crit) | warn/critical |
| api | latency | p95 > 500ms (warn), > 2000ms (crit) | warn/critical |
| api | errors | Error rate > 5% (warn), > 20% (crit) | warn/critical |
| api | connections | Active connections > 100 | warn |
| api | p99_latency | p99 > 2000ms (warn), > 5000ms (crit) | warn/critical |
| proxy | latency | p95 > 1000ms (warn), > 5000ms (crit) | warn/critical |
| proxy | errors | 5xx rate > 5% (warn), > 20% (crit) | warn/critical |
| proxy | concurrency | Requests in progress > 500 | warn |
| proxy | tls | Handshake failures > 10 | warn |
| proxy | rate_limit | Rate-limited > 100 | info |
| db | cache | Cache hit ratio < 95% (info), < 90% (warn) | info/warn |
| db | connections | Total connections > 80 | warn |
| db | deadlocks | Deadlocks > 0 | warn |
| db | slow_queries | Slow queries > 5 | warn |
| db | waiting | Waiting connections > 5 | warn |
| db | checkpoint | Backend fsyncs > 0 | warn |
| db | wal | WAL bytes > 100MB | info |
| db | replication:{slot} | Replication lag > 10s (warn), > 30s (crit) | warn/critical |
| performance | io_read | Avg disk read latency > 50ms | warn |
| performance | io_write | Avg disk write latency > 50ms | warn |
| performance | {table} | High update churn | warn |
| process | {name} | Memory > 500MB (warn), > 1GB (crit) | warn/critical |
| process | {name} | CPU > 30% (warn), > 80% (crit) | warn/critical |
| process | {name} | FDs > 500 (info), > 1000 (warn) | info/warn |
| process | {name} | Crashed/stopped | critical |
| index | unused:{name} | Unused indexes > 10MB (excludes primary keys) | info |
| query | query:{id} | Avg > 500ms (info), > 2000ms (warn) | info/warn |
| query | temp:{id} | Temp blocks spilled > 10000 | info |
| security | {event_type} | Security events detected | warn |

### `src/log_ingest.rs`

Streaming log ingestion engine that polls 13 sources every 15 seconds:

**Sources** — 13 built-in log sources across two ingestion strategies:

| Source | Type | Path/Unit | Parser |
|--------|------|-----------|--------|
| `postgresql` | file | `/var/log/postgresql/postgresql-16-main.log` | postgresql |
| `api` | journal | `pqcrypta-api` | journalctl |
| `proxy` | journal | `pqcrypta-proxy` | journalctl |
| `collector` | journal | `pqcrypta-collector` | journalctl |
| `auth` | file | `/var/log/auth.log` | syslog |
| `fail2ban` | file | `/var/log/fail2ban.log` | fail2ban |
| `apache_error` | file | `/var/log/apache2/api.pqcrypta.com-error.log` | apache_error |
| `apache_internal_error` | file | `/var/log/apache2/pqcrypta-internal-error.log` | apache_error |
| `health_check` | file | `/var/log/pqcrypta_health_check.log` | simple_timestamp |
| `blocklist` | file | `/var/log/pqcrypta-proxy/blocklist_sync.log` | simple_timestamp |
| `bot_detection` | file | `/var/log/pqcrypta-proxy/bot_detection.log` | simple_timestamp |
| `kernel` | file | `/var/log/kern.log` | syslog |
| `certbot` | file | `/var/log/letsencrypt/letsencrypt.log` | certbot |

**File ingestion** — Tracks byte offset and inode per source in `log_positions`. On each tick: checks inode for log rotation (reset offset if changed or file shrank), reads up to 64KB from saved offset, parses complete lines only, batch INSERTs with multi-row VALUES.

**Journal ingestion** — Runs `journalctl -u UNIT -o json --after-cursor=X -n N` as an async subprocess. Handles MESSAGE fields that arrive as byte arrays (ANSI-encoded output from Rust tracing). Strips ANSI escape sequences and extracts level/component from tracing format. First run limits to 100 lines to avoid massive backfill.

**7 parsers:**
1. **postgresql** — `%m [%p] %q%u@%d LEVEL: message`, handles multi-line STATEMENT continuation, skips empty messages from HINT/DETAIL lines
2. **journalctl** — JSON objects with `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER`, `__CURSOR`; handles byte-array MESSAGE and Rust tracing format
3. **syslog** — RFC 3164 (`Mon DD HH:MM:SS hostname process[pid]: message`) and RFC 5424/ISO timestamps (`2026-02-15T00:00:54.808228-06:00`)
4. **apache_error** — `[timestamp] [module:level] [pid N] message`
5. **simple_timestamp** — `[YYYY-MM-DD HH:MM:SS] message` or `YYYY-MM-DD HH:MM:SS message`
6. **fail2ban** — `YYYY-MM-DD HH:MM:SS,mmm fail2ban.module [pid]: LEVEL message`
7. **certbot** — `YYYY-MM-DD HH:MM:SS,mmm:LEVEL:module:message`

**Fingerprinting** — Normalizes messages (digits to `#`, IPs to `<IP>`, UUIDs to `<UUID>`), hashes with SHA-256 truncated to 16 hex chars: `sha256(source|level|normalized)[..16]`.

### `src/log_analysis.rs`

Log-specific analysis that runs on `intel_tick` (every 5 minutes), feeding into the existing insights, recommendations, and alerts tables:

**Pattern detection** — Upserts `log_patterns` from entries in the last 5 minutes, grouped by fingerprint, source, and level. Auto-resolves patterns not seen in 1 hour.

**Error spike detection** — Compares 5-minute error count against the hourly baseline from `log_metrics_hourly`. If error count exceeds 3x the baseline average, inserts an `error_spike` insight (domain=`logs`).

**Security event detection:**
- SSH brute force: >5 failed logins from same IP in 5 minutes -> alert (type=`ssh_brute_force`)
- Fail2ban ban actions -> insight (domain=`logs`, metric=`fail2ban_ban`)
- UFW/firewall blocks from same IP >10 in 5 minutes -> insight (domain=`logs`, metric=`firewall_block`)

**Actionable recommendations** — Context-aware pattern matching generates specific remediation steps:
- SSH brute force -> numbered steps: check attacking IPs, block with `ufw`, verify fail2ban, disable password auth
- PostgreSQL connection errors -> check API server, verify `pg_hba.conf`, test connectivity
- Deadlock detection -> review transaction ordering, check long-running queries
- Permission denied errors -> check file ownership, verify service user permissions
- Disk/IO errors -> check filesystem health, review SMART status
- Recurring error patterns (>10 occurrences/hour) -> source-specific recommendations with diagnostic commands

**Cross-domain log correlation** — Detects when log error spikes coincide with metric anomalies (e.g., API error spike in logs at the same time as latency anomaly in metrics).

**Hourly aggregation** (on `agg_tick`) — Rolls up `log_entries` into `log_metrics_hourly` (source, level, count per hour). Runs `cleanup_log_data()` for retention enforcement.

### `src/watchdog.rs`

Health monitoring runs on a dedicated 30s tick, decoupled from the 5s heartbeat:
- **Heartbeat** (5s): Inserts liveness row into `collector.heartbeat`
- **Staleness check** (30s): Alerts if last heartbeat exceeds 6x heartbeat interval
- **Table health** (30s): Detects bloated tables (dead tuples > 20% with > 1000 live rows and > 500 dead rows) and tables not vacuumed in 7+ days
- **Process health** (30s): Checks for high CPU (> 50%), high memory (> 1GB RSS), high FDs (> 1000), high threads (> 500), bad states (zombie/D-state/stopped), and expected processes that are missing. Condition-based auto-resolve: when a process metric returns to healthy, its alert is resolved immediately (no time-based delay).
- **API error rate** (30s): Alerts if error rate exceeds 5% with > 100 total requests. Auto-resolves when error rate drops below threshold.
- **DB response time** (30s): Alerts if 5-minute average DB response time exceeds 100ms. Auto-resolves when response time drops below threshold.

All watchdog alerts use deduplication (matching alert type + subject prefix) and have both condition-based and time-based auto-resolution fallbacks.

### `src/config.rs`

TOML configuration with environment variable overrides. Default path: `/etc/pqcrypta/collector.toml`.

```toml
[database]
host = "127.0.0.1"       # env: DB_HOST
port = 5432               # env: DB_PORT
name = "pqcrypta"         # env: DB_NAME
user = "pqcrypta_user"    # env: DB_USER
password = ""             # env: DB_PASS

[intervals]
system_secs = 10          # env: SYSTEM_INTERVAL_SECS
app_secs = 10             # env: APP_INTERVAL_SECS
heartbeat_secs = 5        # env: HEARTBEAT_INTERVAL_SECS

[scrape]
api_metrics_url = "http://127.0.0.1:3003/metrics"
proxy_metrics_url = "http://127.0.0.1:8082/metrics/json"

[retention]
raw_days = 14
hourly_days = 90
daily_days = 365

[collector]
batch_size = 50
processes = ["pqcrypta-api", "pqcrypta-proxy", "pqcrypta-collector", "postgres", "apache2", "php-fpm"]

[logs]
enabled = true                # env: LOG_ENABLED
tick_secs = 15                # env: LOG_TICK_SECS
batch_size = 100              # max rows per INSERT
max_lines_per_tick = 500      # cap per tick across all sources
chunk_size = 65536            # bytes to read per file source
```

## Database Schema

Five migration files in `migrations/`:

**001_collector_schema.sql** — Core tables:
- `collector.system_metrics_raw` — Host CPU, memory, load, disk, network (14 columns)
- `collector.process_metrics_raw` — Per-process CPU, RSS, FDs, threads, state, uptime (10 columns)
- `collector.api_metrics_raw` — API request counts, latency percentiles, errors, cache stats, DB response time (17 columns)
- `collector.proxy_metrics_raw` — Proxy connections, TLS stats, rate limiting, upstream latency (24 columns)
- `collector.db_metrics_raw` — PostgreSQL connection pool, transaction counts, cache ratios, slow queries (14 columns)
- `collector.heartbeat` — Collector liveness tracking
- `collector.alerts` — Alert storage with deduplication and resolution tracking
- `collector.system_metrics_hourly` — Hourly system aggregates (CPU, load, memory, network)
- `collector.process_metrics_hourly` — Hourly per-process aggregates (CPU, RSS, FDs, threads)
- `collector.api_metrics_hourly` — Hourly API aggregates (RPS, latency percentiles, error rate, request deltas, DB response time)
- `collector.db_metrics_hourly` — Hourly DB aggregates (connections, cache ratio, transaction counts, deadlocks, slow queries)
- `collector.system_metrics_daily` — Daily system aggregates

**002_extended_pg_metrics.sql** — Extended PostgreSQL monitoring:
- Adds bgwriter, WAL, wait event, and lock columns to `db_metrics_raw`
- `collector.table_metrics_raw` — Per-table live/dead tuples, seq/idx scans, modifications, autovacuum timing
- `collector.index_metrics_raw` — Per-index scans, reads, fetches, size, bloat
- `collector.io_metrics_raw` — Per-table heap/index/toast block reads and hits
- `collector.replication_metrics_raw` — Replication state, write/flush/replay lag, sent/write/flush/replay LSN
- `collector.statement_metrics_raw` — Top N statements by total time (calls, rows, mean/total time, shared block stats)
- Corresponding hourly aggregate tables for table, IO, and statement metrics

**003_intelligence_schema.sql** — Intelligence layer:
- `collector.baselines` — Statistical baselines (domain, metric, metric_key, time_window, mean, stddev, min, max, sample_count)
- `collector.insights` — Detected anomalies, drift events, correlations, SLO violations (type, severity, domain, metric, z_score, drift_pct, direction)
- `collector.recommendations` — Actionable recommendations (category, severity, target, title, description, action, acknowledged)
- `collector.slo_tracking` — SLO computation results (slo_name, target, actual, met, budget_consumed, violations, total_periods)

**004_log_tables.sql** — Log ingestion and analysis:
- `collector.log_entries` — Raw log rows (ts, source, level, component, message, context JSONB, fingerprint). 7-day retention.
- `collector.log_metrics_hourly` — Hourly counts by source+level for trend charts. 90-day retention.
- `collector.log_positions` — Per-source byte offset/inode (files) or journal cursor (systemd). Permanent.
- `collector.log_patterns` — Recurring error fingerprints with occurrence count, first/last seen, sample message, resolved status. 30-day retention after resolved.
- Indexes: `(ts DESC)`, `(source, ts DESC)`, partial on `level IN ('error','warn')`, `(fingerprint, ts DESC)`.
- Cleanup function: `collector.cleanup_log_data()` enforces retention policies.

**005_log_enhancements.sql** — Additional log analysis tables:
- `collector.log_fingerprint_hourly` — Hourly trending for top error fingerprints (bucket, fingerprint, source, level, count). 90-day retention.
- `collector.security_events` — Security event summary populated by `detect_security_events` (ts, event_type, source_ip, details, count). 30-day retention.
- Enhanced `cleanup_log_data()` — Extended to clean up both new tables alongside existing retention policies.

All tables use `ts TIMESTAMPTZ` as the primary time column with descending indexes for efficient latest-value queries.

## Deployment

### systemd

A service unit is provided in `pqcrypta-collector.service`:

```ini
[Service]
ExecStart=/usr/local/bin/pqcrypta-collector
Restart=always
RestartSec=5
MemoryMax=64M
CPUQuota=5%
Environment=RUST_LOG=info
Environment=COLLECTOR_CONFIG=/etc/pqcrypta/collector.toml
```

### Build

```bash
cargo build --release
cp target/release/pqcrypta-collector /usr/local/bin/
```

### Prerequisites

- PostgreSQL 15+ with the `collector` schema created
- Run migrations in order: `001_collector_schema.sql`, `002_extended_pg_metrics.sql`, `003_intelligence_schema.sql`, `004_log_tables.sql`, `005_log_enhancements.sql`
- API server running on port 3003 with `/metrics` endpoint
- Proxy server running on port 8082 with `/metrics` endpoint (optional)
- Read access to log files in `/var/log/` (auth.log, kern.log, postgresql, apache2, fail2ban, letsencrypt, pqcrypta-proxy)
- `journalctl` available for systemd journal sources (pqcrypta-api, pqcrypta-proxy, pqcrypta-collector)

## Resource Usage

The collector is designed to be lightweight despite 10s collection intervals:
- Single-threaded async runtime (`tokio` current-thread)
- Memory capped at 64MB via systemd `MemoryMax`
- CPU capped at 5% via systemd `CPUQuota`
- Batched database writes minimize connection overhead (single DB connection, no pool)
- Fast-path metrics (sys, app) read from kernel shared memory and localhost HTTP — no disk I/O
- Heartbeat decoupled from heavier watchdog checks to keep liveness detection fast
- All intervals are configurable via TOML or environment variables

## License

MIT
