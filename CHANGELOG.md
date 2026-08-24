# Changelog

All notable changes to **SnackCheck - Nutrition Check** (`k6eh14EFRNlU5GUv`).

Versioning follows [Semantic Versioning](https://semver.org/).

## [1.0.0] - 2026-08-23

First published version of the inspected workflow (`active: true`, version `ea851cae-35fa-496e-91aa-2a9af2566a16`).

### Added

- POST webhook path `nutrition-check` with Respond-to-Webhook replies.
- Empty-body usage JSON on `Respond - Usage Documentation` (configured **200** `application/json`).
- Barcode required and digits-only validation on `Respond - Missing Barcode` / `Respond - Invalid Barcode` (configured **400** JSON `validation_error`).
- Open Food Facts v2 product lookup by barcode; `Respond - Product Not Found` when `status != 1` (configured **404** `product_not_found`).
- Nutrition extract from `sugars_100g` / `salt_100g` / `fat_100g` and `nutriments['energy-kcal_100g']`.
- `Respond - Insufficient Nutrition Data` when sugar, salt, or fat is missing (configured **200** JSON `status: insufficient_data`; Groq skipped).
- Deterministic traffic lights: sugar `<5` / `<=22.5` / `>22.5`; salt high `>1.5`; fat high `>17.5`.
- `concern_score` in Code node `Calculate - Concern Score` (`runOnceForEachItem`, high-flag count only).
- Overall verdict = worse of traffic-light and Nutri-Score; unknown Nutri-Score uses traffic-light only.
- Dual AI tone prepare nodes, Merge append, `Select - Active AI Tone` keeps `selected === true`.
- Groq model `openai/gpt-oss-20b` (`maxTokensToSample` 200, temperature 0.2) for copy only; `Respond - Health Verdict` configured **200** `text/plain; charset=utf-8`.

Configured HTTP status and Content-Type above are from Respond to Webhook node settings. They were not captured from a production HTTP request. TC-001–008 were manual MCP executions.

### Security

- Groq API key lives in n8n credential `Groq account`. It is not stored in these documentation files.

### Known issues

- `maxTokensToSample` 200 can yield empty or truncated visible Groq text on `openai/gpt-oss-20b` (manual exec `13`–`15`).
- Workflow description text still says “Inactive” while the workflow is published.

[1.0.0]: https://mbpicazo.app.n8n.cloud/workflow/k6eh14EFRNlU5GUv
