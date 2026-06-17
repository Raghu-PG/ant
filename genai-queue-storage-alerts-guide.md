# GenAI Queue Storage Consumption — Alerts Guide (GENAI-2580)

Companion to `genai-queue-storage-alerts.thanos.rules.yaml` (static thresholds) and
`genai-queue-storage-anomaly.thanos.rules.yaml` (z-score baseline, opt-in).

This doc explains **what each alert does, how to calibrate it, and the PromQL you
can run in Grafana Explore / Thanos to verify the thresholds against real data
before committing.** Treat the "exploration query" blocks as the queries to paste
into Grafana to get the insight behind each number.

---

## 1. Design decisions (agreed)

| Decision | Choice | Consequence |
|---|---|---|
| Rule engine | **Thanos Ruler** (evaluated against Thanos Query) | Global, cross-cluster view. `partial_response_strategy: abort` set on every group so partial query failures abort eval instead of producing false data. |
| Environment model | **All envs, one ruleset** | No `cluster/namespace/release` filters in expressions. Every aggregation is `... by (cluster, namespace, release, queue_name)` so prod/dev/dr series never merge. Environment travels on the alert as labels. |
| Scope | Table alerts **+ hardened extras** | The 9 agreed alerts, plus duplication-risk, DLQ store/delete failures, deletion failures, and absent()-backstops for the "exporter/worker is gone" blind spot. |
| Routing metadata | **Placeholders** (`TODO(conventions)`) | Replace `service`/`component` labels, add your `team`/routing labels, and fill `runbook_url` / `dashboard_url` once you paste your conventions. |

---

## 2. Alert inventory

### Static thresholds — `genai-queue-storage-alerts.thanos.rules.yaml`

| Alert | Metric | Threshold | for | Severity | From table? |
|---|---|---|---|---|---|
| QueueOldestMessageAgeWarning | `queue_oldest_message_age_seconds` | > 600s (10m) | 5m | warning | ✅ |
| QueueOldestMessageAgeCritical | `queue_oldest_message_age_seconds` | > 1800s (30m) | 5m | critical | ✅ |
| QueueDepthWarning | `queue_approximate_message_count` | > 1000 | 10m | warning | ✅ |
| QueueDepthCritical | `queue_approximate_message_count` | > 5000 | 5m | critical | ✅ |
| QueueConsumerTaskStalled | `increase(queue_consumer_runs_total[10m])` | == 0 | 5m | critical | ✅ |
| QueueProcessingFailureRateHigh | dequeued − uploaded(success) ÷ dequeued | > 10% (+vol guard) | 5m | warning | ✅ |
| QueueFallbackToCosmosActive | `queue_messages_sent_total{status="fallback_cosmosdb"}` | rate > 0 | 2m | warning | ✅ |
| DLQDepthWarning | `queue_dead_letter_approximate_count` | > 0 | 5m | warning | ✅ |
| DLQDepthCritical | `queue_dead_letter_approximate_count` | > 100 | 5m | critical | ✅ |
| QueueProcessingDurationNearVisibilityTimeout | p99 of `..._duration_seconds_bucket` | > 240s (80%) | 5m | warning | ➕ extra |
| QueueProcessingDurationExceedsVisibilityTimeout | p99 of `..._duration_seconds_bucket` | > 285s (95%) | 5m | critical | ➕ extra |
| QueueDeletionFailures | `queue_messages_deleted_total{status="failure"}` | rate > 0 | 5m | warning | ➕ extra |
| DLQStoreFailures | `queue_dead_letter_store_failures_total` | rate > 0 | 5m | critical | ➕ extra |
| DLQDeleteFailures | `queue_dead_letter_delete_failures_total` | rate > 0 | 5m | warning | ➕ extra |
| QueueMetricsAbsent | `absent_over_time(queue_approximate_message_count[10m])` | absent | 5m | critical | ➕ backstop |
| QueueConsumerMetricsAbsent | `absent_over_time(queue_consumer_runs_total[15m])` | absent | 5m | critical | ➕ backstop |

### Anomaly baseline — `genai-queue-storage-anomaly.thanos.rules.yaml`

| Alert | Signal | Condition | for | Severity |
|---|---|---|---|---|
| QueueThroughputAnomalousDrop | z-score of `queue:processing_throughput:rate5m` vs trailing 6h | z < −3 (+ baseline guards) | 10m | warning |

---

## 3. Why the hardened extras matter (the blind spots in pure thresholds)

1. **Dead exporter = silent thresholds.** `queue_oldest_message_age_seconds > 1800`
   only fires while the series exists. If the consumer/exporter pod dies, the
   series goes stale and disappears — the threshold rule evaluates to *nothing*
   and never fires. The exact "celery beat died" outage you described can hide
   here. `QueueMetricsAbsent` / `QueueConsumerMetricsAbsent` cover it.
   *(These are intentionally coarse — `absent()` can't iterate `queue_name`, so
   they key on cluster+namespace.)*

2. **`QueueConsumerTaskStalled` vs `QueueConsumerMetricsAbsent`.** Stalled =
   counter still exported but flat (beat scheduler dead, process alive). Absent =
   the metric itself is gone (process dead). You want both; they catch different
   failures.

3. **Duplication risk.** p99 processing time approaching the 300s
   `visibility_timeout` means a message can reappear mid-processing and be handled
   twice. Static depth/age alerts won't catch this.

4. **DLQ store failures = potential data loss.** A message that should be
   dead-lettered but can't be stored may be lost entirely — that's why it's
   `critical`, stricter than DLQ-depth-> 0 (`warning`).

---

## 4. Exploration queries — calibrate before you commit

Run these in **Grafana → Explore** (Thanos/Prometheus datasource) over a 7–30 day
range. They tell you whether a threshold will be quiet in normal operation and
loud during incidents. Adjust the rule numbers if your data disagrees.

**Queue depth — does "near 0 normally" hold? What is p95/p99?**
```promql
quantile_over_time(0.95, sum by (cluster, namespace, release, queue_name)(queue_approximate_message_count)[30d:5m])
quantile_over_time(0.99, sum by (cluster, namespace, release, queue_name)(queue_approximate_message_count)[30d:5m])
max_over_time(sum by (cluster, namespace, release, queue_name)(queue_approximate_message_count)[30d:5m])
```
> If p99 is well under 1000 and only incidents push it higher, the 1000/5000
> thresholds are safe. If some env routinely sits at 2–3k, raise its threshold or
> split per-env (see §6).

**Oldest message age — normal vs tail.**
```promql
quantile_over_time(0.95, max by (cluster, namespace, release, queue_name)(queue_oldest_message_age_seconds)[30d:1m])
max_over_time(max by (cluster, namespace, release, queue_name)(queue_oldest_message_age_seconds)[30d:1m])
```
> Confirms the "~4 min = 2 beat cycles" assumption. If p95 age is ~120s, the
> 600s warning has comfortable headroom.

**Consumer run cadence — is a 10m no-run window actually abnormal?**
```promql
# runs per 10m, per queue — the floor should be > 0 in healthy operation
sum by (cluster, namespace, release, queue_name)(increase(queue_consumer_runs_total[10m]))
# longest observed gap proxy: min runs-per-10m over the last 7d
min_over_time(sum by (cluster, namespace, release, queue_name)(increase(queue_consumer_runs_total[10m]))[7d:1m])
```
> If healthy minimum is comfortably ≥ 1 per 10m, the `== 0 for 5m` stall rule
> won't false-fire. If beat runs every ~2 min, 10m is ~5 expected runs of slack.

**Processing failure rate — what is the normal baseline, and is the volume guard right?**
```promql
# failure ratio (mirror of the alert)
(
  sum by (cluster, namespace, release, queue_name)(increase(queue_messages_dequeued_total[5m]))
  - sum by (cluster, namespace, release, queue_name)(increase(queue_messages_uploaded_total{status="success"}[5m]))
)
/ clamp_min(sum by (cluster, namespace, release, queue_name)(increase(queue_messages_dequeued_total[5m])), 1)

# typical dequeue volume per 5m — informs the "> 20" guard
quantile_over_time(0.50, sum by (cluster, namespace, release, queue_name)(increase(queue_messages_dequeued_total[5m]))[7d:5m])
```
> If most 5m windows process < 20 messages, lower the guard or it will rarely
> evaluate. If there's a `queue_messages_uploaded_total{status="failure"}` series,
> see §6 for a cleaner failure-rate expression.

**Processing duration vs visibility_timeout (300s).**
```promql
histogram_quantile(0.99, sum by (cluster, namespace, release, queue_name, le)(rate(queue_message_processing_duration_seconds_bucket[5m])))
histogram_quantile(0.999, sum by (cluster, namespace, release, queue_name, le)(rate(queue_message_processing_duration_seconds_bucket[5m])))
```
> If p99 normally sits at 5–30s, the 240/285 thresholds are pure safety nets and
> won't be noisy. Confirm `visibility_timeout` is really 300s for every env.

**DLQ inflow — is "> 100" a meaningful mass-failure line for your traffic?**
```promql
sum by (cluster, namespace, release, queue_name, reason)(increase(queue_dead_letter_messages_total[1h]))
max_over_time(sum by (cluster, namespace, release, queue_name)(queue_dead_letter_approximate_count)[30d:5m])
```

**Fallback frequency — is `rate > 0 for 2m` going to be chatty?**
```promql
sum by (cluster, namespace, release, queue_name)(increase(queue_messages_sent_total{status="fallback_cosmosdb"}[5m]))
```

---

## 5. Z-score baseline (the dynamic / "estimated baseline" piece)

**Idea.** Static thresholds catch absolute breakage. The z-score catches *relative*
degradation: a queue processing at 30% of its own normal rate while still under
the depth threshold. Formula:

```
z = (current_value − mean_over_baseline) / stddev_over_baseline
```

`|z| > 3` ≈ outside the 99.7% band of recent behaviour. The shipped rule is
**one-sided** (`z < −3`) on processing throughput — we only care about slowdowns.

**Verify the baseline behaves before trusting alerts.** Plot the z-score directly
in Explore:
```promql
(
  queue:processing_throughput:rate5m
  - avg_over_time(queue:processing_throughput:rate5m[6h])
)
/ stddev_over_time(queue:processing_throughput:rate5m[6h])
```
> Healthy operation should hover in roughly [−2, +2]. If it's constantly past −3,
> the window is too short or the queue is too bursty — see tuning below.

**Tuning levers.**
- **Baseline window** (`[6h]`): longer = smoother, slower to react. 1h is a common
  sweet spot for fast signals; 6h–24h reduces false positives on bursty queues.
- **Seasonality:** if traffic has a daily/weekly shape, compare against the same
  window in the past instead of a trailing window, e.g.
  `avg_over_time(metric[1h] offset 1w)` as the baseline mean.
- **Smoothed sigma (anti-false-positive):** record a short-window stddev then
  average it, so one spike doesn't inflate sigma and mask real drops:
  ```promql
  # record: queue:throughput:stddev5m  -> stddev_over_time(queue:processing_throughput:rate5m[5m])
  # baseline sigma: avg_over_time(queue:throughput:stddev5m[24h])
  ```
- **Threshold & `for`:** start at `z < −3`, `for: 10m`. Only tighten to −2.5 after
  a week of clean data.

**Warmup caveat (Thanos):** the recording rule must accumulate a full baseline
window in TSDB before `avg_over_time/stddev_over_time` are meaningful. Expect no /
noisy firing for the first ~6h after deploy.

---

## 6. Open items / things to confirm before merge

1. **Paste your label + URL conventions.** Everywhere marked `TODO(conventions)`:
   the routing labels you use for Alertmanager (e.g. `team`, `squad`, `owner`),
   your runbook URL pattern, and your Grafana base URL. I'll swap them in and the
   `service`/`component` labels to match your scheme.

2. **Failure-rate denominator.** The rule uses the dashboard's definition
   (`dequeued − uploaded_success`). If the consumer also emits
   `queue_messages_uploaded_total{status="failure"}` (or a `..._failed_total`),
   a cleaner, less in-flight-sensitive expression is:
   ```promql
   sum by (...)(rate(queue_messages_uploaded_total{status="failure"}[5m]))
   / sum by (...)(rate(queue_messages_uploaded_total[5m])) > 0.10
   ```
   Tell me which metrics/labels actually exist and I'll switch it.

3. **Per-env thresholds?** You chose one global ruleset, so Queue Depth critical
   is a single `> 5000`. If you'd rather honour each env's `max_messages_per_run`
   (3200 dev-ext, 6400 dev, …), I can either (a) split the rule with
   `release=~"..."` matchers, or (b) template it via Helm values. Say the word.

4. **Packaging.** These are plain Prometheus rule groups for Thanos Ruler. If your
   repo wraps rules in a `PrometheusRule` CRD or a ConfigMap, I'll wrap them —
   `partial_response_strategy` is preserved at group level in both. Send one
   existing rule file and I'll match its exact shape.

5. **Validation done:** both files were parsed and checked with `promtool check
   rules` (Thanos uses the same rule schema). See the delivery note.

---

## 7. FINAL BUILD (v2) — what changed after the Grafana introspection

The canonical file is now **`genai-queue-storage.ext-dev.rules.yaml`** (all groups
folded into one, repo convention). The earlier `*.thanos.rules.yaml` pair is
superseded. Changes driven by real Thanos data:

| Change | Why (evidence) |
|---|---|
| Gauges switched `sum` → **`max`** by (cluster,namespace,release,queue_name) | Metric is emitted **per pod** — two `celery-worker` pods share one `queue_name`. `sum` would double depth/age/DLQ-depth. Counters stay `sum` (additive across pods). |
| **4 alerts commented out** | `queue_messages_sent_total`, `queue_messages_deleted_total`, `queue_dead_letter_store_failures_total`, `queue_dead_letter_delete_failures_total` do **not exist** in Thanos (only 23 queue_* metrics; these aren't among them). Includes the table's **Fallback Active** — enable when the producer emits the metric. |
| Failure-rate now uses **`queue_messages_uploaded_total{status="failure"}`** | That series was confirmed to exist; cleaner and less in-flight-sensitive than `dequeued − uploaded(success)`. Denominator = dequeued rate (no dependence on a `success` series, which was not observed). |
| Throughput baseline uses **`rate(queue_message_processing_duration_seconds_count)`** | Status-label-independent processing rate — works even though no `uploaded{status="success"}` series appeared. |
| **Removed `partial_response_strategy`** | Repo deploys rules as ConfigMaps to a Thanos-**sidecar Prometheus**, not a standalone Thanos Ruler. CI `promtool` (v3.x) rejects the field. |
| **Title-case severity** + repo routing labels | `Warning`/`Critical`/`Low`; added `alert_namespace`, `alert_owner`, `email_subject`, `alert_name` per `validate_alert_labels.py`. Dropped non-routing `service`/`component`. |

### Deployment facts (confirmed)

- Metric currently exists **only in dev**: cluster `aks-aife-mt-dev-ca05`,
  namespace `tenant-genai-platform-ext-dev`, release `dev-pr2280` (**ephemeral,
  PR-based**). No prod data yet.
- Prometheus replica for that env: **`prometheus-sre-aks-aife-mt-dev-ca05-0`**
  (goes in the kustomization annotation).
- Ephemeral releases have two consequences, both documented inline in the rules:
  the **staleness/absent backstops** will fire on normal PR-env teardown, and the
  **z-score anomaly** rarely accumulates its 6h baseline. Both are most useful in
  prod on stable releases.

### Set-these-before-merge checklist

1. Global find/replace `alert_owner: genai-platform-team@pg.com` → your real
   routing distribution list (CI requires a valid email).
2. Global find/replace `alert_namespace: genai-queue-consumption` → a Spyglass
   **registered** alert_namespace (coordinate with SRE / EM onboarding).
3. Fill the `TODO(conventions)` `runbook_url` / `dashboard_url` bases.
4. Reconcile `kustomization.ext-dev.yaml` against a real one in the repo
   (selector labels + exact `replica` annotation key) and rename to
   `kustomization.yaml` in the target folder.
5. Run `.github/scripts/validate_alert_labels.py` and `promtool check rules`
   locally to confirm the label schema and PromQL pass CI.
6. Decide routing depth: **email-only** (current labels are enough) vs **critical
   → ServiceNow** (needs EM onboarding + `snow_*` labels from
   `servicenow-incident-labels.md`).

### Still-open (non-blocking) calibration inputs

- `visibility_timeout` per env (assumed 300s → drives 240s/285s duplication
  thresholds).
- prod `max_messages_per_run` and whether to make Queue-Depth-critical per-env.
- celery beat cadence (assumed ~2 min → underpins the 10m stall window).
- Is `tenant-genai-platform-dr` passive? If so, exclude it from the
  heartbeat/age alerts when this promotes beyond dev.

### Bonus signals available but unused (optional)

`queue_message_age_milliseconds` (histogram → age **distribution**, not just
oldest), `queue_reader_count` (could be a cleaner "workers alive" check than the
heartbeat), `queue_latency`, `queue_slice_count`, `queue_actions` (may carry the
enqueue/fallback signal — worth checking with
`count by (__name__, action, status, reason)(queue_actions)`).
