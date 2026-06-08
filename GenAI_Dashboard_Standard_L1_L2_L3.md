# GenAI Observability Dashboard Standard — L1 / L2 / L3

**Purpose:** a reusable standard for building tiered observability dashboards on the GenAI platform (Grafana + Falcon LogScale / Prometheus). It defines the **fixed stacks (rows/sections)** each tier must contain and what each means, so every dashboard — current and future, any provider or subsystem — looks and behaves consistently. **Panels inside a stack flex** based on the data available; **the stacks and their meaning do not.**

**How to use it:** when a new dashboard request arrives, (1) decide the tier (L1/L2/L3), (2) run the intake checklist (§9), (3) clone the matching skeleton JSON, (4) fill each fixed stack with the panels the data supports, (5) check against the Definition of Done (§10). Use this doc together with the three skeleton files: `skeleton_L1_dashboard.json`, `skeleton_L2_dashboard.json`, `skeleton_L3_dashboard.json`.

---

## 1. The L1 → L2 → L3 model

| Tier | Question it answers | Audience | Scope | Default range |
|---|---|---|---|---|
| **L1** | *Is the platform healthy right now?* | On-call glance, leadership | Whole platform, SLO-level | now-6h |
| **L2** | *Which component / provider / model is the problem?* | On-call triage | Platform → component/provider/status summary | now-24h |
| **L3** | *What exactly is failing, where, and why?* | Engineer investigating | One provider/subsystem, deep dive | now-6h to 24h |

Each tier **drills into the next** via panel data-links that carry variables + the active time range. L1 panels link to L2; L2 panels link to the relevant L3; L3 links back up to L2 and across to sibling L3s.

## 2. Universal principles (apply to every tier)
1. **Observability, not monitoring.** A panel must help answer *what/where/why*, not just display a number. If a panel can't drive an action or a drill-down, cut it.
2. **Four golden signals** are mandatory coverage across the tier: **Throughput, Errors, Latency, Availability**. Lay them out in that priority, top-to-bottom.
3. **Signal must match the source of truth.** Classify by the field that actually means what you want (route/integration metadata, structured status codes) — never by a proxy (log level, a shared timing key) that only looks right. (Lesson: 4xx/429 were logged at INFO, so `levelname=ERROR` missed every 429.)
4. **Verify before trust.** Every query is run against live data before it ships. Anything unverified is labelled, not hidden.
5. **Fewer, meaningful panels.** No raw INFO/WARNING gauges, no duplicates, no "looks nice but doesn't debug." Every panel gets a description.
6. **Consistent identity.** Naming, tags, variables, links follow §8 so dashboards are discoverable and navigable as a set.

## 3. The "stack" concept
A **stack** is a named row representing one observability intent. Stacks are **fixed per tier** (name + meaning). The **panels** inside are chosen from what the data supports — e.g. an L3 "Errors" stack always means "true failures of this subsystem," but the panels may be status-code tables for one provider and exception-type tables for another. Keep the stack even if you can only fill it with one panel; if a stack genuinely cannot be filled, note why in a text panel rather than deleting it silently.

---

## 4. L1 stacks (tags `[L1]`)

| Stack (row) | Meaning / intent | Golden signal | Mandatory | Example panels (flex) | Drill-down |
|---|---|---|---|---|---|
| **1. SLO & Error Budget** | Are we within SLO; how much budget is left; are we burning it fast? | Availability/Errors | Yes | Availability SLO gauge, error-budget-used stat, burn-rate timeseries (multi-window) | → L2 |
| **2. Golden-Signal Topline** | Platform-wide error rate, p95 latency, throughput, availability at a glance | All four | Yes | Aggregate error-rate stat, p95 latency stat, requests/sec timeseries, availability stat | → L2 |
| **3. Component & Dependency Health** | Are the supporting systems healthy (compute, cache, DB, mesh)? | Availability | Yes | Pod count/restarts table, Redis/Cosmos memory gauges, dependency up/down | → L2 / L3 pods |

L1 rule: **summary only**. No per-request detail. Each tile links to the L2 view with `var-*` + time pre-filled.

## 5. L2 stacks (tags `[L2]`)

| Stack (row) | Meaning / intent | Golden signal | Mandatory | Example panels (flex) | Drill-down |
|---|---|---|---|---|---|
| **1. Component & Infra Metrics** | Per-component compute/cache/DB behaviour | Availability | Yes | Pod count (instant + history), restart reasons, Redis/Cosmos usage & IO | → L3 pods |
| **2. Errors & Status Summary** | Which provider / status code / project is erroring | Errors | Yes | Requests-by-status-code per provider (incl. an "Other" bucket), errors-vs-project / errors-vs-module breakdowns | → provider L3 |
| **3. Latency Summary** | Where is latency concentrated (provider/project) | Latency | Yes | p95 latency per provider per project (timeseries) | → provider L3 |
| **4. Throughput Summary** | Traffic distribution across providers/projects | Throughput | Recommended | Requests per provider/project (timeseries/table) | → provider L3 |

L2 rule: **break the platform down by the dimension that routes the investigation** (provider, status, project). Every provider should be reachable from here — add a per-provider panel + link for each new L3 (don't let a provider sit only inside an "Other" bucket).

## 6. L3 stacks (tags `[GenAI, <PROVIDER>, L3]`)

| Stack (row) | Meaning / intent | Golden signal | Mandatory | Example panels (flex) |
|---|---|---|---|---|
| **1. Overview / Golden Signals** | One-glance health of this subsystem | All four | Yes | Total · Successful · Failed · Error Rate · Availability (server-side SLO) · p95 Latency (stats) |
| **2. Request Volume (Throughput)** | How much traffic and where | Throughput | Yes | Count over time, by model/dimension (time + bar), by project |
| **3. Upstream Errors (source-of-truth signal)** | True failures **of this subsystem** | Errors | Yes | Error-rate over time, status codes over time, requests-by-status-code, errors-by-model |
| **4. Secondary / Platform Errors** | Broad/ancillary errors during these requests, clearly separated | Errors | If applicable | Platform error count over time, top platform error messages (labelled "includes sub-steps") |
| **5. Latency** | Latency distribution & where it concentrates | Latency | Yes if available | p50/p90/p95/p99 over time, latency by model/dimension |
| **6. Breakdown** | Which model/dimension/project is affected | All | Yes | Usage (project × model, distinct requests), success/failure ratio per model |
| **7. Troubleshooting** | Jump to the actual failing requests | — | Yes | Recent failures (with status code + correlation id), full filtered log explorer |

L3 rule: **separate the true-subsystem signal from ancillary noise** (stacks 3 vs 4), always count **distinct request ids** for "requests," and end with a **troubleshooting** stack that exposes correlation IDs.

---

## 7. Stack → golden-signal coverage (quick check)
| Golden signal | L1 | L2 | L3 |
|---|---|---|---|
| Throughput | Topline | Throughput Summary | Request Volume |
| Errors | SLO + Topline | Errors & Status Summary | Upstream + Platform Errors |
| Latency | Topline | Latency Summary | Latency |
| Availability | SLO & Error Budget | (derived in status summary) | Overview (server-side SLO) |
A tier is incomplete if any column is empty.

## 8. Cross-cutting conventions
**Dashboard names:** `<Scope> <Tier>` — e.g. "GenAI SRE Operations Dashboard L1", "Anthropic Requests" (L3 implied by tag/folder). **Folder:** the platform folder (e.g. GenAI platform).

**Row (stack) names:** use the fixed stack names above, verbatim where possible, so they read the same across dashboards.

**Panel titles:** `<Subject> <Metric> [by <Dimension>] [(qualifier)]` — e.g. "Anthropic Error Rate", "Anthropic p95 Latency by Model", "Anthropic Requests by Status Code". No vague titles ("Graph 1", "Errors"); no duplicate titles in one dashboard. Every panel has a **description** stating its purpose.

**Tags:** L1 → `[L1]`; L2 → `[L2]`; L3 → `[GenAI, <Provider>, L3]`. Tags drive the cross-tier link dropdowns, so they are mandatory.

**Variables (standard set):**
- `$datasource` (type=datasource) — never hardcode a datasource uid in a template.
- `$cluster`, `$namespace`, `$environment` (release), `$container` — the scoping chain.
- Dimension vars as the data supports: `$project`, `$model`, `$region`, `$status_code`. Use `includeAll` with `allValue = "*"` so "All" works via wildcard match (`field = "$var"`).
- Only add a variable if the underlying field exists and is useful. Keep values **raw** unless the source guarantees a normalized field (don't normalize only the dropdown — it desyncs from raw panel filters).

**Links / drill-down:**
- Use **tag-based dashboard-link dropdowns** (`asDropdown`, `includeVars`, `keepTime`) to hop between tiers — no hardcoded uids needed (L1→tag `L2`; L2→tags `L1`,`L3`; L3→tags `L2`,`L3`).
- Use **panel data-links** for specific drill-downs that must carry context, with `var-*` + time pass-through.
- Reference target dashboards by **uid** in data-links; keep uids stable.

**Datasource rules:** classify the dashboard's datasource up front. **Log-derived** signals → LogScale (LSQL); **native infra/cloud metrics** → Prometheus/Thanos (PromQL). **Never copy queries across datasources** — the same-looking metric may not exist (e.g. Azure `azure_cognitiveservices_*` exists for OpenAI but not for log-based providers).

**Query hygiene (hard-won lessons):**
- Count **distinct `request_id`** for "requests," not raw log lines.
- To aggregate everything into one row, use a **constant key** (`| key := "all" | groupBy([key], …)`) — an **empty `groupBy([])` returns no rows** in LogScale.
- `percentile(field=x, percentiles=[95])` outputs a field named **`_95`** (the `as=` is ignored for a list) — rename before `select`.
- Classify by **route + structured status code**, not log level.
- Provide an **explorer-ready** copy of queries (literals substituted, one filter per pipe, comments stripped) for manual verification.

## 9. New-dashboard intake checklist (ask these first)
1. **Tier?** Is this platform health (L1), component/provider summary (L2), or a subsystem deep-dive (L3)?
2. **Scope/subject?** Which platform, provider, or subsystem? What's the dashboard name + folder + tags?
3. **Datasource(s)?** LogScale, Prometheus/Thanos, or mixed? What's the datasource uid (or use `$datasource`)?
4. **Fields available?** Which fields/labels exist (request_id, status code, latency, model, project, region…)? Don't assume one provider's fields exist for another.
5. **Source-of-truth signals?** How is *success/failure* actually determined (status code? log level? structured field?) — confirm against code, not assumption.
6. **Golden-signal coverage?** Can each of the four be filled? If latency/status isn't available, which stacks become "noted but empty"?
7. **Dimensions?** What breakdowns matter (model, project, region) and which become variables?
8. **Drill-down?** What's the parent (up) and children (down)? Confirm the dashboard is reachable from its parent.
9. **SLO (L1/L2)?** What is the availability/error/latency SLO and the error-budget window?
10. **Open questions?** List anything that can't be decided without more info — don't silently guess.

## 10. Definition of Done (every tier)
- [ ] Correct tier, name, folder, and tags.
- [ ] All fixed stacks for the tier present (empty ones explained, not deleted).
- [ ] Four golden signals covered for the tier.
- [ ] Standard variables wired; only supported ones added; `All` works.
- [ ] Drill-down links in/out wired (tag dropdowns + data-links with var + time pass-through); reachable from parent.
- [ ] Panel titles + descriptions per §8; no duplicates; no noise panels.
- [ ] Every query verified against live data; explorer-ready copies provided.
- [ ] Source-of-truth signal confirmed (not a proxy).
- [ ] Risks, assumptions, and open questions listed.

## 11. Using the skeleton JSONs
Each skeleton imports cleanly (uses a `$datasource` variable, so Grafana prompts you to pick the datasource — nothing is hardcoded). It contains the fixed stacks for its tier as rows, the standard variables, tag-based cross-tier link dropdowns, the correct tags, and a **text panel in each stack** describing what goes there (purpose, golden signal, example panels, required fields) plus a placeholder viz panel showing the pattern. To use: import → set name/folder/tags/uid → replace placeholders with real panels for your subsystem → run the DoD checklist. Delete the text "guide" panels once the stack is filled (or keep one as inline documentation).
