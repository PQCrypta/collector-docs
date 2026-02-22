# pqcrypta-collector

> Auto-synced to [collector-docs](https://github.com/PQCrypta/collector-docs)

Async metrics collector, log ingestion engine, and intelligence layer for PQCrypta infrastructure. Single-threaded Rust binary that scrapes system, process, application, and database metrics on configurable intervals, ingests logs from 13 sources with structured parsing, writes everything to PostgreSQL with batched inserts, performs time-series aggregation and retention, runs statistical anomaly detection with SLO tracking and actionable recommendations, and provides disk-backed durable queuing with cardinality protection.

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
   │ /proc/stat│ │HTTP+PG │ │logs │ │anomalies│ │rollups│ │staleness│
   │ +sysinfo  │ │scrape  │ │files│ │health   │ │retain │ │health   │
   └─────┬────┘ └────┬───┘ │jrnl │ │capacity │ │baselin│ └────┬────┘
         │           │     └──┬──┘ │SLOs+recs│ │SLO    │      │
         │           │        │    └────┬───┘ │log agg│      │
         ▼           ▼        ▼         ▼     └───┬──┘      ▼
   ┌──────────────────────────────────────────────────────────────┐
   │              MetricWriter + LogIngester (batched)             │
   │  10 metric buffers + log batch INSERT on flush                │
   │  disk-backed spill queue (JSONL) on overflow                  │
   └──────────┬──────────────────────────────┬────────────────────┘
              ▼                              ▼
        ┌───────────┐               ┌────────────────┐
        │ PostgreSQL │               │ Disk Queue     │
        └───────────┘               │ (queue.jsonl)  │
                                    └────────────────┘
```

### Tick intervals

| Tick | Interval | Responsibility |
|------|----------|----------------|
| `sys_tick` | 10s | CPU (direct `/proc/stat` delta), memory, load, disk, network via `sysinfo`. Also emits self-monitoring metrics (buffer depths captured pre-flush, flush/spill counts, tick duration, rows written) to `collector_self_metrics`. |
| `app_tick` | 10s | API metrics (HTTP scrape port 3003), proxy metrics (HTTP scrape port 8082), DB stats (direct pg_stat queries) |
| `heartbeat_tick` | 5s | Lightweight liveness heartbeat (single INSERT) |
| `log_tick` | 15s | Log ingestion from 13 file and journal sources, batch INSERT into `log_entries` |
| `reconnect_tick` | 5s | DB health monitoring, automatic reconnection on connection loss |
| `watchdog_tick` | 30s | Staleness detection, table health, process health, API error rate, DB response time, long queries, connection trend, IO pressure |
| `config_tick` | 60s | Config file hot-reload (mtime-based change detection) |
| `pg_extended_tick` | 5min | Per-table, per-index, IO, replication, statement stats |
| `intel_tick` | 5min | Anomaly detection, recommendations, health scores, capacity predictions, log pattern analysis, error spike detection, security event detection |
| `agg_tick` | 1hr | Hourly/daily rollups, retention cleanup (raw 14d, hourly 90d, daily 365d defaults), baseline recomputation, SLO computation, log metric aggregation, log data cleanup, stale cardinality pruning, fingerprint budget reset, trend forecast computation (linear regression over 7-day hourly series), seasonal baseline computation (168 hour-of-week slots over 30 days). First tick is consumed at startup (no-op); a one-shot 3-minute delayed startup runs aggregation + baselines + SLOs + trend/seasonal computation once data has accumulated. |

The fast-path ticks (sys, app, heartbeat) are designed for negligible resource impact:
- `sysinfo` reads from `/proc` (kernel shared memory, no disk I/O)
- HTTP scrapes are localhost loopback (~0.1ms round-trip)
- DB stat catalog reads use PostgreSQL shared memory
- Batched INSERTs amortize write overhead

## Modules

### `src/metrics/`

**`system.rs`** — Collects host-level metrics. CPU usage is read directly from `/proc/stat` (delta-based jiffies calculation between ticks) providing accurate user, system, and idle percentages — bypasses the `sysinfo` crate's unreliable CPU reporting on Linux. Memory (total/used/available/swap), load averages (1/5/15 min), disk usage per mount (total/available/usage_pct stored as JSONB), and network bytes per interface still use `sysinfo`.

**`process.rs`** — Per-process metrics from `/proc/{pid}/stat` and `/proc/{pid}/fd`. Tracks CPU percentage (delta-based calculation between samples), RSS bytes, VSZ bytes, file descriptor count, thread count, and process state. Collects a configurable list of watched process names plus all other processes with non-zero RSS, sorted by memory usage descending.

**`app.rs`** — Application-level metric collection with two strategies:
- **HTTP scrape**: Fetches JSON from API (`/metrics`) and proxy (`/metrics/json`) endpoints. Parses `ApiMetrics` (request counts, latency percentiles, error rates, `waf_blocked_requests`, active connections, cache stats, DB response time) and `ProxyMetrics` (connection counts, TLS handshake stats, rate limiting counters, upstream latency percentiles). `waf_blocked_requests` tracks requests blocked by the WAF IP-blocklist and bot-blocklist — kept separate from `failed_requests` so bot attacks do not inflate error-rate SLOs.
- **Direct PG queries**: Executes against `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_stat_io` (PG16+), `pg_stat_replication`, `pg_stat_statements`, `pg_stat_bgwriter`, `pg_stat_wal`, `pg_stat_activity` (wait events/locks). Collects 6 tiers of database metrics: connection pool stats, per-table stats (live/dead tuples, seq/idx scans, modifications), per-index stats (scans, reads, fetches, size), per-backend-type IO stats (reads, writes, hits, evictions, fsyncs with timing), replication lag, and statement-level stats (calls, total/mean time, rows).

### `src/db/`

**`writer.rs`** — `MetricWriter` with 10 typed `VecDeque` ring buffers (system, process, API, proxy, DB, table, index, IO, replication, statement) and a disk-backed spill queue. Each `push_*` method appends to the buffer; when a buffer reaches `max_capacity` (batch_size × 20), the oldest entry is serialized to the disk queue instead of being dropped. `flush()` wraps all INSERTs in a `BEGIN`/`COMMIT` transaction with per-query timeout protection; after successful commit, any spilled records are drained from disk and flushed in batches. On any error, `ROLLBACK` is executed and buffers are restored. `spill_all_to_disk()` drains all 10 in-memory buffers to disk at shutdown when DB is unhealthy. `drain_all_spilled()` recovers spilled data from a previous run on startup. Tracks `flush_count` and `total_rows_written` counters for self-monitoring, exposed along with `buffer_depths()` (per-buffer row counts) and `disk_queue_bytes()` for the dashboard Collector tab. Buffer depths are captured before the flush so the self-metrics chart shows how many rows accumulated during each collection cycle rather than always showing zero (post-flush empty buffers).

**`disk_queue.rs`** — Disk-backed durable queue for metric spill during DB outages. Uses a single append-only JSONL file (`queue.jsonl`) with tagged serde (`SpilledRecord` enum covering all 10 metric types). `spill()` appends a JSON line with disk budget enforcement (default 100MB). `drain(batch_size)` reads records from the front and atomically rewrites the remainder via temp file + rename. `current_bytes()` exposes queue size for self-monitoring. Configurable via `queue_dir` (default `/var/lib/pqcrypta-collector/queue`) and `queue_max_mb` (default 100).

**`helpers.rs`** — `timed_execute()` wraps `client.execute()` with a `tokio::time::timeout` to prevent stuck queries from blocking the event loop indefinitely.

**`retention.rs`** — Time-series aggregation and cleanup:
- **Hourly rollups**: Aggregates raw rows from the past hour into `*_hourly` tables using `AVG`, `MAX`, `MIN`. Covers system, process, API, DB, table, and IO metrics. Uses `ON CONFLICT (bucket) DO UPDATE` for idempotent upserts.
  - **Restart-safe counter handling**: API metrics `total_requests`, `failed_requests`, and `waf_blocked_requests` are cumulative counters that reset to 0 on API restart. The hourly rollup computes per-interval deltas using a LAG window function (`GREATEST(col - LAG(col), 0)`), clamping negative deltas (counter resets) to 0 before summing across the hour. This prevents an API restart mid-window from producing a false spike equal to the pre-restart maximum.
  - **DB response time**: Includes `avg(db_response_ms)` in the API hourly rollup for baseline tracking.
- **Daily rollups**: Aggregates hourly rows from the past day into `*_daily` tables.
- **Retention cleanup**: Deletes raw data older than configured days (default 14), hourly data older than configured days (default 90), daily data older than configured days (default 365).
- **Consolidated cleanup** (hourly): Removes heartbeats older than 24 hours, resolved alerts older than 7 days, resolved insights older than 7 days. Moved from the 30s watchdog tick to reduce unnecessary frequency.
- **Cardinality pruning** (hourly): Deletes stale `log_patterns` not seen in 7 days and stale `baselines` not updated in 30 days to prevent unbounded table growth.

### `src/intelligence.rs`

Statistical intelligence engine with `Severity` enum (`Info`, `Warn`, `Critical`) for type-safe alert classification:

**Baselines** (runs on `agg_tick`, hourly) — Computes 7-day and 30-day rolling mean, stddev, and percentiles (p5/p25/p50/p75/p95) for 43 global metrics across 9 domains plus dynamic per-table `dead_tup_ratio`. Stores in `collector.baselines` with `ON CONFLICT` upsert. Requires minimum 6 samples before establishing a baseline. Skips NULL, NaN, and Inf values to prevent pollution from incomplete data windows.

**Trend forecasts** (runs on `agg_tick`, hourly) — For each of 49 baselined metrics (same set as baselines, covering all 9 domains plus per-table `dead_tup_ratio`), fetches hourly values from the past 7 days and runs linear regression using the existing `linear_regression()` function (slope, intercept, R²). Computes `slope_per_hour` (converted from per-epoch slope × 3600), `forecast_1h` / `forecast_6h` / `forecast_24h` extrapolations, a `trend_direction` label (`rising`, `falling`, `stable`), and a `confidence` score (R²). Results are upserted into `collector.trend_forecasts`. Feeds the Baselines table's Trend column in the monitor dashboard.

**Seasonal baselines** (runs on `agg_tick`, hourly) — For the same 49 metrics, fetches 30 days of raw values grouped by `hour_of_week` (0–167, computed as `(ISODOW - 1) * 24 + HOUR`). Groups and aggregates in Rust by `i16` hour-of-week key, computing mean, stddev, p50, p95, and sample_count per slot. Upserts into `collector.baselines_hourly` (168 slots per metric). Enables time-of-week-aware anomaly context — the Baselines table's Seasonal column shows the current-hour-of-week baseline. Requires `hour_of_week` stored as `smallint` (`i16`) in PostgreSQL.

| Domain | Metrics |
|--------|---------|
| system | `cpu_system`, `load_1`, `mem_used_pct` |
| api | `p95_ms`, `rps`, `error_rate_pct`, `p99_ms`, `db_response_ms`, `active_connections` |
| db | `cache_hit_ratio`, `active_conn`, `deadlocks`, `slow_queries`, `waiting_conn`, `blks_read`, `wal_bytes`, `xact_rollback`, `buffers_backend`, `checkpoint_write_time` |
| proxy | `latency_p95_ms`, `request_server_errors`, `latency_p50_ms`, `conn_active`, `handshake_failures`, `rl_requests_blocked`, `conn_total`, `request_client_errors` |
| logs | `error_count`, `warn_count`, `total_count`, `error_rate_pct` |
| process | `cpu_pct_max`, `rss_max`, `fd_avg` (MAX/AVG across all tracked processes per hour) |
| io | `read_time`, `write_time`, `evictions` (SUM across backend types per hour from `pg_stat_io`) |
| replication | `replay_lag_ms`, `flush_lag_ms` (MAX lag across slots — worst-case replication health) |
| statement | `mean_exec_time_ms`, `temp_blks_written` (aggregate across top-N queries from `pg_stat_statements`) |
| table (dynamic) | `dead_tup_ratio` per table (top 200 by activity, computed separately for each table with sufficient history) |

**Anomaly detection** (runs on `intel_tick`, every 5 min) — For each baselined metric, fetches the latest raw value and computes:
- Z-score: `(value - mean) / stddev`
- Drift percentage: `(value - mean) / mean * 100`
- Severity: `critical` if |z| >= 4 or |drift| >= 300%, `warn` if |z| >= 3 or |drift| >= 200%, `info` if |z| >= 2 or |drift| >= 100%
- Direction: `spike` or `drop` based on sign
- Warn and critical anomalies require 2+ consecutive detection cycles before being recorded (transient spike suppression)

Includes per-table anomaly detection for dead tuple ratio baselines. Deduplicates insights within 30 minutes.

**Lower-is-better suppression** — For metrics where a decrease is an improvement (not an anomaly), large negative drifts (>50%) are suppressed:
- `logs/error_count`, `logs/warn_count`, `logs/error_rate_pct`
- `db/slow_queries`, `db/deadlocks`, `db/waiting_conn`, `db/xact_rollback`, `db/buffers_backend`, `db/checkpoint_write_time`
- `proxy/request_server_errors`, `proxy/handshake_failures`, `proxy/rl_requests_blocked`, `proxy/request_client_errors`
- `io/read_time`, `io/write_time`
- `replication/replay_lag_ms`, `replication/flush_lag_ms`
- `statement/temp_blks_written`
- `table/dead_tup_ratio` (a drop means vacuum worked)

**Cross-domain metric correlation** — When 2+ anomalies from different domains co-occur in the same detection cycle, a `correlation` insight is generated linking them (e.g., API latency spike + DB cache drop + proxy error increase noted as a single correlated event). Rate-limited to one correlation insight per 30 minutes.

**Log-metric cross-correlation** — When a log error spike coincides with metric anomalies from other domains, a `log_metric_correlation` insight is generated with causal hypothesis tagging. The system identifies likely root causes based on which domains are affected (e.g., "Database performance issue may be propagating to application errors" when log spikes co-occur with DB anomalies). Rate-limited to one per 30 minutes.

**SLO tracking** (runs on `agg_tick`, hourly) — Data-driven evaluation of all SLO definitions in `collector.slo_definitions`. Each SLO specifies domain, metric, target value, comparison operator (gte/lte), and error budget target percentage. Ten seeded SLOs with 30-day sliding window.

`api_error_rate` and `api_uptime` SLOs subtract `waf_blocked_requests` from `failed_requests` before computing the error rate and uptime ratios. WAF IP-blocklist and bot-blocklist blocks are security events (the service responded correctly); excluding them prevents bot attacks from generating false SLO breaches.

| SLO | Domain | Target | Comparison | Budget |
|-----|--------|--------|------------|--------|
| `api_uptime` | api | 99.9% | gte | 99.9% |
| `api_latency_p95` | api | 500ms | lte | 99.9% |
| `api_latency_p99` | api | 2000ms | lte | 99.0% |
| `api_error_rate` | api | 1% | lte | 99.0% |
| `db_cache_hit` | db | 99% | gte | 99.0% |
| `db_deadlocks` | db | 0 | lte | 100% |
| `proxy_latency_p95` | proxy | 500ms | lte | 99.0% |
| `proxy_uptime` | proxy | 99.9% | gte | 99.9% |
| `system_cpu` | system | 80% | lte | 95.0% |
| `system_memory` | system | 85% | lte | 95.0% |

Computes error budget as `violations / allowed_violations * 100`. Generates `slo_violation` insights and `slo_breach` alerts when SLOs are not met. Skipped for the first 2 minutes after collector restart to let a few collection cycles populate fresh data. New SLOs can be added by inserting rows into `slo_definitions`.

**Health scores** (runs on `intel_tick`, every 5 min) — Per-domain composite health score (0–100) computed as a weighted average of baseline z-score components and breached SLO components. Each baselined metric in a domain gets a component score: 100 (|z| < 1), 80 (|z| < 1.5), 60 (|z| < 2), 40 (|z| < 2.5), 20 (|z| < 3), 0 (|z| >= 3) — weight 1 each. "Lower is better" metrics (latency, errors, CPU) only penalize positive z-scores (spikes). Breached SLOs (met = false) for the domain add weighted penalty components (weight 2): score 50 if budget ≤ 200%, 30 if budget ≤ 400%, 10 if budget > 400%. Only SLOs that are actively breached contribute — SLOs within budget (even at 100% consumed) do not penalize. Domain score = `round(weighted_sum / weighted_count)`. Stored in `collector.health_scores` with JSON component breakdown.

Three special-case overrides prevent false-positive health penalties:
- **Cumulative DB counters excluded**: `xact_rollback`, `wal_bytes`, `wal_sync_time`, `checkpoint_write_time`, and `checkpoint_sync_time` are monotonically increasing `pg_stat` accumulators — their absolute values grow over the lifetime of the cluster, making z-score comparison against a rolling baseline meaningless. These metrics are skipped during health score computation for the `db` domain. (`blks_read` was already excluded.)
- **API `error_rate_pct` uses hourly average**: The raw `api_metrics_raw.error_rate_pct` column stores the cumulative `failed/total` ratio since the last API start. As the total request count grows, this ratio drifts slowly toward zero regardless of current behaviour. Health scoring for `api/error_rate_pct` instead reads `error_rate_avg` from `api_metrics_hourly` (the per-interval average, updated by the restart-safe LAG rollup), which accurately reflects current error rates.
- **WAF blocks excluded from SLO failure counts**: `api_error_rate` and `api_uptime` health/SLO computations subtract `waf_blocked_requests` from `failed_requests`. Requests blocked by the WAF IP-blocklist or bot-blocklist are security responses (the service operated correctly); counting them as failures would cause bot attacks to depress domain health scores and breach error-rate SLOs falsely.

**Capacity predictions** (runs on `intel_tick`, every 5 min) — Linear regression on 24-hour hourly trends for 15 key metrics. Predicts when values will cross critical thresholds within the next 24 hours. Only alerts when R² >= 0.3 (reasonable trend confidence) and the current value is still below the threshold. Deduplicated to one alert per domain/metric per hour. Stored in `collector.capacity_alerts`.

**Recommendations** — Rule-based checks generating actionable recommendations with auto-cleanup when conditions normalize. Deduplication uses `ON CONFLICT (category, target, title)` — all recommendation titles must be stable across ticks (no dynamic values like averages or counts that change every cycle).

The vacuum-check DISTINCT ON queries use `AND ts > now() - INTERVAL '30 minutes'` to scan only the ~N rows from the most recent collector tick instead of the full history (14-day default), reducing execution time from ~200ms to <1ms. The index recommendation query excludes primary key (`_pkey`), unique constraint (`_key`) indexes by naming convention — avoiding the expensive `pg_constraint` catalog join (removed in favour of naming-convention filters for 18× fewer buffer hits). Uses `AND ts > now() - INTERVAL '30 minutes'` for the same reason.

| Category | Target | Trigger | Severity |
|----------|--------|---------|----------|
| vacuum | `{schema}.{table}` | Dead tuples > 20% (warn), > 50% (crit); requires ≥1000 live rows | warn/critical |
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
| db | replication:{slot} | Replication lag > 1s (warn), > 30s (crit) | warn/critical |
| db | query:{id} | Avg exec time > 500ms (info), > 2000ms (warn) | info/warn |
| db | temp:{id} | Temp blocks spilled > 10000 | info |
| performance | io_read | Avg disk read latency > 50ms | warn |
| performance | io_write | Avg disk write latency > 50ms | warn |
| performance | {table} | High update churn (updates/live > 2.0) | info |
| process | {name} | Memory > 500MB (warn), > 1GB (crit) | warn/critical |
| process | {name} | CPU > 30% (warn), > 80% (crit) | warn/critical |
| process | {name} | FDs > 500 (info), > 1000 (warn) | info/warn |
| process | {name} | Crashed/stopped | critical |
| process | {name} | Recently restarted (uptime < 5 min, known services) | warn |
| index | unused:{name} | Unused indexes > 1MB (excludes `_pkey`, `_key`, `_unique`, constraint types `p`/`u`) | info |
| logs | {source} | Recurring error/warn patterns (>10 occurrences/hour) | info/warn |
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

**File ingestion** — Tracks byte offset, inode, and last file size per source in `log_positions`. On each tick: checks for log rotation via inode change, file shrinkage below offset, or file shrinkage below last known size (handles copytruncate). Reads up to 64KB from saved offset, parses complete lines only, batch INSERTs with multi-row VALUES. Messages longer than 4096 characters are truncated.

**Journal ingestion** — Runs `journalctl -u UNIT -o json --after-cursor=X -n N` as an async subprocess. Handles MESSAGE fields that arrive as byte arrays (ANSI-encoded output from Rust tracing). Strips ANSI escape sequences and extracts level/component from tracing format. First run limits to 100 lines to avoid massive backfill.

**Continuation-line filtering** — When a Rust service uses `tracing_subscriber` at `DEBUG` level with multi-line spans (e.g. SQL queries formatted across multiple lines), journald captures each line as a separate journal entry. Lines that are continuations of a multi-line tracing event share no timestamp prefix and begin with leading whitespace. The ingester skips any message that starts with a space or tab and does not begin with a 4-digit year (ISO timestamp), preventing SQL fragment lines from being stored as individual log entries and inflating the `total_count` baseline. The upstream fix is to set the API's `tracing_subscriber` to `INFO` level with `.with_ansi(false)` — see the [API log level note](#api-log-level) in the Deployment section.

**7 parsers:**
1. **postgresql** — `%m [%p] %q%u@%d LEVEL: message`, handles multi-line STATEMENT continuation, skips empty messages from HINT/DETAIL lines
2. **journalctl** — JSON objects with `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER`, `__CURSOR`; handles byte-array MESSAGE and Rust tracing format
3. **syslog** — RFC 3164 (`Mon DD HH:MM:SS hostname process[pid]: message`) and RFC 5424/ISO timestamps (`2026-02-15T00:00:54.808228-06:00`)
4. **apache_error** — `[timestamp] [module:level] [pid N] message`
5. **simple_timestamp** — `[YYYY-MM-DD HH:MM:SS] message` or `YYYY-MM-DD HH:MM:SS message`
6. **fail2ban** — `YYYY-MM-DD HH:MM:SS,mmm fail2ban.module [pid]: LEVEL message`
7. **certbot** — `YYYY-MM-DD HH:MM:SS,mmm:LEVEL:module:message`

**Fingerprinting** — Normalizes messages (digits to `#`, IPs to `<IP>`, UUIDs to `<UUID>`), hashes with SHA-256 truncated to 16 hex chars: `sha256(source|level|normalized)[..16]`. A per-source cardinality budget (default 1000 unique fingerprints) prevents unbounded memory growth from high-cardinality log sources. New fingerprints beyond the budget are silently dropped. The budget resets hourly on `agg_tick`, with skip counts logged at `warn` level before clearing.

### `src/log_analysis.rs`

Log-specific analysis that runs on `intel_tick` (every 5 minutes), feeding into the existing insights, recommendations, and alerts tables:

**Pattern detection** — Upserts `log_patterns` from error/warn entries in the last 1 hour, grouped by fingerprint, source, and level, requiring at least 2 occurrences. Auto-resolves patterns not seen in 1 hour.

**Error spike detection** — Compares 5-minute error/warn count against the 7-day statistical baseline from `collector.baselines` (falling back to 24h average from `log_metrics_hourly`). Scales baseline to 5-minute window (baseline/12). Requires at least 5 error/warn entries in the 5-minute window. Inserts an `error_spike` insight (domain=`logs`) if count exceeds 3x the expected rate.

**Security event detection:**
- SSH brute force: >5 failed logins from same IP in 5 minutes -> alert (type=`ssh_brute_force`)
- Fail2ban ban actions -> insight (domain=`logs`, metric=`fail2ban_ban`)
- UFW/firewall blocks from same IP >10 in 5 minutes -> insight (domain=`logs`, metric=`firewall_block`)

**Actionable recommendations** — Context-aware pattern matching generates specific remediation steps:
- SSH brute force -> numbered steps: check attacking IPs, block with `ufw`, verify fail2ban, disable password auth
- PostgreSQL connection errors -> check API server, verify `pg_hba.conf`, test connectivity
- PostgreSQL "relation does not exist" -> check migration status, verify table names
- Deadlock detection -> review transaction ordering, check long-running queries
- Permission denied errors -> check file ownership, verify service user permissions
- Disk/IO errors -> check filesystem health, review SMART status
- Connection/timeout errors -> check service connectivity, verify network
- Collector crash loop detection -> check logs and configuration
- NOUSER shadow lookup errors -> check user/group configuration
- Recurring error patterns (>10 occurrences/hour) -> source-specific recommendations with diagnostic commands

**Cross-domain log correlation** — Detects when log error spikes coincide with metric anomalies (e.g., API error spike in logs at the same time as latency anomaly in metrics).

**Hourly aggregation** (on `agg_tick`) — Rolls up `log_entries` into `log_metrics_hourly` (source, level, count per hour). Runs `cleanup_log_data()` for retention enforcement.

### `src/watchdog.rs`

Health monitoring runs on a dedicated 30s tick, decoupled from the 5s heartbeat:
- **Heartbeat** (5s): Inserts liveness row into `collector.heartbeat`
- **Staleness check** (30s): Alerts if last heartbeat exceeds 6x heartbeat interval
- **Table health** (30s): Detects bloated tables (dead tuples > 20% with > 1000 live rows and > 500 dead rows) and tables not vacuumed in 7+ days
- **Process health** (30s): Checks for high CPU (> 50%), high memory (> 3GB RSS), high FDs (> 1000), high threads (> 500), bad states (zombie/D-state/stopped), and expected processes that are missing. The 3 GB threshold accounts for `pqcrypta-api`'s baseline RSS (~2.4 GB with 31 cryptographic engine libraries and ML models loaded at startup). Condition-based auto-resolve: when a process metric returns to healthy, its alert is resolved immediately (no time-based delay).
- **API error rate** (30s): Alerts if error rate exceeds 5% with > 100 total requests. Auto-resolves when error rate drops below threshold.
- **DB response time** (30s): Alerts if 5-minute average DB response time exceeds 100ms. Auto-resolves when response time drops below threshold.
- **Long queries** (30s): Queries `pg_stat_activity` for queries running > 30 seconds. Records `long_query` alert with PID, duration, and truncated query text. Auto-resolves when no long queries detected.
- **Connection trend** (30s): Compares current active connection count to 1 hour ago. Alerts `connection_leak` if connections increased by >50% AND current count exceeds 80% of `max_connections`. Auto-resolves when condition clears.
- **IO pressure** (30s): Checks for `load_1 > 8.0` combined with high checkpoint write time (>1000ms) or high buffers_backend (>100). Records `io_pressure` alert when both CPU and IO conditions are met. Auto-resolves when conditions clear.

All watchdog alerts use deduplication (matching alert type + subject prefix) and have both condition-based and time-based auto-resolution fallbacks.

### `src/config.rs`

TOML configuration with environment variable overrides. Default path: `/etc/pqcrypta/collector.toml` (override via `COLLECTOR_CONFIG` env var). Supports hot-reload: every 60 seconds the collector checks the config file mtime and reloads safe fields (batch_size, query_timeout_secs, system_secs, app_secs, raw_days, hourly_days, daily_days) without restart. DB credentials, heartbeat interval, processes, scrape URLs, log config, queue_dir, queue_max_mb, and max_fingerprints_per_source require a full restart.

```toml
[database]
host = "localhost"        # env: DB_HOST
port = 5432               # env: DB_PORT
name = "mydb"             # env: DB_NAME
user = "myuser"           # env: DB_USER
password = ""             # env: DB_PASS

[intervals]
system_secs = 10          # env: SYSTEM_INTERVAL_SECS
app_secs = 10             # env: APP_INTERVAL_SECS
heartbeat_secs = 5        # env: HEARTBEAT_INTERVAL_SECS

[scrape]
api_metrics_url = "http://127.0.0.1:3003/metrics"      # env: API_METRICS_URL
proxy_metrics_url = "http://127.0.0.1:8082/metrics/json"  # env: PROXY_METRICS_URL

[retention]
raw_days = 14             # env: RAW_RETENTION_DAYS
hourly_days = 90          # env: HOURLY_RETENTION_DAYS
daily_days = 365          # env: DAILY_RETENTION_DAYS

[collector]
batch_size = 50           # env: BATCH_SIZE
query_timeout_secs = 10   # env: QUERY_TIMEOUT_SECS — per-query timeout to prevent stuck queries blocking the event loop
queue_dir = "/var/lib/pqcrypta-collector/queue"  # env: QUEUE_DIR — disk spill directory
queue_max_mb = 100        # env: QUEUE_MAX_MB — max disk budget for spill queue
max_fingerprints_per_source = 1000  # env: MAX_FINGERPRINTS_PER_SOURCE — cardinality limit per log source
processes = ["pqcrypta-proxy", "pqcrypta-api", "pqcrypta-collector", "postgres", "apache2", "php-fpm"]

[logs]
enabled = true                # env: LOG_ENABLED
tick_secs = 15                # env: LOG_TICK_SECS
batch_size = 100              # max rows per INSERT
max_lines_per_tick = 500      # cap per tick across all sources
chunk_size = 65536            # bytes to read per file source
```

## Database Schema

Seven migration files in `migrations/`:

**001_collector_schema.sql** — Core tables:
- `collector.system_metrics_raw` — Host CPU, memory, load, swap, network, disk JSONB (15 columns)
- `collector.process_metrics_raw` — Per-process CPU, RSS, VSZ, FDs, threads, state, uptime (10 columns)
- `collector.api_metrics_raw` — API request counts, latency percentiles, errors, `waf_blocked_requests`, cache stats, DB response time (21 columns)
- `collector.proxy_metrics_raw` — Proxy connections, TLS stats, rate limiting, upstream latency (28 columns)
- `collector.db_metrics_raw` — PostgreSQL connection pool, transaction counts, cache ratios, slow queries (16 base columns)
- `collector.heartbeat` — Collector liveness tracking
- `collector.alerts` — Alert storage with deduplication and resolution tracking
- `collector.system_metrics_hourly` — Hourly system aggregates (CPU, load, memory, network)
- `collector.process_metrics_hourly` — Hourly per-process aggregates (CPU, RSS, FDs, threads)
- `collector.api_metrics_hourly` — Hourly API aggregates (RPS, latency percentiles, error rate, request deltas including `waf_blocked_requests`, DB response time)
- `collector.db_metrics_hourly` — Hourly DB aggregates (connections, cache ratio, transaction counts, deadlocks, slow queries)
- `collector.system_metrics_daily` — Daily system aggregates

**002_extended_pg_metrics.sql** — Extended PostgreSQL monitoring:
- Adds bgwriter (9 cols), WAL (8 cols), wait event (9 cols), and lock (9 cols) columns to `db_metrics_raw` (51 total columns)
- `collector.table_metrics_raw` — Per-table live/dead tuples, seq/idx scans, modifications, autovacuum timing
- `collector.index_metrics_raw` — Per-index scans, reads, fetches, size
- `collector.io_metrics_raw` — Per-backend-type IO statistics from `pg_stat_io` (PG16+): reads, writes, hits, evictions, fsyncs with timing
- `collector.replication_metrics_raw` — Replication state, write/flush/replay lag, sent/write/flush/replay LSN
- `collector.statement_metrics_raw` — Top N statements by total time (calls, rows, mean/total time, shared block stats)
- Corresponding hourly aggregate tables for table and IO metrics

**003_intelligence_schema.sql** — Intelligence layer:
- `collector.baselines` — Statistical baselines (domain, metric, metric_key, time_window, mean, stddev, p5, p25, p50, p75, p95, sample_count, updated_at)
- `collector.insights` — Detected anomalies, drift events, correlations, SLO violations (insight_type, severity, domain, metric, metric_key, current_value, baseline_mean, baseline_stddev, z_score, drift_pct, message, resolved, expires_at)
- `collector.recommendations` — Actionable recommendations (category, severity, target, title, description, action_sql, acknowledged, expires_at)
- `collector.slo_tracking` — SLO computation results (slo_name, target, actual, met, budget_consumed, violations, total_periods)
- `collector.slo_definitions` — Data-driven SLO configuration (10 seeded). Columns: slo_name, domain, metric, target_value, comparison, target_pct, enabled, description.
- `collector.health_scores` — Per-domain composite health scores (0–100) with JSON component breakdown, computed every 5 minutes from baseline z-scores
- `collector.capacity_alerts` — Predictive threshold breach alerts from linear regression on 24h trends (domain, metric, current_value, predicted_value, threshold, hours_until, confidence, message)

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

**006_intelligence_v2.sql** — Intelligence v2 tables:
- `collector.baselines_hourly` — Seasonal baselines with 168 hour-of-week slots (domain, metric, metric_key, hour_of_week, mean, stddev, sample_count)
- `collector.trend_forecasts` — Trend forecast storage for capacity prediction history
- `collector.capacity_alerts` — Predictive threshold breach alerts (domain, metric, current_value, predicted_value, threshold, hours_until, confidence, message)
- `collector.health_scores` — Per-domain composite health scores (domain, score float8, components jsonb)

**007_collector_self_metrics.sql** — Collector self-monitoring:
- `collector.collector_self_metrics` — Per-tick telemetry about the collector process itself: PID, uptime, per-buffer depths (10 buffers), total buffer depth, flush count, spill count, disk queue bytes, DB health status, tick duration (ms), total rows written. Indexed by `ts DESC`. Same raw retention as other metrics tables.

**008_waf_blocked_metrics.sql** — WAF block counter columns:
- Adds `waf_blocked_requests bigint NOT NULL DEFAULT 0` to `collector.api_metrics_raw` and `collector.api_metrics_hourly`. Stores the count of requests rejected by the WAF IP-blocklist and bot-blocklist separately from `failed_requests` so that bot attacks cannot cause false SLO breaches for `api_error_rate` and `api_uptime`.

All tables use `ts TIMESTAMPTZ` as the primary time column with descending indexes for efficient latest-value queries.

### Query-optimisation indexes (applied 2026-02-20)

Applied directly to the live database; no migration file required for these performance indexes:

| Index | Table | Type | Purpose |
|-------|-------|------|---------|
| `idx_index_raw_schema_table_name_ts` | `index_metrics_raw` | btree composite | Supports DISTINCT ON sort on `(schema_name, table_name, index_name, ts DESC)` |
| `idx_index_raw_unused_large` | `index_metrics_raw` | partial (`idx_scan=0 AND size>1MB`) | Fast path for unused-index recommendation scans |
| `idx_table_raw_schema_table_ts_desc` | `table_metrics_raw` | btree composite | `(schema_name, table_name, ts DESC)` — eliminates incremental sort on DISTINCT ON queries |
| `idx_table_raw_live_positive_schema_table_ts` | `table_metrics_raw` | partial (`n_live_tup > 0`) | Vacuum-check DISTINCT ON with ts-bounded filter |
| `idx_table_raw_live_schema_table_ts` | `table_metrics_raw` | partial (`n_live_tup > 100`) | Monitor dashboard DISTINCT ON with live-tuples filter |
| `idx_table_raw_tablename_ts_desc` | `table_metrics_raw` | btree | `(table_name, ts DESC)` — direct seeks for single-table dead-ratio lookups |
| `idx_key_vault_created_by_cleanup` | `key_vault` | btree | `(created_by, usage_count, created_at)` — key cleanup DELETE scans |
| `idx_log_fph_level_bucket` | `log_fingerprint_hourly` | btree | `(level, bucket DESC)` — level-filtered fingerprint aggregation |

## Deployment

### systemd

A service unit is provided in `pqcrypta-collector.service`:

```ini
[Unit]
Description=PQCrypta Metrics Collector
After=postgresql.service pqcrypta-api.service
Wants=postgresql.service

[Service]
Type=simple
ExecStart=/var/www/html/public/ent/target/release/pqcrypta-collector
Restart=on-failure
RestartSec=10
MemoryMax=64M
CPUQuota=5%
Environment=RUST_LOG=pqcrypta_collector=info
Environment=COLLECTOR_CONFIG=/etc/pqcrypta/collector.toml

[Install]
WantedBy=multi-user.target
```

### API log level

The API service (`pqcrypta-api`) must run with `tracing_subscriber` at `INFO` level and ANSI output disabled. Using `DEBUG` level causes multi-line tracing spans (e.g. SQL statements) to be emitted to stderr across multiple lines; journald records each line as a separate entry, and the collector ingests all of them, inflating the `logs/total_count` baseline by 3× or more.

Required configuration in `api/src/main.rs`:

```rust
tracing_subscriber::fmt()
    .with_max_level(tracing::Level::INFO)
    .with_target(true)
    .with_thread_ids(true)
    .with_line_number(true)
    .with_ansi(false)       // no ANSI escape codes in journald
    .init();
```

### Build

```bash
cargo build --release
```

The service file runs the binary from `target/release/` directly. Alternatively, copy to a system path and update `ExecStart`.

### Prerequisites

- PostgreSQL 15+ with the `collector` schema created
- Run migrations in order: `001_collector_schema.sql`, `002_extended_pg_metrics.sql`, `003_intelligence_schema.sql`, `004_log_tables.sql`, `005_log_enhancements.sql`, `006_intelligence_v2.sql`, `007_collector_self_metrics.sql`, `008_waf_blocked_metrics.sql`
- API server running on port 3003 with `/metrics` endpoint
- Proxy server running on port 8082 with `/metrics/json` endpoint (optional)
- Read access to log files in `/var/log/` (auth.log, kern.log, postgresql, apache2, fail2ban, letsencrypt, pqcrypta-proxy)
- `journalctl` available for systemd journal sources (pqcrypta-api, pqcrypta-proxy, pqcrypta-collector)

## Resilience

The collector is hardened for production reliability:

- **Graceful DB reconnection**: If the PostgreSQL connection drops, the collector continues collecting metrics in memory. A `reconnect_tick` (5s) attempts reconnection using a `tokio::sync::watch` health channel. On reconnect, schema is verified and buffered data is flushed. No `std::process::exit` — the process stays alive.
- **Disk-backed durable queue**: When in-memory ring buffers overflow during a DB outage, evicted records are serialized to a JSONL file on disk (default 100MB budget) instead of being dropped. On successful DB reconnection, spilled records are drained back and flushed in batches. At shutdown with an unhealthy DB, all in-memory buffers are spilled to disk for recovery on next startup.
- **Ring buffer backpressure**: All 10 metric buffers use `VecDeque` with `max_capacity = batch_size × 20`. When a buffer hits capacity during a DB outage, the oldest entry is spilled to the disk queue. Spill counts are logged on the next successful flush.
- **Transaction-wrapped flushes**: All INSERTs in a flush cycle are wrapped in `BEGIN`/`COMMIT`. On any error, `ROLLBACK` is executed and buffers are restored so no data is lost.
- **Query timeout protection**: Every DB query uses `tokio::time::timeout` (default 10s, configurable via `query_timeout_secs`). Prevents stuck queries from blocking the single-threaded event loop.
- **Config hot-reload**: Every 60s the collector checks the config file mtime and reloads safe fields (batch_size, query_timeout, intervals, retention days) without restart.
- **Log rotation detection**: Handles both standard log rotation (inode change) and copytruncate rotation (file size shrinkage between ticks) via `last_file_size` tracking.
- **Baseline NULL filtering**: Statistical baselines skip NULL, NaN, and Inf values to prevent pollution from incomplete data windows.
- **Cardinality protection**: Per-source fingerprint budget (default 1000) prevents log ingestion from creating unbounded unique entries. Per-table baseline computation is limited to the top 200 tables by activity. Stale log patterns (>7d) and baselines (>30d) are pruned hourly.

## Security Model

### Connection Security

The collector operates entirely on localhost with no listening ports:
- **Database**: Connects to PostgreSQL on `localhost:5432` via Unix domain socket or TCP loopback. No remote DB connections by default.
- **HTTP scraping**: Fetches metrics from `127.0.0.1:3003` (API) and `127.0.0.1:8082` (proxy) — loopback only, no external network access.
- **Journal access**: Reads from local systemd journal via `journalctl` subprocess.
- **File access**: Reads log files from local filesystem (`/var/log/`).
- **No listening sockets**: The collector binary does not bind any ports or accept any inbound connections.

### Secret Handling

- **Config file**: `/etc/pqcrypta/collector.toml` with recommended permissions `0600 root:root`. Contains database credentials.
- **Environment variable overrides**: All sensitive fields (DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASS) can be set via environment variables, avoiding config file storage entirely.
- **systemd integration**: Environment variables can be set in the service unit file or via `EnvironmentFile=` pointing to a restricted credentials file.
- **No hardcoded secrets**: Zero credentials in source code. All secrets come from config file or environment at runtime.
- **Memory handling**: Database password is held in a `String` field — not persisted to disk beyond the config file.

### Query Safety

- **Parameterized queries**: All database writes use `$1, $2, ...` parameterized queries via `tokio-postgres`. No string interpolation of user-controlled data into SQL.
- **Query timeouts**: Every database operation is wrapped in `tokio::time::timeout` (default 10s, configurable) to prevent stuck queries from blocking the event loop.
- **Schema-qualified tables**: All table references use the `collector.` schema prefix, preventing accidental cross-schema access.
- **Transaction wrapping**: All flush operations use explicit `BEGIN`/`COMMIT`/`ROLLBACK` for atomicity. Failed writes trigger rollback and buffer restoration.
- **Read-only external access**: HTTP metric scraping and log file reads are read-only operations. The collector never writes to external services.

### Data Classification

| Data Type | Sensitivity | Retention | Notes |
|-----------|-------------|-----------|-------|
| System metrics (CPU, memory, load) | Low | 14d raw, 90d hourly, 365d daily | No PII |
| Process metrics (names, PIDs, CPU, RSS) | Low | 14d raw, 90d hourly | Process names only, no arguments |
| API/Proxy metrics (latency, error rates) | Low | 14d raw, 90d hourly | Aggregate counters, no request content |
| Database metrics (connections, cache, WAL) | Low | 14d raw, 90d hourly | Statistical aggregates only |
| Log entries (messages, timestamps) | Medium | 7d | May contain IPs, usernames, error details |
| Log fingerprints | Low | 30d after resolved | SHA-256 hashes of normalized messages |
| Security events (IPs, ban actions) | Medium | 30d | Source IPs from fail2ban/auth logs |
| Baselines/anomalies | Low | 30d stale pruning | Statistical summaries |
| Database credentials | High | Runtime only | In config file or env vars |

### Threat Model

**In scope:**
- **Database credential exposure**: Mitigated by file permissions (0600), env var overrides, and localhost-only connections.
- **PII in log messages**: Log messages may contain IP addresses, usernames, or error context. Mitigated by 7-day retention, fingerprint normalization (IPs → `<IP>`, UUIDs → `<UUID>`), and message truncation (4096 char limit).
- **Config file exposure**: Mitigated by restrictive file permissions and systemd sandboxing.
- **Cardinality DoS**: Malicious or runaway log sources could generate unbounded unique fingerprints. Mitigated by per-source fingerprint budget (default 1000), per-table baseline limit (top 200), and stale cardinality pruning.
- **Disk exhaustion from spill queue**: Mitigated by configurable disk budget (default 100MB) with hard cap enforcement.

**Out of scope:**
- **Host compromise**: If an attacker has root access to the host, all bets are off. The collector assumes the host OS is trusted.
- **Network-level attacks**: The collector has no listening ports and makes only localhost connections. Network attacks require compromising the loopback interface.
- **PostgreSQL server compromise**: The collector trusts the database server. A compromised PostgreSQL instance could return malicious data, but the collector only reads statistical views.

### Hardening Recommendations

- **File permissions**: Ensure `/etc/pqcrypta/collector.toml` is `0600 root:root`. Ensure the queue directory (`/var/lib/pqcrypta-collector/queue`) is `0700` owned by the service user.
- **Dedicated DB user**: Use a dedicated `pqcrypta_collector` database user with minimal privileges: `CONNECT` on the database, `USAGE` and `CREATE` on the `collector` schema, `SELECT` on `pg_stat_*` views, `INSERT`/`UPDATE`/`DELETE` on collector tables.
- **TLS for remote DB**: If the database is on a separate host, configure `sslmode=verify-full` in the connection string and provide CA certificates.
- **Disk encryption**: Use LUKS or dm-crypt for the queue directory partition to protect spilled metrics at rest.
- **systemd sandboxing**: The provided service unit includes `MemoryMax=64M` and `CPUQuota=5%`. Consider adding:
  ```ini
  ProtectSystem=strict
  ProtectHome=true
  ReadWritePaths=/var/lib/pqcrypta-collector
  ReadOnlyPaths=/etc/pqcrypta /var/log
  PrivateTmp=true
  NoNewPrivileges=true
  ```
- **Log rotation**: Ensure all monitored log files have rotation configured (logrotate) to prevent unbounded growth. The collector handles both standard rotation (inode change) and copytruncate rotation.

## Resource Usage

The collector is designed to be lightweight despite 10s collection intervals:
- Single-threaded async runtime (`tokio` current-thread)
- Memory capped at 64MB via systemd `MemoryMax`
- CPU capped at 5% via systemd `CPUQuota`
- Batched database writes minimize connection overhead (single DB connection, no pool)
- Fast-path metrics (sys, app) read from kernel shared memory and localhost HTTP — no disk I/O
- Heartbeat decoupled from heavier watchdog checks to keep liveness detection fast
- All intervals are configurable via TOML or environment variables

## Monitor Dashboard

The collector's data is surfaced through a real-time web dashboard at `/monitor/`. The dashboard auto-refreshes every 30 seconds and is organised into 12 tabs, each backed by a dedicated `api.php` mode that queries the `collector` schema. All charts use a custom canvas-based renderer (`MC.lineChart`, `MC.barChart`, `MC.pieChart`, `MC.gauge`, `MC.stackedBar`) — no external charting library. `MonTables.enhance()` wraps every sortable/filterable table with in-page pagination and live search.

| Tab | API mode | Primary data sources |
|-----|----------|----------------------|
| System | `full` | `system_metrics_raw`, `system_metrics_hourly`, `alerts`, `health_scores` |
| Processes | `full` | `process_metrics_raw`, `process_metrics_hourly` |
| API & Proxy | `full` | `api_metrics_raw`, `api_metrics_hourly`, `proxy_metrics_raw` |
| PostgreSQL | `full` | `db_metrics_raw`, `db_metrics_hourly`, `alerts` |
| Tables & Indexes | `full` | `table_metrics_raw`, `index_metrics_raw` |
| Queries | `full` | `statement_metrics_raw`, `io_metrics_raw` |
| Logs | `logs` | `log_entries`, `log_metrics_hourly`, `log_patterns`, `security_events` |
| Alerts | `full` | `alerts`, `heartbeat` |
| Insights | `full` | `insights`, `baselines`, `recommendations`, `slo_tracking`, `health_scores`, `capacity_alerts` |
| Replication | `full` | `replication_metrics_raw` |
| Collector | `full` | `collector_self_metrics`, `process_metrics_raw`, `heartbeat` |
| Event Snapshot | `snapshot` | all raw metric tables, `alerts`, `insights` |

### System tab

Overview of host-level resource health. Stat cards show the latest values with colour-coded thresholds (warn/critical) and trend arrows derived from baseline z-scores:

- **CPU** — combined user + system %, idle %. Warn ≥ 70%, critical ≥ 80%.
- **Memory** — used bytes, available bytes, swap used/total. Mem warn ≥ 75%, critical ≥ 85%; swap warn ≥ 50%.
- **Load averages** — 1/5/15-minute load. Colour-codes against baseline.
- **Network** — latest `net_rx` / `net_tx` byte totals from `sysinfo`.
- **Disk** — per-mount usage card grid (filtered to exclude snap.rootfs and pseudo-mounts). Cards turn red ≥ 90%, yellow ≥ 75%. Horizontal bar chart shows all mounts at a glance.

Time-series charts cover CPU %, memory used vs available (GB), all three load averages, and network RX/TX rate (bytes/s computed as delta ÷ 10s interval). The network chart computes deltas between successive raw samples, clamping negative deltas (counter resets) to zero.

### Processes tab

Per-process metrics from `/proc/{pid}/stat` and `/proc/{pid}/fd`. Issues are classified across five dimensions and colour-coded per column:

| Dimension | Info | Warn | Critical |
|-----------|------|------|----------|
| CPU % | ≥ 20% | ≥ 50% | ≥ 80% |
| RSS | ≥ 256 MB | ≥ 512 MB | ≥ 1 GB |
| File descriptors | ≥ 200 | ≥ 500 | ≥ 1000 |
| Threads | ≥ 100 | ≥ 200 | ≥ 500 |
| State | — | T (stopped) | Z (zombie) / D (uninterruptible) |

Processes are sorted highest-severity-first, then by CPU descending within the same severity. A filter bar above the table lets you narrow to any single issue dimension (Issues / CPU / RSS / FDs / Threads / State). The filter selection persists in `localStorage`. The tab badge and each filter button show a live count of affected processes, colour-coded to the worst severity seen. `MonTables` search re-counts the badge to reflect the currently visible rows rather than the full dataset.

Columns: process name, PID, CPU %, RSS, VSZ, FD count, thread count, state, uptime.

### API & Proxy tab

Split into two sections: API server (port 3003) and reverse proxy (port 8082).

**API section** — stat cards: uptime, RPS, total/successful/failed requests, error rate %, average response ms, active connections, active sessions, CPU %, memory, throughput MB/s, DB connections, DB response time ms, DB cache hit ratio. DB response > 50 ms turns amber, > 100 ms turns red. Cache hit < 99% turns amber, < 95% turns red.

Two time-series charts:
- **RPS & Error Rate** — dual series (RPS on primary axis, error rate % on secondary) with threshold zones at 1% (warn) and 5% (critical) error rate.
- **Latency percentiles** — p50/p95/p99 ms with reference lines at 200 ms (good), 500 ms (SLO), 1000 ms (slow).

**Per-endpoint error breakdown** — fetched separately from `api.php?mode=api_errors` with a 30-second client-side cache. Renders a two-row grouped header (Client Errors 4xx / Server Errors 5xx) with individual status code columns (400, 401, 403, 404, 405, 408, 409, 413, 422, 429, 500, 502, 503, 504). Each row shows endpoint path, total failures, per-code counts, last seen timestamp, and share of total failures. Clickable badge filters narrow by last HTTP status code.

**Recent failures table** — the 100 most recent individual failed requests with path, status code, timestamp. Badge filter by status code.

**Proxy section** — stat cards: uptime, total/success/client-error/server-error/in-progress request counts, bytes received/sent, latency p50/p95/p99, connection counts (HTTP/3, HTTP/2, HTTP/1.1, WebTransport), TLS stats (total/PQC/classical handshakes, PQC enabled flag), rate-limit stats (checked/allowed/limited/blocked), active connections, handshake failures.

### PostgreSQL tab

Deep PostgreSQL health view backed by `db_metrics_raw` which aggregates `pg_stat_*` catalog views.

- **Cache hit gauge** — radial gauge (0–100%) with colour zones: red < 90%, amber < 97%, green ≥ 97%.
- **Wait events pie** — IO / Lock / LWLock / BufferPin / Activity / Client / IPC / Timeout segments; empty segments hidden.
- **Lock distribution bar** — horizontal bar chart across seven PostgreSQL lock modes (AccessShare → AccessExclusive).
- **Connection cards** — active, idle, waiting connections. Waiting > 1 turns amber, > 5 turns red.
- **Transaction cards** — commit and rollback counts. Deadlocks > 0 turns red; slow queries > 1 turns amber, > 5 turns red.
- **Checkpoint & WAL cards** — timed/requested checkpoints, checkpoint write/sync time, buffers written by checkpoint/cleaner/backends, WAL records/bytes/full-page images, WAL write/sync time.
- **Transaction trend chart** — commits vs rollbacks over the history window.

### Tables & Indexes tab

Per-table and per-index stats from `pg_stat_user_tables` and `pg_stat_user_indexes`.

**Tables table** — schema.table, total size, live tuples, dead tuples, dead tuple %, seq scans, index scans, last autovacuum, last autoanalyze. Dead tuple % ≥ 10% turns amber, ≥ 20% turns red.

**Vacuum needed table** — tables where dead tuples exceed 10% of live tuples and live count > 100, showing dead count, ratio, and time since last autovacuum.

**Unused indexes table** — indexes with zero scans sorted by size descending. Shows schema.table, index name, size, scan count. Helps identify candidates for removal.

All three sub-tables are independently sortable and searchable via `MonTables`.

### Queries tab

Top SQL statements from `pg_stat_statements` plus I/O breakdown from `pg_stat_io` (PG16+).

**Top queries table** — rank, total execution time, call count, mean exec time, max exec time, rows returned, shared blocks hit/read, temp blocks read/written, block read/write time, WAL records, query text (truncated to 120 chars, full text in tooltip). Query text is sanitised server-side to redact password/secret/token literals before display. Mean > 500 ms turns amber, > 1000 ms turns red. Max > 2000 ms turns amber, > 5000 ms turns red. Temp blocks > 0 turns amber, > 1000 turns red.

**I/O by backend type** — horizontal stacked bar chart aggregating reads, writes, and hits across all backend types (client backend, autovacuum worker, WAL sender, background writer, etc.).

**I/O summary cards** — total read time, write time, evictions, fsyncs, and buffer hit ratio aggregated across all backend types.

**I/O detail table** — per-row breakdown of backend type, object, context, reads, read time, writes, write time, hits, evictions, reuses, fsyncs, fsync time.

### Logs tab

Log data is fetched separately via `api.php?mode=logs` with a 30-second client-side cache. The tab is split into eight sub-sections:

- **Log health gauge** — radial gauge (0–100%) derived from the error/warn ratio in the most recent hour.
- **Error distribution pie** — proportion of error vs warn vs info log entries.
- **Source health grid** — one card per log source showing entry count, error rate, and health status.
- **Error rate trend chart** — hourly error/warn/total counts over the past 24 hours from `log_metrics_hourly`.
- **Security events table** — SSH brute force, fail2ban ban actions, UFW/firewall blocks from `security_events`. Filterable by event type badge.
- **Top error patterns table** — recurring fingerprints from `log_patterns` with occurrence count, first/last seen, and sample message. Filterable by source and level via dual badge filter.
- **Log volume chart** — stacked hourly volume by level (error/warn/info) over 24 hours.
- **Log entries table** — up to 500 most recent raw log entries (50 per source, merged and sorted newest-first). Columns: ts, source, level, component, message. Filterable by source and level badge filters; full-text searchable via `MonTables`.

### Alerts tab

Active alerts from the watchdog and intelligence engine.

**Alerts table** — timestamp, alert type, component, message. Alert type badges allow one-click filtering by type (staleness, table_health, process_health, api_error_rate, db_response_time, long_query, connection_leak, io_pressure, slo_violation, etc.). Empty table shows green "No active alerts" confirmation.

**Heartbeat table** — recent heartbeat records from `collector.heartbeat` showing timestamp, component, and status (ALIVE / other). Status is colour-coded green/red. Filterable by component badge.

Both tables use `MonTables` for pagination and search.

### Insights tab

The intelligence layer's output in a single view. Six distinct sections:

**Insight summary cards** — clickable cards for total insight count, per-severity counts (critical/warn/info), and per-insight-type counts. Clicking a card applies a filter to the Active Insights table below.

**Health scores** — per-domain composite scores (0–100) displayed as colour-coded stat cards for system, api, db, proxy, logs, process, io, replication, and statement domains. Score < 40 turns red, < 70 turns amber.

**Active Insights table** — anomalies detected in the last 7 days: timestamp, insight type, severity, domain, metric key, current value, baseline mean, z-score, drift %, message. Rows are clickable — clicking any row opens the **Event Snapshot** tab scoped to that anomaly's timestamp. Severity column colour-coded (critical = red, warn = amber, info = green). Row background accent for `correlation` and `log_metric_correlation` types.

**Baselines table** — the 43 statistical baselines across 9 domains: domain, metric, 7-day mean, stddev, p50, p95, sample count, last updated. Filterable by domain badge. Gives a snapshot of what "normal" looks like for each metric.

**Recommendations table** — active actionable recommendations with category, severity, target, title, and description. Category badges (vacuum/system/api/proxy/db/performance/process/index/logs/security) and severity badges allow independent filtering. Badge counts update immediately on click without a data re-fetch.

**SLO tracking table** — per-SLO actual vs target values, met/not-met status, error budget consumed, violation count, and evaluation period count. Colour-codes the met column green/red.

**Capacity predictions table** — linear regression forecasts from `capacity_alerts`: domain, metric, current value, predicted value, threshold, hours until breach, R² confidence, message. Only shown when predictions exist.

### Replication tab

Per-slot replication health from `pg_stat_replication`.

**Summary cards** — slot count, maximum replay lag ms (warn ≥ 100 ms, critical ≥ 1000 ms), maximum flush lag ms (same thresholds), WAL status summary.

**Slots table** — slot name, type, active (green/red), client address, state, WAL status, replay lag ms, flush lag ms, write lag ms. All three lag columns are colour-coded (amber ≥ 100 ms, red ≥ 1000 ms). Sortable and searchable via `MonTables`. Hidden with an empty-state message when no replication slots exist.

The tab badge shows the count of slots with replay lag ≥ 1000 ms, coloured amber when non-zero.

### Collector tab

Self-monitoring for the collector binary itself using `collector_self_metrics` (written by the collector each tick) and cross-referenced against `process_metrics_raw`.

**Overview cards** — running status (green/red), PID, uptime, binary size, binary build time (mtime), total rows written lifetime.

**Resource usage cards** — CPU % (warn ≥ 3%, critical ≥ 5%), RSS (warn ≥ 48 MB, critical ≥ 60 MB), VSZ, file descriptor count (warn ≥ 50, critical ≥ 200), thread count (warn ≥ 4, critical ≥ 10), process state (Z/T = red, D = amber). Cards highlight with a border accent when thresholds are breached.

**Resource trend charts** — CPU % and RSS over the history window, from `process_metrics_raw` for the collector PID.

**Operational health cards** — total buffer depth (warn ≥ 500, critical ≥ 800), flush count, spill count (any spill > 0 highlights amber — indicates DB was unavailable), disk queue bytes (any non-zero highlights amber), DB connection health (healthy/unhealthy), tick duration ms (warn ≥ 5 s, critical ≥ 9 s).

**Buffer breakdown table** — per-buffer depth (System/Process/API/Proxy/DB/Table/Index/IO/Replication/Statements) as an inline progress bar. Bar turns amber ≥ 40% full, red ≥ 80% full. Buffer depths are captured pre-flush so the chart reflects actual accumulation during each collection cycle.

**Tick performance charts** — tick duration ms and total buffer depth over time, useful for identifying slow ticks or accumulation during DB outages.

**Heartbeat monitor cards** — age of last heartbeat (warn > 30 s, critical > 60 s), heartbeat count, gap count. Gaps table shows each detected gap with previous timestamp, gap timestamp, and gap duration in seconds. Empty-state message shown when no gaps exist.

**Configuration table** — hot-reloadable configuration fields read from `collector_self_metrics` (batch size, query timeout, collection intervals, retention days). Allows verifying the live config without accessing the TOML file.

### Event Snapshot tab

The Event Snapshot tab opens a time-windowed dashboard scoped to the exact moment of any detected anomaly. Click any row in the **Active Insights** table (Insights tab) to activate it.

**How it works** — clicking an insight row stores the insight's timestamp and switches to the Event Snapshot tab, which immediately fetches `api.php?mode=snapshot&ts=<ISO timestamp>&before=<min>&after=<min>`. The API queries all six raw metric tables (`system_metrics_raw`, `api_metrics_raw`, `db_metrics_raw`, `proxy_metrics_raw`, `alerts`, `insights`) for rows falling in the window `[ts − before, ts + after]` and returns them as a single JSON object. The dashboard renders the results without a full-page reload.

**Insight context card** — displayed at the top of the snapshot panel: insight type, severity pill, domain/metric, detection timestamp, current value, baseline mean, z-score, drift %, and the full human-readable message. Every element in this card carries a tooltip:
- **Severity pill** — explains the severity level (critical/warn/info) and what it implies.
- **Type pill** — describes the detection method or insight category.
- **Detected / Current / Baseline / Z-Score / Drift stat boxes** — each box tooltip explains what the value represents, how to interpret it, and the relevant thresholds. For chart-point snapshots, Selected Time and Value are explained instead.

**Time window controls** — preset buttons for ±2, 5, 10, 15, and 30 minutes flank the insight timestamp. Custom before/after values can be typed directly into the minute inputs. A Refresh button re-fetches with the current window. The window is clamped server-side to 1–60 minutes per side to prevent runaway queries. Every control carries a tooltip: preset buttons show the total window duration, the numeric inputs explain the valid range, and the Apply button describes the action.

**Summary row** — peak and minimum stat cards for CPU, memory, load average, RPS, error rate, latency, DB connections, DB cache hit ratio, deadlocks, slow queries, top process CPU, and log error/warning counts are computed from the fetched window rows and displayed above the chart grid. Each card carries an intelligent tooltip with:
- What the metric measures and how it is calculated.
- Warning (⚠) and critical (⛔) thresholds with exact values.
- The healthy range for reference.
- Actionable guidance (e.g., "run EXPLAIN ANALYZE", "increase shared_buffers", "check the Top Process table").

**Section headings** — the four section headings (Peak/Min Values, System State Charts, Top Processes, Co-occurring Events) each carry a tooltip summarising the section's purpose and how to read it.

**Charts** — twelve canvas-based time-series charts drawn for the fetched window: CPU system/user %, memory used vs available (GB), 1m/5m load averages, network RX/TX (MB/s), API RPS, API error rate %, API p50/p95/p99 latency ms, DB active/idle/waiting connections, DB cache hit ratio %, DB wait events (Lock/IO/LWLock/Client), DB write activity (inserted/updated/deleted tuples), and proxy p95/p99 latency ms. Each chart box carries a tooltip on hover explaining what the series lines mean, what to look for (e.g., diverging p50/p99, used↑/available↓ divergence), and how to correlate it with other charts. Each chart renders a vertical dashed red annotation line at the insight's detection timestamp so the anomaly moment is immediately visible relative to surrounding metric behaviour.

**Top Processes table** — processes observed during the window ranked by peak CPU. Column headers carry tooltips: Process (aggregation explained), Peak CPU % (threshold indicators), Peak Memory (RSS explained). Rows sorted by peak CPU descending.

**Co-occurring events table** — alerts, co-occurring insights, and log entries from the same time window are merged into a single chronological table showing timestamp, type, severity, source, and message. Column headers carry tooltips explaining each field. Individual rows carry tooltips showing the full message text, event kind, severity, source, and absolute timestamp — useful when the Message column is truncated. This surfaces correlated signals (e.g., a DB cache drop insight alongside an API latency alert) without switching tabs.

**Click handling** — row click detection uses event delegation on the `insights-tbody` element rather than per-row listeners. This is necessary because `MonTables.enhance()` rebuilds table rows via `cloneNode(true)` on every pagination or sort event, which strips any listeners attached directly to `<tr>` elements. The delegated listener reads a `data-snap-ins-idx` attribute (preserved through cloning) to look up the insight object from a module-level reference array.

**Tooltip system** — all tooltips in the Event Snapshot tab use the same JS tooltip engine (`data-tip` attribute + mouseover delegation, rendered in a fixed-position element appended to `<body>`) as every other dashboard tab. This makes them immune to `overflow: hidden` clipping on chart containers and cards. All elements with `data-tip` automatically receive `cursor: help` via the global `[data-tip] { cursor: help }` CSS rule.

## License

MIT
