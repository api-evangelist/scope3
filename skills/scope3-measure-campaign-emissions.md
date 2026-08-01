---
name: Measure advertising campaign emissions
description: Calculate the carbon footprint of advertising media and creatives with the Scope3 Carbon Calculator API, and benchmark against country/channel percentiles.
api: openapi/scope3-measurement-openapi.yml
base_url: https://api.scope3.com/v2
operations: [measure, creative, benchmarks]
---

# Measure advertising campaign emissions

Use the Scope3 measurement API to compute greenhouse-gas emissions for advertising media.

## Auth
Send `Authorization: Bearer scope3_<accessClientId>_<accessClientSecret>` on every request
(see `authentication/scope3-authentication.yml`). Access is granted per customer by a Scope3 rep.

## Steps
1. **Measure media emissions** — call `measure` (POST `/measure`) with your delivery rows
   (inventory, country, channel, impressions/plays). Use `?includeRows=true` to get per-row
   breakdowns. Batch 4,000-8,000 rows per request; do not exceed ~100,000 rows.
2. **Measure creative emissions** — call `creative` (POST `/creative`) for the creative-distribution
   footprint of a specific asset.
3. **Benchmark** — call `benchmarks` (GET `/benchmarks`) to compare against country/channel
   emission percentiles.

## Conventions & errors
- Responses return an emissions breakdown plus a `RequestId`. See `conventions/scope3-conventions.yml`.
- On `400`/`403` the body is a plain `Error` object (not RFC 9457). See `errors/scope3-problem-types.yml`.
- No idempotency key is supported; retries are not deduplicated.
