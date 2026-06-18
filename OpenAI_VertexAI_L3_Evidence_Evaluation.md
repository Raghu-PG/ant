# OpenAI & Vertex AI L3 — Evidence Evaluation & Build Decision
Grounded entirely in the LogScale + Grafana/Thanos screenshots you ran (30-day window, prod). No generic suggestions — every panel below is backed by a query that returned real data.

## 1. Verdict matrix (what's implementable, and from where)

| Signal | OpenAI | Vertex | Confidence |
|---|---|---|---|
| Traffic / volume by model/deployment | **LS** route + `/deployments/` regex | **LS** `models/<model>` regex | High |
| Traffic by region | **AZ** `dimension_region` | **LS** `/locations/<region>/` regex | High |
| True error rate / status codes | **LS** `origin_status_code` ✅ present | **LS** `origin_status_code` ✅ present | High |
| Server vs client errors | **AZ** servererrors / clienterrors metrics | **LS** 5xx vs 4xx from status code | High |
| Throttling (429) | **AZ** RatelimitKey + **LS** `origin_status_code=429` | **LS** `origin_status_code=429` | High (OAI) / low-volume (VTX) |
| Real p95/p99 latency | **LS** `execution_stats.openAI` ✅ | **LS** `execution_stats.vertexAI` ✅ | High |
| Provider availability (SuccessRate) | **AZ** `successrate_percent_minimum` | — (no native metric) | High (OAI only) |
| Deployment/operation/api-version breakdown | **AZ** RatelimitKey / operationName / apiname | — | High (OAI only) |
| Project/tenant breakdown | **LS** `request_project_name` ✅ | **LS** `request_project_name` ✅ | High (note dup names) |
| Retry / fallback / timeout | **LS** keywords present, exact wording TBD | same | Medium |
| Token / quota usage metric | ❌ none | ❌ none (only `RESOURCE_EXHAUSTED` text) | n/a |
| Azure percentile latency | ❌ (avg only, no `le`) → use LS | n/a | n/a |

## 2. What each screenshot told us (interpreted)

**OpenAI route shape (D.4).** Multiple route families: `…/openai/foundry/openai/deployments/<deployment>/chat/completions` (e.g. `llama-3.3-70b-instruct`), `…/openai/deployments/<deployment>/embeddings` (e.g. `TEXT-EMBEDDING-3-SMALL`, `text-embedding-ada-002`), and `…/openai/v1/<hash>/v0/…/content|remix` (a responses-style API with **no** `/deployments/` segment). → Deployment extraction must use `/deployments/(?<deployment>[^/]+)/`; the `v0/content|remix` routes have no deployment and will (correctly) not appear in deployment panels.

**OpenAI status codes (D.5).** `origin_status_code` **is present**: **400=25,852 · 401=1,418 · 404=2 · 429=6,521 · 500=102 · 503=224.** Reading: OpenAI pain is overwhelmingly **client-side 400s** + **throttling 429s**; true provider 5xx is rare (326 total). This is a real RCA signal → a dedicated "4xx vs 429 vs 5xx" split is justified.

**OpenAI latency (D.6a/b).** Two independent extractions agree: `execution_stats.openAI` → p50 **0.46s** / p95 **8.16s** / p99 **24.3s**; "finished in Ns" → 0.49 / 8.32 / 23.87. Both work; I'll use the `'openAI'` exec-stats key (cleaner, 24M hits) for parity with Anthropic/Vertex.

**OpenAI projects (D.7).** Strong coverage (~97 projects). Dominant: CPI (62M), ConsumerPainpoints (635k), Cloud AI Assistant (628k), FPS Invoice Data Extraction (2.7M), Eureka (462k). **Data-quality note:** `CPIInnovationPlayground` appears multiple times (casing/variant fragmentation) — same class of issue as the GenAI project split; flag at source, keep raw.

**OpenAI failures sample (D.8).** Matching 429/timeout/5xx events exist in volume; the table showed metadata columns only (message body not captured) — so retry/timeout *wording* still needs one look at a raw message (amber).

**Azure labels (Grafana/Thanos) — casing settled definitively:**
- `dimension_region` (lowercase) ✅ returns all Azure regions (East US 2, Sweden Central, …). → the existing **"Total Server Errors in Region" panel using `dimension_Region` (capital R) is silently empty** — confirmed bug, fix to lowercase.
- `dimension_apiname` (lowercase) ✅ = **API version** ("Azure OpenAI API version 2024-10-21", …) — note many empty `{}` series.
- `dimension_operationName` (capital N) ✅ = **operation** (`chatcompletions_create`, `Embeddings_Create`, `Files_List`, `Models_List`, …). The Latency dashboard's `$Operation`/`dimension_operationname` (lowercase) is the wrong casing → rewire to capital-N.
- `dimension_RatelimitKey` (capital R/K) ✅ = **deployment** (`GPT-4O-2025-08-06`, `GPT-5-2025-08-07`, `GPT-01-2025-10-15-NO-CONTENT-FILTERS`, …). This is the Azure-side deployment dimension.
- Confirmed **absent**: percentile-latency, `le` buckets, token/quota, `StatusCode`, `ModelName` → percentiles & per-request detail come from LogScale.

**Vertex route + region (D.9).** `…/vertexai/v1beta1/projects/<proj>/locations/<region>/publishers/google/models/<model>:<method>`. Models: `gemini-3.1-flash-lite` (22.2M), `gemini-2.5-flash` (18.4M), `gemini-2.0-flash-001` (3.96M), `gemini-3.5-flash` (156k), `text-embedding-004`, `gemini-embedding-001`. Regions: `us-central1`, `any`. Publisher: `google`. Methods: `generateContent`, `streamGenerateContent`, `predict`. → region (`/locations/<r>/`), model (`models/<m>` up to `:`), and method are all extractable.

**Vertex status codes (D.10).** `origin_status_code` present: **404=45,105 · 400=13,969 · 499=85 · 429=37 · 403=10 · 401=8 · 500=2.** Reading: Vertex's dominant failure is **404 (Not Found)** by a wide margin, then **400**; 429/5xx are negligible. The 404 spike is a standout RCA signal (likely model-name/endpoint/`locations/any` mismatch) → **a dedicated "Vertex 404 by model/region/project" panel is warranted** (evidence-driven, not generic). This also means the *old* Vertex dashboard (only 429 panels) was watching the wrong thing.

**Vertex latency (D.11).** `execution_stats.vertexAI` works for non-Anthropic Vertex: p50 **2.08s** / p95 **21.06s** / p99 **48.16s** — materially slower than OpenAI (Gemini generateContent). Real percentiles available.

**Vertex quota/samples (D.12).** Events matching `RESOURCE_EXHAUSTED|quota|429|Timeout|5xx` exist (message body not captured) — quota panel is buildable but the exact `RESOURCE_EXHAUSTED` wording needs one raw-message look (amber).

**Vertex projects (D.13).** Strong: CKDP (33.6M), CoreGIS (18.1M), Eureka (664k), FPS Invoice (7.3M), Dossier (24k). **Data-quality note:** comma-concatenated names appear (`Cosmos, Cosmos`, `Cosmos, Cosmos, Cosmos`) — a real fragmentation/serialization bug → flag at source, keep raw.

**Retry/fallback/timeout (D.14).** Keywords `retry|fallback|timeout` do appear (grouped by request_url, one Gemini route at 8,941). So the signal exists, but I haven't seen the exact field/wording → Rows for retry/fallback are **best-effort** until one sample message is shared.

## 3. Corrections to the existing dashboards (carry into the rebuild)
1. **Fix label casing:** use `dimension_region`, `dimension_apiname`, `dimension_operationName`, `dimension_RatelimitKey`. Removes the silently-empty "Total Server Errors in Region" panel bug.
2. **Delete the fake p95:** `histogram_quantile(0.95, … latency_milliseconds_average …)` is invalid (average gauge, no `le`). Replace with LogScale percentiles.
3. **Rewire `$Operation`:** `query_result()` placeholder → `label_values(azure_cognitiveservices_accounts_clienterrors_count_total{rg}, dimension_operationName)`.
4. **Vertex error model:** replace the old `levelname=ERROR + message=/429/` with `origin_status_code [45]\d{2}` (now confirmed) and lead with **404**, not 429.
5. **Counting:** distinct `request_id` for "requests"; constant-key `groupBy([key])` for single-value KPIs; `percentile(…)`→ rename `_95` (all proven on Anthropic).

## 4. Confidence buckets
- **Green (build now):** OpenAI — SuccessRate, total/server/client (AZ); status-code split, latency percentiles, by-deployment, by-project, by-region, 400/429 breakdowns (LS). Vertex — total/success/failed/error-rate/availability, status codes incl **404 RCA**, latency percentiles, by-model, by-region, by-project, 429 (LS).
- **Amber (build, label "validate"):** retry/fallback/timeout rows; Vertex `RESOURCE_EXHAUSTED` quota panel; OpenAI token/quota (no metric — omit, note).
- **Red (cannot do from this data):** Azure-native percentiles, Azure token/quota/StatusCode dims — explicitly omitted, percentiles sourced from logs instead.

→ Both dashboards are **green to build now.** Section 5 (in the JSON files) implements exactly this. The two production JSONs are generated alongside this doc.
