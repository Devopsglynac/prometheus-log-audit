# Prometheus Alert Tuning Audit — DEV-289

## Objective
Analyze Prometheus alert behavior over an extended local audit window to identify
alerts that fire frequently but require no action ("noise"), tune them to reduce
alert fatigue, and validate that genuinely actionable alerts still function
correctly after tuning.

## Methodology
- Reproduced the reference PoC environment locally (Docker Compose: Prometheus,
  Alertmanager, Grafana, Loki, sample webapp, Postgres, Redis).
- Ran `generate_load.sh` and `generate_spikes.sh` continuously for approximately
  24 hours to build a realistic, sustained dataset rather than a short synthetic
  burst.
- Queried Prometheus directly via PromQL (`count_over_time(ALERTS{...}[24h])`)
  to get accurate, per-alert firing and pending counts, since the reference
  repo's own `export_alerts.sh` script was found to have a bug (see Findings).

## Key Findings

### 1. Consistently firing (confirmed noise)
| Alert | Firing count (24h) | Root cause |
|---|---|---|
| `HighResponseTime` | 2028 | Test-only `/slow` endpoint included in global p95 latency calculation, permanently skewing the metric |
| `RedisMemoryHigh` | 2021 | Redis instance is configured as an LRU cache — being "full" is expected by design, not a fault |

### 2. Pending churn (threshold crossed repeatedly, never sustains to firing)
| Alert | Pending count (24h) | Root cause |
|---|---|---|
| `LowCacheHitRate` | 852 | Triggers during normal, brief cache-refresh events |
| `HighDatabaseConnections` | 586 | Normal connection pool behavior; threshold too low for actual usage |
| `HighCPUUsage` | 11 | Threshold (`>50%`) too low; spikes cross it briefly without indicating a real problem |

This "pending churn" pattern is distinct from steady firing — the alert's `for:`
duration is currently preventing pages, but the repeated threshold-crossing still
represents miscalibration worth correcting, and adds noise to the Prometheus/
Alertmanager UI even without notifying anyone.

### 3. Critical bug found: RedisDown was non-functional
The reference repo's own `alerts.yml` marks `RedisDown` as "ACTIONABLE: Real
issue" — the one alert assumed to reliably detect real outages. Investigation
during this audit found it was **silently broken**:

- Original expression: `up{job="redis"} == 0`
- This metric reflects whether Prometheus can reach the **redis-exporter**
  container, not whether Redis itself is running.
- Verified via failure injection: stopping the `redis` container directly
  left `up{job="redis"}` at `1` (exporter still reachable) while the
  exporter's own `redis_up` metric correctly dropped to `0`.
- Further testing found a second edge case: prolonged Redis unavailability
  eventually caused the exporter itself to stop responding to scrapes,
  making `up{job="redis"}` drop to `0` while `redis_up` became entirely
  absent (not `0` — no data). A simple `redis_up == 0` check alone would
  have missed this failure mode too.
- **Fix**: `up{job="redis"} == 0 or redis_up == 0` — catches both "exporter
  unreachable" and "exporter reachable but Redis itself down."
- Validated end-to-end: alert fires within the `for: 1m` window when Redis
  is stopped, and correctly resolves once Redis recovers.

### 4. Bug found in reference repo's export tooling
`scripts/export_alerts.sh`'s "notifications by alertname" query
(`sum by(alertname)(increase(alertmanager_alerts_received_total[...]))`)
cannot work as written — `alertmanager_alerts_received_total` has no
`alertname` label (only `instance`, `job`, `status`, `version`), so the
query silently returns 0 for every alert regardless of actual volume. Firing
counts in this report were instead derived directly from `ALERTS{alertstate=...}`,
which does carry the `alertname` label correctly.

### 5. Not reproduced locally
`HighMemoryUsage`, `DiskSpaceWarning`, `ApplicationMemoryHigh`,
`PostgresConnectionErrors`, `PostgresSlowQueries` did not fire during this
audit's load window. Tuning recommendations for these follow the reference
repo's own documented rationale rather than first-hand local evidence.

## Tuning Actions Applied
See `prometheus/alerts_tuned.yml` for full rule definitions. Summary:

| Alert | Change |
|---|---|
| `HighResponseTime` | Exclude `/slow` route from p95 calculation; `for` raised 2m→5m |
| `RedisMemoryHigh` | Replaced with `RedisEvictionRateHigh` (monitors eviction rate, the actual signal for capacity pressure on an LRU cache) |
| `LowCacheHitRate` | `for` raised 5m→10m to absorb routine refresh blips |
| `HighDatabaseConnections` | Threshold raised 15→30 connections to reflect real pool sizing |
| `HighCPUUsage` | Threshold raised 50%→85%, `for` raised 3m→5m |
| `RedisDown` | **Critical fix** — corrected to `up{job="redis"} == 0 or redis_up == 0` |
| `HighMemoryUsage`, `DiskSpaceWarning`, `ApplicationMemoryHigh`, `PostgresConnectionErrors`, `PostgresSlowQueries` | Tuned per reference repo baseline (not independently reproduced) |

## Results & Validation
- Applied tuned rules via live Prometheus config reload (`/-/reload`), preserving
  the full 24h data history for before/after comparison.
- Confirmed via `ALERTS{alertstate="firing"}` query: both `HighResponseTime` and
  `RedisMemoryHigh` — 4049 combined firing events over 24h — stopped firing
  entirely under the tuned configuration with unchanged underlying traffic.
- Confirmed `RedisDown` fires correctly on injected failure (container stopped)
  and resolves correctly on recovery — verifying the critical fix works as
  intended and the tuning didn't compromise real incident detection.

## Attachments
- `prometheus/alerts.yml` — original rules
- `prometheus/alerts_tuned.yml` — tuned rules (validated via `promtool check rules`)
- Raw data: `alert_analysis_*.json`, `alert_noise_top10_*.csv`
