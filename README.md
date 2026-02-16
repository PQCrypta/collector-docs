# pqcrypta-collector

Async metrics collector, log ingestion engine, and intelligence layer for PQCrypta infrastructure. Single-threaded Rust binary that scrapes system, process, application, and database metrics on configurable intervals, ingests logs from 13 sources with structured parsing, writes everything to PostgreSQL with batched inserts, performs time-series aggregation and retention, and runs statistical anomaly detection with SLO tracking and actionable recommendations.

## Architecture

```
                    ┌─────────────────────────────┐
                    │         main loop           │
                    │   tokio::select! event hub  │
                    └──────┬──────────────────┬───┘
                           │                  │
         ┌────────────┐────┼──────────┐───────┼──────────┐
         ▼            ▼    ▼          ▼       ▼          ▼
    sys_tick      app_tick log_tick intel_tick agg_tick watchdog
     (10s)         (10s)   (15s)    (5min)    (1hr)    (30s)
         │            │      │        │         │        │
         ▼            ▼      ▼        ▼         ▼        ▼
   ┌──────────┐ ┌────────┐ ┌─────┐ ┌─────────┐ ┌───────┐ ┌─────────┐
   │  sysinfo │ │HTTP+PG │ │logs │ │baselines│ │rollups│ │staleness│
   │  /proc   │ │scrape  │ │files│ │anomalies│ │retain │ │health   │
   └─────┬────┘ └────┬───┘ │jrnl │ │SLOs     │ │log agg│ └────┬────┘
         │           │     └──┬──┘ │recs     │ └───┬───┘      │
         │           │        │    │log anlys│     │          │
         ▼           ▼        ▼    └────┬────┘     ▼          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │              MetricWriter + LogIngester (batched)            │
   │  10 metric buffers + log batch INSERT on flush               │
   └──────────────────────────┬───────────────────────────────────┘
                              ▼
                        ┌───────────┐
                        │ PostgreSQL│
                        └───────────┘
```

### Tick intervals

| Tick | Interval | Responsibility |
|------|----------|----------------|
| `sys_tick` | 10s | CPU, memory, load, disk, network via `sysinfo` + `/proc` |
| `app_tick` | 10s | API metrics (HTTP scrape port 3003), proxy metrics (HTTP scrape port 8082), DB stats (direct pg_stat queries) |
| `heartbeat_tick` | 5s | Lightweight liveness heartbeat (single INSERT) |
| `log_tick` | 15s | Log ingestion from 13 file and journal sources, batch INSERT into `log_entries` |
| `watchdog_tick` | 30s | Staleness detection, table health, process health, API error rate |
| `pg_extended_tick` | 5min | Per-table, per-index, IO, replication, statement stats |
| `intel_tick` | 5min | Intelligence layer: baselines, anomaly detection, SLO computation, recommendations, log pattern analysis, error spike detection, security event detection |
| `agg_tick` | 1hr | Hourly/daily rollups, retention cleanup (raw 7d, hourly 90d, daily 2yr), log metric aggregation, log data cleanup |

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
- **HTTP scrape**: Fetches JSON from API (`/metrics`) and proxy (`/metrics`) endpoints. Parses `ApiMetrics` (request counts, latency percentiles, error rates, active connections, cache stats) and `ProxyMetrics` (connection counts, TLS handshake stats, rate limiting counters, upstream latency percentiles).
- **Direct PG queries**: Executes against `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_statio_user_tables`, `pg_stat_replication`, `pg_stat_statements`, `pg_stat_bgwriter`, `pg_stat_wal`, `pg_stat_activity` (wait events/locks). Collects 6 tiers of database metrics: connection pool stats, per-table stats (live/dead tuples, seq/idx scans, modifications), per-index stats (scans, reads, fetches, bloat), IO stats (heap/index/toast block reads/hits), replication lag, and statement-level stats (calls, total/mean time, rows).

### `src/db/`

**`writer.rs`** — `MetricWriter` with 10 typed `Vec` buffers (system, process, API, proxy, DB, table, index, IO, replication, statement). Each `push_*` method appends to the buffer. `flush()` builds bulk `INSERT` statements with `$1..$N` parameter binding and executes them in a single round-trip per table. Buffers are cleared after successful flush.

**`retention.rs`** — Time-series aggregation and cleanup:
- **Hourly rollups**: Aggregates raw rows from the past hour into `*_hourly` tables using `AVG`, `MAX`, `MIN`. Covers system, process, API, DB, table, and IO metrics. Uses `ON CONFLICT (hour) DO UPDATE` for idempotent upserts.
- **Daily rollups**: Aggregates hourly rows from the past day into `*_daily` tables.
- **Retention cleanup**: Deletes raw data older than 7 days, hourly data older than 90 days, daily data older than 2 years. Also cleans resolved alerts older than 30 days and old insights/recommendations.

### `src/intelligence.rs`

Statistical intelligence engine that runs every 5 minutes:

**Baselines** — Computes 7-day rolling mean and stddev for 20 global metrics across 3 domains (API, DB, proxy) plus per-table `dead_tup_ratio`. Stores in `collector.baselines` with `ON CONFLICT` upsert. Requires minimum 6 samples before establishing a baseline.

**Anomaly detection** — For each baselined metric, fetches the latest raw value and computes:
- Z-score: `(value - mean) / stddev`
- Drift percentage: `(value - mean) / mean * 100`
- Severity: `critical` if |z| > 3, `warn` if |z| > 2, `info` otherwise
- Direction: `spike` or `drop` based on sign

Includes per-table anomaly detection for dead tuple ratio baselines. Deduplicates insights within 60 minutes.

**SLO tracking** — Three SLOs with 30-day sliding window:
- `api_uptime`: Target 99.9% — counts periods where error rate exceeds 1%
- `api_latency_p95`: Target 200ms — counts periods where p95 > target
- `db_cache_hit`: Target 99% — counts periods where cache hit ratio < target

Computes error budget as `violations / allowed_violations * 100`. Generates `slo_violation` insights when SLOs are not met.

**Recommendations** — Rule-based checks generating actionable recommendations:
- Table bloat (dead tuples > 20% of live)
- Missing/unused indexes (seq scans > 1000, index usage < 10%)
- Slow queries (mean execution time > 100ms)
- Process issues (high CPU > 80%, high memory > 512MB, recent restart < 5min uptime)
- API performance (avg latency > 500ms, error rate > 5%)
- Proxy issues (handshake failures > 10, rate limiting blocks > 100)
- DB issues (waiting connections > 5, checkpoint pressure, WAL activity > 100MB)

Auto-cleanup resolves stale recommendations when conditions normalize (e.g., CPU drops below 20%, uptime exceeds 10 minutes, dead tuple ratio drops below threshold).

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

**6 parsers:**
1. **postgresql** — `%m [%p] %q%u@%d LEVEL: message`, handles multi-line STATEMENT continuation, skips empty messages from HINT/DETAIL lines
2. **journalctl** — JSON objects with `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER`, `__CURSOR`; handles byte-array MESSAGE and Rust tracing format
3. **syslog** — RFC 3164 (`Mon DD HH:MM:SS hostname process[pid]: message`) and RFC 5424/ISO timestamps (`2026-02-15T00:00:54.808228-06:00`)
4. **apache_error** — `[timestamp] [module:level] [pid N] message`
5. **simple_timestamp** — `[YYYY-MM-DD HH:MM:SS] message` or `YYYY-MM-DD HH:MM:SS message`
6. **fail2ban** — `YYYY-MM-DD HH:MM:SS,mmm fail2ban.module [pid]: LEVEL message`
7. **certbot** — `YYYY-MM-DD HH:MM:SS,mmm:LEVEL:module:message`

**Fingerprinting** — Normalizes messages (digits→`#`, IPs→`<IP>`, UUIDs→`<UUID>`), hashes with SHA-256 truncated to 16 hex chars: `sha256(source|level|normalized)[..16]`.

### `src/log_analysis.rs`

Log-specific analysis that runs on `intel_tick` (every 5 minutes), feeding into the existing insights, recommendations, and alerts tables:

**Pattern detection** — Upserts `log_patterns` from entries in the last 5 minutes, grouped by fingerprint, source, and level. Auto-resolves patterns not seen in 1 hour.

**Error spike detection** — Compares 5-minute error count against the hourly baseline from `log_metrics_hourly`. If error count exceeds 3x the baseline average, inserts an `error_spike` insight (domain=`logs`).

**Security event detection:**
- SSH brute force: >5 failed logins from same IP in 5 minutes → alert (type=`ssh_brute_force`)
- Fail2ban ban actions → insight (domain=`logs`, metric=`fail2ban_ban`)
- UFW/firewall blocks from same IP >10 in 5 minutes → insight (domain=`logs`, metric=`firewall_block`)

**Actionable recommendations** — Context-aware pattern matching generates specific remediation steps:
- SSH brute force → numbered steps: check attacking IPs, block with `ufw`, verify fail2ban, disable password auth
- PostgreSQL connection errors → check API server, verify `pg_hba.conf`, test connectivity
- Deadlock detection → review transaction ordering, check long-running queries
- Permission denied errors → check file ownership, verify service user permissions
- Disk/IO errors → check filesystem health, review SMART status
- Recurring error patterns (>10 occurrences/hour) → source-specific recommendations with diagnostic commands

**Cross-domain correlation** — Detects when log error spikes coincide with metric anomalies (e.g., API error spike in logs at the same time as latency anomaly in metrics).

**Hourly aggregation** (on `agg_tick`) — Rolls up `log_entries` into `log_metrics_hourly` (source, level, count per hour). Runs `cleanup_log_data()` for retention enforcement.

### `src/watchdog.rs`

Health monitoring runs on a dedicated 30s tick, decoupled from the 5s heartbeat:
- **Heartbeat** (5s): Inserts liveness row into `collector.heartbeat`
- **Staleness check** (30s): Alerts if last heartbeat exceeds 6x heartbeat interval
- **Table health** (30s): Detects bloated tables (dead tuples > 20% with > 100 live rows) and tables not vacuumed in 7+ days
- **Process health** (30s): Checks for high CPU (> 50%), high memory (> 1GB RSS), high FDs (> 1000), high threads (> 500), bad states (zombie/D-state/stopped), and expected processes that are missing
- **API error rate** (30s): Alerts if error rate exceeds 5% with > 100 total requests

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

Four migration files in `migrations/`:

**001_collector_schema.sql** — Core tables:
- `collector.system_metrics_raw` — Host CPU, memory, load, disk, network
- `collector.process_metrics_raw` — Per-process CPU, RSS, FDs, threads, state
- `collector.api_metrics_raw` — API request counts, latency percentiles, errors, cache stats
- `collector.proxy_metrics_raw` — Proxy connections, TLS stats, rate limiting, upstream latency
- `collector.db_metrics_raw` — PostgreSQL connection pool, transaction counts, cache ratios
- `collector.heartbeat` — Collector liveness tracking
- `collector.alerts` — Alert storage with deduplication and resolution tracking
- Corresponding `*_hourly` and `*_daily` aggregate tables

**002_extended_pg_metrics.sql** — Extended PostgreSQL monitoring:
- Adds bgwriter, WAL, wait event, and lock columns to `db_metrics_raw`
- `collector.table_metrics_raw` — Per-table live/dead tuples, seq/idx scans, modifications, autovacuum timing
- `collector.index_metrics_raw` — Per-index scans, reads, fetches, size, bloat
- `collector.io_metrics_raw` — Per-table heap/index/toast block reads and hits
- `collector.replication_metrics_raw` — Replication state, write/flush/replay lag, sent/write/flush/replay LSN
- `collector.statement_metrics_raw` — Top N statements by total time (calls, rows, mean/total time, shared block stats)
- Corresponding hourly aggregate tables

**003_intelligence_schema.sql** — Intelligence layer:
- `collector.baselines` — Statistical baselines (domain, metric, metric_key, time_window, mean, stddev, min, max, sample_count)
- `collector.insights` — Detected anomalies and drift events (type, severity, domain, metric, z_score, drift_pct, direction)
- `collector.recommendations` — Actionable recommendations (category, severity, target, title, description, action, acknowledged)
- `collector.slo_tracking` — SLO computation results (slo_name, target, actual, met, budget_consumed, violations, total_periods)

**004_log_tables.sql** — Log ingestion and analysis:
- `collector.log_entries` — Raw log rows (ts, source, level, component, message, context JSONB, fingerprint). 7-day retention.
- `collector.log_metrics_hourly` — Hourly counts by source+level for trend charts. 90-day retention.
- `collector.log_positions` — Per-source byte offset/inode (files) or journal cursor (systemd). Permanent.
- `collector.log_patterns` — Recurring error fingerprints with occurrence count, first/last seen, sample message, resolved status. 30-day retention after resolved.
- Indexes: `(ts DESC)`, `(source, ts DESC)`, partial on `level IN ('error','warn')`, `(fingerprint, ts DESC)`.
- Cleanup function: `collector.cleanup_log_data()` enforces retention policies.

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
- Run migrations in order: `001_collector_schema.sql`, `002_extended_pg_metrics.sql`, `003_intelligence_schema.sql`, `004_log_tables.sql`
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


Live dashboard: https://pqcrypta.com/monitor/
