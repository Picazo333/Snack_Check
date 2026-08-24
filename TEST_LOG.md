# TEST_LOG — SnackCheck - Nutrition Check

**Workflow ID:** `k6eh14EFRNlU5GUv`  
**How tests were run:** n8n **manual / MCP** `execute_workflow` (`mode: manual`). **No production webhook execution** is recorded for TC-001–008.  
**HTTP status / Content-Type:** **not** captured from a production HTTP request. Documented as **verified from the Respond to Webhook node that ran** (node configuration + last node in the execution).  
**PASS** only when that case **succeeded**. Groq-credential errors are failed attempts, not PASS.

**Observed n8n workflow execution duration** = `stoppedAt − startedAt` from execution metadata. This is **not** an external HTTP latency benchmark.

| Exec | Started (UTC) | Stopped (UTC) | Status | Observed n8n duration | Maps to |
|------|---------------|---------------|--------|----------------------|---------|
| 5 | 2026-08-24T05:03:39.325Z | 2026-08-24T05:03:39.437Z | success | 112 ms | TC-001 |
| 6 | 2026-08-24T05:03:51.738Z | 2026-08-24T05:03:51.799Z | success | 61 ms | TC-002 |
| 7 | 2026-08-24T05:03:52.483Z | 2026-08-24T05:03:52.529Z | success | 46 ms | TC-003 |
| 8 | 2026-08-24T05:03:53.711Z | 2026-08-24T05:03:54.073Z | success | 362 ms | TC-004 |
| 9 | 2026-08-24T05:04:24.128Z | 2026-08-24T05:04:26.534Z | error | 2406 ms | TC-005 attempt 1 |
| 10 | 2026-08-24T05:04:25.574Z | 2026-08-24T05:04:26.558Z | error | 984 ms | TC-006 attempt 1 |
| 11 | 2026-08-24T05:04:27.442Z | 2026-08-24T05:04:27.768Z | success | 326 ms | TC-008 |
| 12 | 2026-08-24T05:04:28.145Z | 2026-08-24T05:04:28.279Z | error | 134 ms | TC-007 attempt 1 |
| 13 | 2026-08-24T05:19:07.175Z | 2026-08-24T05:19:10.506Z | success | 3331 ms | TC-005 attempt 2 |
| 14 | 2026-08-24T05:19:37.651Z | 2026-08-24T05:19:40.196Z | success | 2545 ms | TC-006 attempt 2 |
| 15 | 2026-08-24T05:19:52.553Z | 2026-08-24T05:19:53.399Z | success | 846 ms | TC-007 attempt 2 |

Executions `2`–`4` were earlier Stage 1 manual runs. They are **not** claimed as TC-001–008.

Coverage labels: **functional**, **integration**, **error**, **performance** (duration note only).

---

### TC-001

| Field | Value |
|-------|--------|
| Type | Error + functional (usage contract) + performance |
| Input | `{}` |
| Expected | Usage docs branch; configured 200 JSON on `Respond - Usage Documentation` |
| Actual | Last node `Respond - Usage Documentation`; exec `5` success |
| Status | **PASS** |
| Observed n8n duration | 112 ms |
| Notes | Groq did not run. Manual MCP only. |

---

### TC-002

| Field | Value |
|-------|--------|
| Type | Error + functional |
| Input | `{"foo":"bar"}` |
| Expected | Missing-barcode branch; configured 400 JSON (`error: validation_error`) |
| Actual | `Respond - Missing Barcode`; exec `6` success |
| Status | **PASS** |
| Observed n8n duration | 61 ms |
| Notes | Non-empty body without `barcode`. Manual MCP only. |

---

### TC-003

| Field | Value |
|-------|--------|
| Type | Error + functional |
| Input | `{"barcode":"ABC123"}` |
| Expected | Invalid-barcode branch; configured 400 JSON (`error: validation_error`) |
| Actual | `Respond - Invalid Barcode`; exec `7` success |
| Status | **PASS** |
| Observed n8n duration | 46 ms |
| Notes | Validation only. Manual MCP only. |

---

### TC-004

| Field | Value |
|-------|--------|
| Type | Integration (Open Food Facts) + error |
| Input | `{"barcode":"0000000000000"}` |
| Expected | Product-not-found branch; configured 404 JSON (`product_not_found`) |
| Actual | OFF `status: 0`; `Check - Product Found` FALSE; `Respond - Product Not Found`; exec `8` success |
| Status | **PASS** |
| Observed n8n duration | 362 ms |
| Notes | OFF may rewrite the code (e.g. leading zeros); still `status != 1`. Groq did not run. Manual MCP only. |

---

### TC-005

| Field | Value |
|-------|--------|
| Type | Functional + integration (OFF + Groq) + performance |
| Input | `{"barcode":"3017620422003"}` (Nutella) |
| Expected | Complete path; `overall_verdict` unhealthy; Groq executes; configured 200 `text/plain` on `Respond - Health Verdict` |
| Actual (PASS run) | Exec `13`: `concern_score` 2; `traffic_light_reading` **unhealthy**; `nutriscore_grade` **e**; `nutriscore_reading` **unhealthy**; `overall_verdict` **unhealthy**; Groq yes; last node `Respond - Health Verdict`; formatted card started `🔴 UNHEALTHY — Nutella` |
| Status | **PASS** (exec `13` only) |
| Observed n8n duration | **3331 ms** (exec `13`) |
| Notes | Exec `9` **error** (2406 ms): `Model - Groq Health Writer` — “Node does not have any credentials set”. Classification on `9` already unhealthy; not PASS. On `13`, Groq `finish_reason: length` and visible LLM `text` was empty; formatted nutrition card still returned. Manual MCP only. |

---

### TC-006

| Field | Value |
|-------|--------|
| Type | Functional (Nutri-Score override vs traffic lights) + integration |
| Input | `{"barcode":"5449000000996"}` (Coca-Cola) |
| Expected | `concern_score` may be 0; traffic-light may be healthy; Nutri-Score reading unhealthy; **overall MUST be unhealthy**; Groq; configured 200 `text/plain` |
| Actual (PASS run) | Exec `14`: `concern_score` **0**; `traffic_light_reading` **healthy**; `nutriscore_grade` **e**; `nutriscore_reading` **unhealthy**; `overall_verdict` **unhealthy**; Groq yes; last node `Respond - Health Verdict` |
| Status | **PASS** (exec `14` only) |
| Observed n8n duration | **2545 ms** (exec `14`) |
| Notes | **This case proves Nutri-Score can override traffic-light severity.** Healthy traffic-light + `concern_score` 0 did **not** yield a healthy overall verdict; Nutri-Score **e** / unhealthy won. Exec `10` failed Groq credentials (984 ms) after the same override was already computed; not PASS. Groq visible `text` empty on `14` (`finish_reason: length`). Manual MCP only. |

---

### TC-007

| Field | Value |
|-------|--------|
| Type | Functional + integration (live Nutri-Score A + complete nutrients) |
| Input | `{"barcode":"6111035000430"}` (Sidi Ali; live OFF Nutri-Score **a**, sugar/salt/fat **0**) |
| Expected | Traffic-light healthy; Nutri-Score healthy; overall healthy; encouraging tone; configured 200 `text/plain` |
| Actual (PASS run) | Exec `15`: `traffic_light_reading` **healthy**; `nutriscore_reading` **healthy**; `overall_verdict` **healthy**; `Prepare - AI Verdict Encouraging` `selected: true`; Groq yes (visible text started `Great choice! Sidi Ali is a healthy, zero‑sugar…`, cut at `finish_reason: length`); last node `Respond - Health Verdict` |
| Status | **PASS** (exec `15` only) |
| Observed n8n duration | **846 ms** (exec `15`) |
| Notes | Numeric zero nutrients accepted as complete. Exec `12` **error** (134 ms) Groq credential missing; classification already healthy; not PASS. Manual MCP only. |

---

### TC-008

| Field | Value |
|-------|--------|
| Type | Error + integration (incomplete nutriments) + functional (Groq skip) |
| Input | `{"barcode":"6111035002175"}` |
| Expected | Insufficient-data branch; configured 200 JSON `insufficient_data`; Groq must not execute |
| Actual | Exec `11`: `Check - Nutrition Data Complete` FALSE; `Respond - Insufficient Nutrition Data`; `AI - Health Verdict` / `Model - Groq Health Writer` **absent** from run data |
| Status | **PASS** |
| Observed n8n duration | 326 ms |
| Notes | Product found (`status: 1`) but sugar/salt/fat not extractable as finite numbers. Manual MCP only. |

---

## Summary

| ID | Status | Observed n8n duration (PASS run) | Coverage |
|----|--------|----------------------------------|----------|
| TC-001 | PASS | 112 ms | error, functional, performance |
| TC-002 | PASS | 61 ms | error, functional |
| TC-003 | PASS | 46 ms | error, functional |
| TC-004 | PASS | 362 ms | integration, error |
| TC-005 | PASS | 3331 ms | functional, integration, performance |
| TC-006 | PASS | 2545 ms | functional (Nutri-Score override), integration |
| TC-007 | PASS | 846 ms | functional, integration |
| TC-008 | PASS | 326 ms | error, integration, functional (no Groq) |

**Not executed:** production webhook POST (or any production webhook execution of TC-001–008). All PASS rows are manual MCP executions.

**Performance note:** times above are **observed n8n workflow execution duration** (`stoppedAt − startedAt`), not an external HTTP latency benchmark. Validation-only 46–112 ms; OFF 404 path 362 ms; insufficient data 326 ms; full Groq path 846–3331 ms.

**Failed attempts (not PASS):** exec `9`, `10`, `12` — Groq credential missing. Credential was attached in the n8n UI before `13`–`15`.
