# Observability Dashboard Standard — L1 / L2 / L3
### A platform-agnostic, SRE-grade standard for tiered dashboards

**What this is.** A reusable, vendor-neutral standard for building three tiers of observability dashboards for *any* system — an HTTP API, a streaming/batch pipeline, a database, an ML platform, a mobile backend, infrastructure. It tells you the **fixed stacks (sections) every tier must contain and what each means**, grounded in the established observability methods. The **panels inside a stack flex** to whatever the system exposes; **the stacks and their intent stay constant** so every dashboard — across teams, platforms, and vendors — reads the same way under pressure.

**Design goal.** A first responder who has never seen your service should be able to open L1 and know *whether* something is wrong in ~10 seconds, drop to L2 to know *what and where* in ~1 minute, and use L3 to understand *why*. Consistency across dashboards is itself a reliability feature: during an incident, muscle memory beats novelty.

**How to use it.** Pick the tier → run the intake checklist (§11) → clone the matching skeleton (`skeleton_L1_universal.json` / `L2` / `L3`) → fill each fixed stack with the panels your telemetry supports → check the Definition of Done (§12) and avoid the anti-patterns (§13).

---

## 0. The three questions (the whole model in one line)
**L1 → "Is the system healthy?"  ·  L2 → "What and where is it broken?"  ·  L3 → "Why is it broken?"**
Everything below is in service of answering those three questions fast, in that order, with a clean drill-down path between them.

## 1. Foundations (the methods this standard is built on)
This standard composes the industry-standard observability methods rather than inventing new ones:

- **The Four Golden Signals** (Google SRE): **Latency, Traffic, Errors, Saturation.** The minimum a user-facing system must show.
- **RED method** (request-driven services): **Rate, Errors, Duration** — apply per service and per route/endpoint.
- **USE method** (resources): **Utilization, Saturation, Errors** — apply per resource (CPU, memory, disk, network, thread/conn pool, queue).
- **SLIs / SLOs / Error budgets:** measure what users experience, set a target, and spend the remaining budget deliberately. Drives L1.
- **Multi-window, multi-burn-rate alerting** (Google SRE Workbook): alert on *how fast* you are burning budget, not on raw thresholds. Drives the SLO stack.
- **The telemetry pillars:** **metrics** (cheap, aggregate, alertable) → **logs** (detailed, high-cardinality) → **traces/exemplars** (causal, cross-service). Higher tiers lean more on logs/traces.

Rule of thumb: **RED for services, USE for resources, Golden Signals for the user-facing topline, SLOs for the headline.** Every tier should make clear which method each stack is satisfying.

## 2. The tier model

| | **L1 — Overview** | **L2 — Triage** | **L3 — Deep Diagnostics** |
|---|---|---|---|
| Question | Is the system healthy? | What and where is broken? | Why is it broken? |
| Audience | NOC / on-call glance / leadership | On-call triaging an alert | Engineer in the incident |
| Scope | Whole service/product, SLO level | Service → component/route/resource | One component/endpoint/instance |
| Primary methods | SLO + Golden Signals (aggregate) | RED (per service) + USE (per resource) + dependencies | Golden Signals (scoped) + error taxonomy + latency distribution + traces |
| Telemetry | Metrics | Metrics (+ some logs) | Logs + traces/exemplars + metrics |
| Default range | now-6h (and an SLO-window view) | now-6h / now-24h | now-1h to now-24h |
| "Done in" | ~10 seconds | ~1 minute | as long as it takes |
| Panels on screen | few, large, no scroll | moderate | many, detailed |

Each tier **drills into the next**: L1 tiles link to the matching L2; L2 tiles link to the matching L3 (and to a resource/instance view); L3 links back up and across to siblings. Links carry variables + the active time range.

## 3. Universal design principles (best practices for any dashboard)
1. **One question per panel.** If you can't state the question a panel answers in a sentence, remove it.
2. **Priority by position.** Most important, most aggregated info **top-left**; detail flows down and right. L1 is the most aggregated; L3 the most detailed.
3. **Percentiles, never averages, for latency.** Show p50/p90/p95/p99 (and max for outliers). Averages hide tail pain. Use a **heatmap** for the full distribution.
4. **Rates, not raw counts, for errors/traffic** — and show **error rate as a proportion** (errors/total), not just a count, so it's comparable across load.
5. **Saturation needs a limit line.** A utilization/queue panel is only meaningful against its capacity/limit threshold.
6. **Semantic, colorblind-safe color.** Green=healthy, amber=warning, red=breach — consistently, everywhere. Don't use color decoratively.
7. **Consistent units & thresholds.** Set units (s/ms, %, req/s, bytes) and threshold steps on every numeric panel.
8. **Annotate changes.** Overlay **deploys/config changes/feature flags** as annotations — "what changed" is the first triage question.
9. **Templating over duplication.** Use variables (`$environment`, `$service`, `$instance`, `$region`, `$datasource`) so one dashboard serves many scopes; default to sensible values.
10. **Drill-down, not dead-ends.** Every aggregate tile links to the next level with context preserved.
11. **No dual Y-axes; tidy legends.** Avoid misleading dual axes; keep legends short (use `{{label}}` templating); cap series or use "top N".
12. **Dashboards as code.** Version the JSON, assign an **owner**, review changes, and keep uids/slugs stable.
13. **Tie panels to alerts/SLOs.** The signals you alert on should be visible here; the SLO stack should mirror your alerting burn-rate policy.
14. **Readable under stress.** Fewer, larger panels beat a wall of sparklines. Optimize L1 for a 3am glance on a phone.

## 4. The stack concept
A **stack** is a named row representing one observability intent (mapped to a method). Stacks are **fixed per tier** (name + meaning); the **panels** inside are whatever your telemetry supports. Keep a stack even if you can fill it with one panel; if it genuinely can't be filled, leave a short text panel explaining why rather than deleting it (so reviewers see the gap). This is what makes dashboards interchangeable across platforms.

## 5. L1 stacks — Overview ("Is the system healthy?")  · tags `[L1]`

| Stack (row) | Intent | Method | Mandatory | Example panels (flex) | Drill-down |
|---|---|---|---|---|---|
| **1. SLO & Error Budget** | Are we meeting our promise; how much budget is left; how fast are we burning it? | SLO | Yes | Availability/latency SLO compliance (stat/gauge), error-budget remaining (bar/stat), multi-window burn-rate (timeseries) | → L2 |
| **2. Golden-Signal Topline** | Aggregate Traffic, Errors, Latency, Saturation right now | Golden Signals | Yes | Requests/sec, overall error rate %, p95/p99 latency, peak saturation — all aggregate stats/timeseries | → L2 |
| **3. Critical Dependencies & Capacity** | Are our key upstreams/downstreams up; do we have headroom? | USE (capacity) | Yes | Dependency up/down matrix, headroom gauges (CPU/mem/queue vs limit), regional/zone status | → L2 / L3 |
| **4. Active Incidents & Changes** *(recommended)* | What's already firing and what just changed | — | Optional | Firing-alerts list, recent deploy annotations/version, change log | → alert/runbook |

L1 rule: **summary only, single screen, no per-request detail.** If a number is red here, one click goes to the L2 that explains it.

## 6. L2 stacks — Triage ("What and where is broken?")  · tags `[L2]`

| Stack (row) | Intent | Method | Mandatory | Example panels (flex) | Drill-down |
|---|---|---|---|---|---|
| **1. Request Health (per service/route)** | Which service/endpoint is failing or slow | RED | Yes | Rate, error rate, and p95/p99 duration **per service and per route**; top failing endpoints | → L3 service |
| **2. Resource Health (per resource)** | Which resource is constrained | USE | Yes | CPU/mem/disk/network utilization + saturation (with limits), thread/conn-pool & queue depth, restarts/OOM | → L3 resource/instance |
| **3. Dependency & Integration Health** | Are downstreams/externals the cause | RED (per dependency) | Yes | Per-dependency rate/errors/latency, timeouts, retries, circuit-breaker state, DB/queue health | → dependency L3 |
| **4. Change & Deploy Correlation** | Did a change cause this | — | Recommended | Deploy/version timeline, config/flag changes overlaid on error/latency | → CI/CD / change record |

L2 rule: **break the system down by the dimension that routes the investigation** (service, route, resource, dependency, version). Anything that has an L3 must be reachable from here.

## 7. L3 stacks — Deep Diagnostics ("Why is it broken?")  · tags `[L3]`

| Stack (row) | Intent | Method | Mandatory | Example panels (flex) |
|---|---|---|---|---|
| **1. Overview / Golden Signals (scoped)** | One-glance health of *this* component | Golden Signals | Yes | Traffic, error rate, p95/p99 latency, saturation — scoped stats |
| **2. Traffic & Demand** | How much, from where, of what kind | RED (Rate) | Yes | Volume over time; by endpoint/method/client/tenant/region; request size |
| **3. Errors (taxonomy)** | Exactly what is failing and why | RED (Errors) | Yes | Error rate over time; by type/code/cause; true failures vs ancillary (kept separate); top error messages |
| **4. Latency (distribution)** | Where the time goes and the tail shape | RED (Duration) | Yes if available | Percentiles over time **+ latency heatmap**; latency by endpoint/dependency; slowest operations |
| **5. Saturation & Resources** | Is it resource-bound | USE | Yes if applicable | Queue depth, pool exhaustion, CPU/mem/GC, I/O wait, backlog/lag — each vs its limit |
| **6. Dependencies (per-dependency RED)** | Which downstream is dragging us | RED | Yes if applicable | Per-dependency rate/errors/latency, retry/timeout/circuit-breaker, connection pool |
| **7. Breakdown (by dimension)** | Which slice is affected | All | Yes | By endpoint/version/region/tenant/instance/model — success-rate & latency tables |
| **8. Troubleshooting (logs/traces)** | Get to the actual failing unit of work | Pillars | Yes | Exemplars/trace links, recent failures with correlation IDs + status, filtered log explorer |

L3 rule: **separate true component failures from ancillary noise**, prefer **distributions over averages**, expose **correlation IDs / trace links**, and end with a **troubleshooting** stack that reaches individual events.

## 8. Method → tier coverage matrix
| Method / signal | L1 | L2 | L3 |
|---|---|---|---|
| Traffic (Rate) | Topline | Request Health | Traffic & Demand |
| Errors | SLO + Topline | Request + Dependency Health | Errors (taxonomy) |
| Latency (Duration) | Topline | Request Health | Latency (distribution) |
| Saturation (USE) | Capacity | Resource Health | Saturation & Resources |
| SLO / Error budget | SLO & Error Budget | (derived) | (scoped overview) |
| Dependencies | Critical deps | Dependency Health | Per-dependency RED |
| Change/Deploy | Incidents & Changes | Change Correlation | (annotations) |
A tier is **incomplete** if any applicable row is empty.

## 9. SLIs, SLOs & burn-rate alerting (drives L1)
**Pick SLIs that reflect user experience.** Common SLI types:
- **Availability** = good events / valid events (e.g. non-5xx / total, or successful jobs / scheduled jobs).
- **Latency** = proportion of requests faster than a threshold (e.g. ≥99% < 300ms).
- **Quality / correctness** = proportion of responses that are correct/complete.
- **Freshness / lag** (pipelines/streaming) = proportion of data processed within the freshness target.
- **Throughput / coverage / durability** as the domain requires.

**Express the SLI as a ratio of good to valid events**, set the SLO target (e.g. 99.9% over 30 days), and the **error budget** is `1 − SLO`. The L1 "SLO & Error Budget" stack shows: current SLI, target, budget remaining, and burn rate.

**Multi-window, multi-burn-rate alerting** (canonical, 30-day window) — mirror this in the burn-rate panel and your alert rules:

| Severity | Budget consumed if sustained | Long window | Short window | Burn rate |
|---|---|---|---|---|
| Page (fast burn) | 2% in 1h | 1h | 5m | 14.4 |
| Page (medium) | 5% in 6h | 6h | 30m | 6 |
| Ticket (slow) | 10% in 3d | 3d | 6h | 1 |

The short window confirms the condition is still active (prevents alerting on an event that already ended); the long window controls precision. Show all three burn rates as series so responders see how fast budget is being spent.

## 10. Cross-cutting conventions
**Naming.** Dashboard: `<Service/Scope> <Tier>` (e.g. "Checkout API L2"). Rows: the fixed stack names above, verbatim. Panels: `<Subject> <Metric> [by <Dimension>] [(qualifier)]` — e.g. "Checkout Error Rate by Route", "p99 Latency (heatmap)". No vague titles; no duplicates; **every panel has a description** stating its question.

**Tags.** `[L1]` / `[L2]` / `[L3]` are mandatory (they power the cross-tier link dropdowns). Add domain tags as useful (`team:`, `service:`, `env:`).

**Variables (standard set, platform-agnostic).**
- `$datasource` (type=datasource) — never hardcode a datasource uid.
- `$environment`, `$region`, `$cluster`, `$service`, `$instance` — the scoping chain (use what applies).
- Dimension vars as supported: `$route`, `$tenant`, `$version`, etc. Use `includeAll` with `allValue="*"` so "All" works. Only add a variable if the field exists and is useful.

**Links / drill-down.** Use **tag-based dashboard-link dropdowns** (`asDropdown`, `includeVars`, `keepTime`) to move between tiers (L1→`L2`; L2→`L1`,`L3`; L3→`L2`,`L3`). Use **panel data-links** for specific drill-downs that must carry context (`var-*` + time). Reference targets by stable **uid**.

**Time & annotations.** Consistent default range per tier; always expose an SLO-window view on L1. Wire a **deploy/change annotation** source so changes overlay on every time panel.

**Units, thresholds, color.** Set explicit units; set threshold steps (green/amber/red) on stats/gauges; reserve red strictly for SLO/limit breach; choose colorblind-safe palettes; put a limit/threshold line on saturation panels.

**Datasource-agnostic telemetry guidance.** This standard works whether you're on Prometheus/Thanos/Mimir (metrics), Loki/Elastic/Splunk/LogScale (logs), Tempo/Jaeger/Zipkin (traces), CloudWatch/Azure Monitor/Stackdriver (cloud), or SQL. Principles: **classify by the field that means the thing** (status code, route, result) — never by a proxy (log level, a shared key) that only looks right; **prefer metrics for L1/L2** (cheap, alertable) and **logs/traces for L3** (detail, causality); **count distinct work units** (request/job IDs) for "how many," not raw log lines; and **verify every query against live data** before trusting it.

**Visualization selection guide.**

| Signal | Recommended visualization |
|---|---|
| SLO compliance / availability | Stat or gauge (with target threshold) |
| Error budget remaining | Bar gauge / stat |
| Burn rate | Timeseries (multi-window series) |
| Traffic / rate | Timeseries (req/s) |
| Error rate | Timeseries (%) + stat |
| Latency | Percentile timeseries **+ heatmap** for distribution |
| Saturation / utilization | Timeseries + gauge, **with a limit line** |
| Breakdown by dimension | Table or bar gauge (top-N, sorted) |
| Dependency matrix / health | Status grid / table |
| Individual events | Logs panel, trace view, table with correlation IDs |

## 11. New-dashboard intake checklist (ask before building)
1. **Tier?** Overview (L1), triage (L2), or deep-dive (L3)?
2. **Scope & ownership?** What service/system; who owns it; name, folder, tags?
3. **User journey / critical path?** What does "healthy" mean to the user?
4. **SLIs/SLOs?** What are the SLIs, targets, and error-budget window? (Needed for L1.)
5. **Telemetry available?** Metrics, logs, traces? Which datasource(s)/uid (or `$datasource`)?
6. **Source-of-truth signals?** How are success/failure, latency, and saturation actually determined — confirm against code/config, not assumption.
7. **Methods coverage?** Can you fill RED (services), USE (resources), and the golden signals? Where are the gaps?
8. **Dimensions?** Which breakdowns matter (route, region, version, tenant, instance) → which become variables?
9. **Dependencies?** Upstreams/downstreams/externals to monitor and link to.
10. **Drill-down?** Parent (up) and children (down) — and is this reachable from its parent?
11. **Change visibility?** Is there a deploy/annotation feed to overlay?
12. **Open questions?** List what can't be decided yet — don't silently guess.

## 12. Definition of Done (every dashboard)
- [ ] Correct tier, name, folder, tags; owner assigned; stored as code/versioned.
- [ ] All fixed stacks for the tier present (empty ones explained, not deleted).
- [ ] Method coverage complete for the tier (RED/USE/Golden/SLO as applicable).
- [ ] Latency shown as percentiles (+ heatmap at L3); errors as rates; saturation against limits.
- [ ] Standard variables wired; only supported ones added; `All` works.
- [ ] Drill-down links in/out (tag dropdowns + data-links carrying var + time); reachable from parent.
- [ ] Deploy/change annotations overlaid; consistent units, thresholds, semantic color.
- [ ] Panel titles + descriptions per §10; no duplicates; no vanity panels.
- [ ] Every query verified against live data; source-of-truth signals confirmed (not proxies).
- [ ] SLO stack mirrors the alerting/burn-rate policy.
- [ ] Risks, assumptions, and open questions documented.

## 13. Anti-patterns (do NOT do these)
- **Averages for latency**, or a single latency line with no distribution.
- **Counts without rates** (can't compare across load).
- **Classifying by proxy** (log level, a shared timing key) instead of the true signal.
- **Wall-of-graphs** with no hierarchy; everything the same size; no clear "most important."
- **Dead-end panels** with no drill-down; orphaned dashboards not reachable from a parent.
- **Saturation without a limit line**; dual Y-axes; rainbow/ decorative color.
- **Vanity metrics** (raw INFO log counts, total-ever counters) that don't drive action.
- **Per-dashboard reinvention** — diverging stack names/layout so responders can't navigate by memory.
- **Hardcoded datasource uids / no templating** — breaks reuse and promotion across environments.

## 14. Observability maturity (how to grow into this)
1. **Level 1 — Logs only:** derive RED from logs; L3-heavy. (Where many teams start.)
2. **Level 2 — Metrics + SLOs:** add metric-based RED/USE and an SLO stack; build real L1/L2.
3. **Level 3 — Traces + exemplars:** link metrics→traces; L3 troubleshooting becomes causal.
4. **Level 4 — SLO-driven ops:** multi-burn-rate alerting, error-budget policy, dashboards-as-code, automated drill-down. 
You can adopt this standard at any level — fill what you have, mark the gaps, and close them over time.

## 15. Using the skeleton JSONs
`skeleton_L1_universal.json` / `_L2_` / `_L3_` each import cleanly (they use a `$datasource` variable — Grafana prompts you to pick one; nothing is hardcoded). Each contains: the **fixed stacks for its tier as rows**, the standard variables, **tag-based cross-tier link dropdowns**, the correct tier tags (+ a `TEMPLATE` tag so they don't appear in normal views), and in **every stack a text "what goes here" panel** (intent, method, recommended viz, example panels, required signals) beside example placeholder tiles showing the layout. To use: import → set name/folder/tags/uid/owner → replace placeholders with real panels for your system → wire variables and links → run the Definition of Done. The "Anthropic Requests L3" dashboard is a worked example of the L3 pattern (Errors stack split into true-failures vs platform-noise, distribution-based latency, troubleshooting with correlation IDs).
