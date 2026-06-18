# OpenAI & Vertex AI — L3 Deep-Dive Dashboard Design
### Follows the universal L3 standard (8 stacks: Golden Signals → Traffic → Errors → Latency → Saturation/Throttling → Dependencies/Retries → Breakdown → Troubleshooting)

This is the **design + data-request** phase (same approach we used for Anthropic: design → confirm fields in LogScale/Grafana → generate verified JSON). Nothing is built until the §"Data I need" items are confirmed. Each panel is tagged by source — **[AZ]** = Prometheus/Azure metric, **[LS]** = Falcon LogScale — and by status — **[confirmed]** (seen in existing JSON) or **[verify]** (needs your confirmation).

---

## A. What exists today (grounding)

**OpenAI** is on **two** telemetry sources:
- **[AZ] Azure metrics** (`azure_cognitiveservices_accounts_*`, resourceGroup `az-rg-aip-mlwgenaiplatform`): `successrate_percent_minimum`, `azureopenairequests_count_total`, `servererrors_count_total`, `clienterrors_count_total`, `latency_milliseconds_average`. Dimension labels seen: `dimension_region`/`dimension_Region`, `dimension_apiname`, `dimension_operationName`/`operationname`, `dimension_RatelimitKey`/`ratelimitkey`. **Note:** label case is inconsistent across the current panels — a real bug to standardize. Azure latency is **average only** (no native percentiles).
- **[LS] LogScale** logs the same OpenAI calls via the platform (`ipa-api-server`, the `/openai/` route, `execution_stats.openAI` latency key, `origin_status_code`, `request_project_name`, `request_id`).

**Vertex AI** is on **LogScale only** — identical field model to Anthropic (route + `origin_status_code` + `execution_stats.vertexAI` + project/model/request_id). The current Vertex dashboard is **only** "429 throttling" panels (6 of them) — no traffic, success rate, latency, or status breakdown.

**Design consequence:** OpenAI L3 = **hybrid** (Azure = provider-side truth; LogScale = platform-side per-request truth). Vertex L3 = **LogScale**, reusing the proven Anthropic L3 pattern. The hybrid split is what lets OpenAI answer *"is this provider-side, platform-side, or client-side?"* directly.

---

## B. OpenAI L3 Dashboard

### B.1 Objective
A deep-dive for OpenAI incidents that lets an on-call engineer, in minutes: confirm whether OpenAI is healthy (Azure SuccessRate), see if the symptom is **provider-side** (Azure 5xx / SuccessRate drop / throttling by RatelimitKey), **platform-side** (our `origin_status_code`, retries, execution latency), or **client-side** (Azure 4xx / bad requests); isolate failures to a **region / deployment / operation / model / project**; investigate **latency** with real p95/p99 (from LogScale, since Azure gives only average); detect **throttling/quota/429**; and drill to the **failing requests** with correlation IDs.

### B.2 Sections (rows) and panels

**Row 1 — Overview / Golden Signals (provider + platform side-by-side)**
| Panel | Src | Viz | Query / metric (high level) | Group-by | Threshold & why-in-incident |
|---|---|---|---|---|---|
| OpenAI SuccessRate (provider) | AZ | stat/gauge | `min(successrate_percent_minimum{rg})` | — | <99 amber, <95 red. The provider-reported health headline. |
| Total Requests | AZ | stat | `sum(sum_over_time(azureopenairequests_count_total[$__range]))` | — | Throughput baseline; sudden drop = upstream/ingress issue. |
| Server Errors (5xx) | AZ | stat (red) | `sum(sum_over_time(servererrors_count_total[$__range]))` | — | >0 trending = provider-side fault. |
| Client Errors (4xx) | AZ | stat (amber) | `sum(sum_over_time(clienterrors_count_total[$__range]))` | — | High = client/request problem, not provider. |
| Platform Error Rate | LS | stat % | `origin_status_code>=400` distinct request_id / total *100 | — | What *our platform* observed; compare to Azure to spot platform-only failures. |
| p95 Latency | LS | stat (ms/s) | p95 of `execution_stats.openAI` | — | Azure gives only avg; true tail comes from logs. |

**Row 2 — Traffic & Demand**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| Requests over time by region | AZ | timeseries | `sum by(dimension_region)(rate(azureopenairequests_count_total[$__rate_interval]))` | region | Regional traffic shifts / drops. |
| Requests by deployment/model over time | LS | timeseries | route/deployment regex, timeChart count | deployment/model | Which deployment carries load. |
| Requests by Operation (ApiName) | AZ | bargauge | `sum by(dimension_apiname)(...requests_count_total)` | apiname/operation | Chat vs embeddings vs completions mix. |
| Requests by project/tenant | LS | bargauge | groupBy `request_project_name` | project | Which app drives traffic. |

**Row 3 — Errors (taxonomy: provider vs client vs platform)**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| Server vs Client error rate over time | AZ | timeseries (2 series) | `rate(servererrors...)` and `rate(clienterrors...)` | — | Instantly separates 5xx (provider) from 4xx (client). |
| OpenAI requests by status code | LS | table | `origin_status_code` groupBy | status_code | True per-status distribution incl 429. |
| Server Errors by Region | AZ | timeseries/bargauge | `sum by(dimension_region)(sum_over_time(servererrors...))` | region | Is a region degraded. |
| Client Errors by Operation / ApiName | AZ | bargauge | `sum by(dimension_operationName)(clienterrors...)` | operation | Which API surface is erroring. |
| Errors by deployment/model (error %) | LS | table | per-deployment failed/total | deployment/model | Pinpoint a bad deployment. |
| Top platform error messages | LS | table | `levelname=ERROR` groupBy(message) head 20 | message | Names the failure (labelled "platform/sub-step"). |

**Row 4 — Latency (distribution; real percentiles from logs)**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| p50/p90/p95/p99 over time | LS | timeseries | `percentile(execution_stats.openAI,[50,90,95,99])` over time | — | Tail latency trend (Azure can't do this). |
| Latency heatmap | LS | heatmap | bucketed `execution_stats.openAI` | — | Distribution shape / bimodality. |
| Avg latency by region (provider) | AZ | timeseries | `avg by(dimension_region)(avg_over_time(latency_milliseconds_average...))` | region | Provider-reported regional latency (avg only — labelled). |
| p95 latency by deployment/model | LS | timeseries | p95 per deployment | deployment/model | Which deployment is slow. |

**Row 5 — Throttling / Rate-Limit / Quota (Saturation analog)**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| 429 over time | LS | timeseries | `origin_status_code=429` timeChart | — | Throttling onset/trend. |
| Errors by RatelimitKey | AZ | bargauge | `sum by(dimension_RatelimitKey)(servererrors+clienterrors)` | RatelimitKey | Localizes throttling to a quota bucket/deployment. |
| SuccessRate by Region | AZ | stat/timeseries | `min by(dimension_region)(successrate...)` | region | Regional throttling/health. |
| Token / quota usage *(if metric exists)* | AZ | timeseries | token-usage metric vs quota | deployment | Are we near token quota → throttling cause. **[verify metric]** |

**Row 6 — Retry / Fallback / Timeout (Dependency behavior)** — all **[LS, verify fields]**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| Retries over time | LS | timeseries | count of retry log lines | deployment | Retry storms mask/aggravate incidents. |
| Timeouts | LS | stat/timeseries | timeout indicator in message | deployment | Timeout vs hard error distinction. |
| Fallback invocations | LS | table | fallback indicator | from→to | Are we silently degrading to another deployment/provider. |

**Row 7 — Breakdown**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| Success/failure by deployment/model | LS | table | total/success/failed/% | deployment | Per-deployment reliability. |
| SuccessRate by Region (table) | AZ | table | `min by(dimension_region)` | region | Regional reliability ranking. |
| Usage by project × deployment | LS | table | groupBy distinct request_id | project,deployment | Who uses what. |

**Row 8 — Troubleshooting** — **[LS]**
| Panel | Src | Viz | Query | Group-by | Why |
|---|---|---|---|---|---|
| Recent failed OpenAI requests | LS | table | `origin_status_code>=400` select asctime,status_code,route,project,request_id,message | — | Jump to real failures with correlation IDs. |
| OpenAI log explorer | LS | logs/table | full `/openai/` stream, livez excluded | — | Free-form investigation. |

### B.3 OpenAI — variables
`$datasource` (or two: one Prometheus, one LogScale), `$environment`/`$release`, `$region` (from `dimension_region` via `label_values`), `$deployment`/`$ratelimitkey`, `$operation` (apiname), plus `$project`/`$model` (LogScale). Standardize the **dimension label case** first (see data request).

---

## C. Vertex AI L3 Dashboard  (LogScale; Google/Gemini + non-Anthropic publishers; **excludes** `publishers/anthropic`)

### C.1 Scope decision (no double-count)
Vertex L3 = `cloud == "vertexai"` **AND** `request_url != */anthropic/*`. This covers Google/Gemini (and any other Vertex publishers) and explicitly leaves the `publishers/anthropic` traffic to the Anthropic L3, so the two dashboards never double-count. Documented at the top of the dashboard.

### C.2 Objective
Same depth as Anthropic L3, for native Vertex models: health at a glance, true error rate (`origin_status_code`), 429/`RESOURCE_EXHAUSTED` quota detection, real p95/p99 latency (`execution_stats.vertexAI`), and isolation by **model / region (location) / project**, ending in request-level drilldown.

### C.3 Sections (reuses the proven Anthropic L3 pattern, generalized beyond claude-*)
**Row 1 — Overview / Golden Signals** [LS]: Total (distinct request_id) · Successful · Failed (`origin_status_code` 4xx/5xx) · Error Rate · Availability (5xx-only SLO) · p95 Latency (`execution_stats.vertexAI`).
**Row 2 — Traffic & Demand** [LS]: count over time · by model (gemini-*, etc.) · by **region/location** (extract from `/locations/<region>/` in route — **[verify]**) · by project.
**Row 3 — Errors** [LS]: error rate over time · requests by `origin_status_code` · 429/500/502/503 over time · errors by model (error %) · top platform error messages (labelled).
**Row 4 — Latency** [LS]: p50/p90/p95/p99 over time · heatmap · p95 by model.
**Row 5 — Throttling / Quota** [LS]: 429 over time · 429 by model · `RESOURCE_EXHAUSTED`/quota-exceeded count (Vertex's quota signal — **[verify message shape]**).
**Row 6 — Retry / Fallback / Timeout** [LS, verify]: retries over time · timeouts · fallback invocations.
**Row 7 — Breakdown** [LS]: usage project × model (distinct requests) · success/failure ratio by model · by region.
**Row 8 — Troubleshooting** [LS]: recent failed Vertex requests (`origin_status_code`, with status_code + request_id) · log explorer.

### C.4 Panel-level notes
Every panel follows the Anthropic L3 idioms we already verified: distinct `request_id` counting; failure by `origin_status_code` `[45]\d{2}` (not log level); constant-key `groupBy([key])` for single-value KPIs; `percentile(...)` → rename `_95`; `execution_stats.vertexAI` regex for latency; route classification not the shared timing key. New for Vertex: a **region/location** dimension if `/locations/<region>/` is present, and a generalized model regex (`models/(?<model>[^/@:"]+)` rather than `claude-…`).

### C.5 Vertex — variables
`$datasource`, `$cluster`, `$namespace`, `$release`, `$container`, `$project`, `$model`, and `$region` (if location is extractable).

---

## D. Data I need from you (to build verified JSONs)

Keep it to what actually changes the build. Grouped by provider; run in Grafana (Prometheus) or the LogScale explorer and paste results/screenshots.

### D.1 OpenAI — Azure/Prometheus side
1. **Metric inventory** — which `azure_cognitiveservices_accounts_*` series exist? In Grafana → Explore (Prometheus), run:
   `count by (__name__)({__name__=~"azure_cognitiveservices_accounts_.*", resourceGroup="az-rg-aip-mlwgenaiplatform"})` and paste the metric names. (Looking specifically for: any **percentile latency**, **token/quota** metrics e.g. `*processedinferencetokens*`, `*generatedtokens*`, `*tokentransaction*`, and a **StatusCode** metric.)
2. **Dimension labels (and their exact case)** — for each of successrate / requests / servererrors / clienterrors / latency, run:
   `label_values(azure_cognitiveservices_accounts_servererrors_count_total{resourceGroup="az-rg-aip-mlwgenaiplatform"}, <label>)` and tell me which of these exist and the **exact casing**: `region`, `apiname`, `operationName`, `RatelimitKey`, `ModelName`, `ModelDeploymentName`, `StatusCode`. (The current panels mix `dimension_region`/`dimension_Region` etc. — I need the real names.)
3. One screenshot of a label dropdown (or `count by(dimension_RatelimitKey)(...)`) so I can see what RatelimitKey values look like (deployment names? quota buckets?).

### D.2 OpenAI — LogScale side
Run these (use the explorer-ready style; substitute prod values):
4. **Route shape** — `... | parseJson(message) | request_url = */openai/* | groupBy(request_url, function=count()) | head(10)` — paste a few request_url examples (I need the deployment/model/operation segments).
5. **Status codes** — `... | request_url=*/openai/* | regex(field=message,"origin_status_code=(?P<sc>\d{3})") | groupBy(sc)` — confirm `origin_status_code` is present for OpenAI failures.
6. **Latency key** — `... | request_url=*/openai/* | "execution statistics:" | regex(field=message,"'openAI':\s*(?<l>\d+\.\d+)") | percentile(field=l,percentiles=[95])` — confirm `openAI` key exists and the unit (s vs ms).
7. **Project coverage** — `... | request_url=*/openai/* | groupBy(request_project_name)`.
8. **Sample failure logs** — 2-3 raw messages for: a 429, a 5xx, and a timeout on OpenAI (so I can confirm retry/timeout/fallback fields).

### D.3 Vertex AI — LogScale side
9. **Route + region** — `... | parseJson(message) | request_url=*vertexai* | request_url!=*/anthropic/* | groupBy(request_url, function=count()) | head(10)` — paste examples; confirm `/locations/<region>/` and `publishers/<pub>/models/<model>` segments and which publishers appear (google, meta, mistral, …).
10. **Status codes** — same `origin_status_code` check for Vertex (exclude anthropic).
11. **Latency** — confirm `execution_stats.vertexAI` percentile works for non-anthropic Vertex.
12. **Quota signal** — sample of a Vertex 429 message (looking for `RESOURCE_EXHAUSTED` / "quota exceeded" text) and a 5xx + a timeout.
13. **Project coverage** — `... vertexai (not anthropic) | groupBy(request_project_name)`.

### D.4 Both — retry/fallback/timeout (only if you want Rows 5–6 fully)
14. Do the app logs emit explicit **retry / fallback / timeout** markers? Share 1-2 sample messages, or say "no" and I'll derive what I can from status codes only.

> If any of D.1–D.13 is unavailable, that stack becomes "noted but empty" (per the standard) rather than guessed.

---

## E. Final JSON creation plan
1. **You provide D.1–D.14** (or confirm what's unavailable).
2. I produce **explorer-ready LSQL + PromQL** for every panel and you verify them (same loop as Anthropic: paste results / "No results" → I fix). Two short verification files: `OpenAI_L3_Queries_to_verify.md`, `Vertex_L3_Queries_to_verify.md`.
3. After verification, I generate two production JSONs — `OpenAI_Requests_L3_v2.json` (hybrid; new uid, e.g. `openai-req-l3`) and `VertexAI_L3_v2.json` (new uid, e.g. `vertexai-req-l3`) — each: 8 standard L3 stacks, tags `[GenAI, OpenAI/VertexAI, L3]`, `$datasource`-based (no hardcoded uids), back-link to L2 + L3 tag dropdown, deploy annotations, percentile latency, distinct-request counting, source-of-truth error classification. Validated for unique IDs / no gridPos overlaps before delivery.
4. Deployment + cross-verification per the existing `Anthropic_L3_Deployment_Guide.md` pattern (new uids coexist with the old OpenAI/Vertex dashboards until sign-off).

### Quick-start (minimum to unblock me)
If you can only send a few things first, send: **D.1 (metric list), D.2 (label casing), D.4 (OpenAI route examples), D.9 (Vertex route examples), and one sample failure log each (D.8, D.12).** That alone lets me lock both designs and start the verification files.
