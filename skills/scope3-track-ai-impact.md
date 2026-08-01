---
name: Track AI inference environmental impact
description: Record and retrieve the energy, carbon (gCO2e), and water (mLH2O) impact of AI inference using the Scope3 AI Impact API and the scope3ai Python SDK.
api: openapi/scope3-ai-openapi-original.yml
base_url: https://aiapi.scope3.com
operations: [getImpact, calculateImpactBigQuery, listModels, getModel]
---

# Track AI inference environmental impact

Use the Scope3 AI Impact API to model the sustainability footprint of generative-AI usage.

## Auth
`Authorization: Bearer <JWT>` on every request (scheme `bearerAuth`, bearerFormat JWT).

## Steps
1. **List available models** — call `listModels` (GET `/model`) to see modeled AI models;
   `getModel` (GET `/model/{modelId}`) for a specific one.
2. **Compute impact for inference rows** — call `getImpact` (POST `/v1/impact`) with rows describing
   the model, task, and token/utilization counts. The response returns `total_energy_wh`,
   `total_gco2e`, and `total_mlh2o` per row.
3. **Batch/warehouse scoring** — call `calculateImpactBigQuery` for BigQuery-backed batch impact runs.

## SDK shortcut
`pip install scope3ai` then `Scope3AI.init(api_key=...)`; wrap calls in `with scope3.trace() as tracer:`
and read `tracer.impact()`. See `packages/scope3-packages.yml`.

## Errors
`400/401/403/404/406/409/415/429/500` return the `Error` schema. See `errors/scope3-problem-types.yml`.
