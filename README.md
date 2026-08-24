# SnackCheck

n8n webhook automation that turns a packaged-food barcode into a shopper-facing nutrition verdict.

Inspected 2026-08-23 (local) / 2026-08-24 UTC against n8n Cloud workflow `k6eh14EFRNlU5GUv`.

| | |
|---|---|
| Workflow | `SnackCheck - Nutrition Check` |
| ID | `k6eh14EFRNlU5GUv` |
| Instance | `https://mbpicazo.app.n8n.cloud` |
| Status | Active / published (`active: true`) |
| Active version | `ea851cae-35fa-496e-91aa-2a9af2566a16` |
| Nodes | 32 (26 executable + 6 sticky notes) |
| Production webhook | `https://mbpicazo.app.n8n.cloud/webhook/nutrition-check` |
| Test webhook | `https://mbpicazo.app.n8n.cloud/webhook-test/nutrition-check` |

The workflow `description` string still says “Inactive.” That is leftover metadata. The live flag is `active: true`.

**Evidence note:** TC-001–008 were **n8n manual / MCP executions**, not calls to the published production webhook. This document does **not** claim a production HTTP test. HTTP status and `Content-Type` below come from **Respond to Webhook node configuration**, and from which Respond node ran in those manual executions.

## Purpose

Accept a **POST** JSON barcode, look up the product on **Open Food Facts**, classify sugar / salt / fat with UK-style traffic lights plus Nutri-Score, set `overall_verdict` to the **worse** of those two readings, then return a shopper card. Groq writes two English sentences of copy. It does not recode the verdict.

Configured success card: **200** and `Content-Type: text/plain; charset=utf-8` on `Respond - Health Verdict`.

This is not medical or dietary advice.

## How It Works

Trigger: `Receive - Nutrition Check Webhook` — **POST**, path `nutrition-check`, `responseMode: responseNode`.

**Stage 1 — validation**

1. `Check - Empty Body` — `Object.keys($json.body ?? {}).length === 0` → `Respond - Usage Documentation` (configured **200** JSON: `service`, `description`, `method`, `path`, `required_field`, `example`).
2. Else `Validate - Barcode Present` (`$json.body?.barcode ?? $json.barcode`, not undefined / null / blank) → `Respond - Missing Barcode` (configured **400** JSON, `error: validation_error`).
3. `Validate - Barcode Numeric` — trimmed value matches `^\d+$` → `Respond - Invalid Barcode` (configured **400** JSON).

**Stage 2 — Open Food Facts**

4. `API Call - Open Food Facts Product` — HTTP Request (method omitted; node default GET)  
   `https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments`  
   with `neverError`, `fullResponse`, `alwaysOutputData`, `onError: continueRegularOutput`.
5. `Check - Product Found` — `($json.body?.status ?? $json.status) == 1` → `Respond - Product Not Found` (configured **404** JSON, `error: product_not_found`).
6. `Extract - Nutrition Fields` maps OFF `sugars_100g` / `salt_100g` / `fat_100g` to `sugar_g` / `salt_g` / `fat_g`, and `nutriments['energy-kcal_100g']` to `energy_kcal`. `nutriscore_grade` is a–e or `unknown`.
7. `Check - Nutrition Data Complete` requires sugar, salt, and fat present and finite (`0` is valid; energy optional). FALSE → `Respond - Insufficient Nutrition Data` (configured **200** JSON, `status: insufficient_data`). Groq is not on this branch.

**Stage 3 — deterministic classification**

8. `Classify - Nutrient Traffic Lights`: sugar `< 5` low, `<= 22.5` medium, else high; salt high `> 1.5` else `within_threshold`; fat high `> 17.5` else `within_threshold`.
9. `Calculate - Concern Score` (Code, `runOnceForEachItem`) — count of flags equal to `high` only. **Not** `overall_verdict`.
10. `Classify - Traffic Light Reading`: concern 0 healthy, 1 moderate, else unhealthy.
11. `Classify - NutriScore Reading`: a/b healthy, c moderate, d/e unhealthy, else unknown.
12. `Decide - Overall Verdict`: if Nutri-Score reading is `unknown`, use traffic-light only; otherwise take the worse of the two readings.

**Stage 4 — copy**

13. `Prepare - AI Verdict Encouraging` (`selected` when `overall_verdict !== 'unhealthy'`) and `Prepare - AI Verdict Cautionary` (`selected` when `overall_verdict === 'unhealthy'`) both always run. Prompt text differs (encouraging vs cautionary / occasional consumption). Both state classification is FINAL.
14. `Merge - AI Tone Candidates` (no `mode` in parameters; Merge v3.2 default **append**) → `Select - Active AI Tone` (TRUE output only; FALSE unwired).
15. `AI - Health Verdict` (prompt = `$json.ai_prompt`) + `Model - Groq Health Writer` (`openai/gpt-oss-20b`, temperature `0.2`, `maxTokensToSample` `200`). Groq credential is referenced as `Groq account` (secret not in workflow JSON).
16. `Format - Shopper Verdict` builds `verdict_text` from `$('Select - Active AI Tone').item.json` plus `$json.text || $json.output`.
17. `Respond - Health Verdict` — configured **200**, body `$json.verdict_text`, header `Content-Type: text/plain; charset=utf-8`.

## Setup

1. n8n Cloud with this workflow **published** (this instance: `mbpicazo.app.n8n.cloud`).
2. On `Model - Groq Health Writer`, attach a Groq credential in the n8n UI. Do not put API keys in node parameters or in these files.
3. Open Food Facts is unauthenticated HTTP.
4. The production URL is registered while `active: true`. Manual MCP `execute_workflow` does not send a live HTTP request and does not return a live HTTP envelope.

## Usage

The webhook node accepts **POST** only.

Example (illustrative; **not** a recorded production-webhook test in `TEST_LOG.md`):

```bash
curl -X POST "https://mbpicazo.app.n8n.cloud/webhook/nutrition-check" ^
  -H "Content-Type: application/json" ^
  -d "{\"barcode\":\"3017620422003\"}"
```

| Body | Workflow branch | Respond node (configured status) |
|------|-----------------|----------------------------------|
| `{}` | Empty body | `Respond - Usage Documentation` (200 JSON) |
| `{"foo":"bar"}` | Missing barcode | `Respond - Missing Barcode` (400 JSON) |
| `{"barcode":"ABC123"}` | Non-numeric | `Respond - Invalid Barcode` (400 JSON) |
| Unknown digits | OFF `status != 1` | `Respond - Product Not Found` (404 JSON) |
| Complete product | Full path | `Respond - Health Verdict` (200 `text/plain`) |
| Found, missing sugar/salt/fat | Incomplete nutrients | `Respond - Insufficient Nutrition Data` (200 JSON) |

Barcodes used in **manual MCP** executions: `3017620422003` (Nutella), `5449000000996` (Coca-Cola), `6111035000430` (Sidi Ali), `6111035002175` (incomplete nutriments), `0000000000000` (not found). See `TEST_LOG.md`.

## Error Handling

Status codes and `Content-Type` are **verified from Respond to Webhook node configuration**, not from a captured production HTTP response.

| Configured status | Configured Content-Type | Node | When |
|-------------------|-------------------------|------|------|
| 200 | `application/json` | `Respond - Usage Documentation` | Empty body |
| 400 | `application/json` | `Respond - Missing Barcode` | No barcode |
| 400 | `application/json` | `Respond - Invalid Barcode` | Non-digits |
| 404 | `application/json` | `Respond - Product Not Found` | OFF `status != 1` |
| 200 | `application/json` | `Respond - Insufficient Nutrition Data` | Missing sugar/salt/fat |
| 200 | `text/plain; charset=utf-8` | `Respond - Health Verdict` | Full path |

n8n platform (not a workflow Respond node): GET on the production URL returns JSON that the webhook is not registered for GET. That is runtime platform behavior. It is not a numbered TC and is not a production POST test.

There is **no** Error Trigger node and **no** `settings.errorWorkflow`. Groq/credential failures abort before `Respond - Health Verdict` (manual executions `9`, `10`, `12`).

## Limitations

- Groq `maxTokensToSample` is 200. On `openai/gpt-oss-20b`, visible copy was empty (manual exec `13`, `14`) or truncated (manual exec `15`) with `finish_reason: length`. Stage 3 classification still ran.
- Open Food Facts is live data; products can change.
- Energy is optional; missing energy is formatted as `N/A`.
- Unknown Nutri-Score does not invent a grade; overall uses traffic lights.
- `concern_score` does not set `overall_verdict` by itself (TC-006, manual exec `14`).
- Not a clinical nutrition tool.
- Webhook is POST-only by node configuration.
- TC-001–008 in `TEST_LOG.md` are manual MCP executions. Production webhook POST was not executed in that log.
