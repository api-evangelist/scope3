---
name: Scope3
description: Use when building buyer agents, storefront integrations, or campaign management workflows. Agents use this skill to discover products, create campaigns, execute media buys, manage inventory sources, and transact through the AdCP protocol.
metadata:
    mintlify-proj: scope3
    version: "1.0"
---

# Scope3 Interchange Skill

## Product Summary

Scope3 Interchange is an agentic advertising platform where buyer agents and seller storefronts transact on the open AdCP (Ad Context Protocol). It's built from the ground up for AI agents as primary callers, not humans. Agents integrate once to reach every storefront; storefronts register once to reach every buyer agent.

**Key files and endpoints:**
- **Buyer REST API:** `https://api.interchange.io/api/v2/buyer`
- **Storefront REST API:** `https://api.interchange.io/api/v2/storefront`
- **Buyer MCP:** `https://api.interchange.io/mcp/buyer`
- **Storefront MCP:** `https://api.interchange.io/mcp/storefront`
- **Skill files:** `https://api.interchange.io/api/v2/buyer/skill.md` and `/storefront/skill.md`
- **OpenAPI specs:** `https://api.interchange.io/api/v2/buyer/openapi.yaml` and `/storefront/openapi.yaml`

Authentication uses OAuth (for MCP connectors) or API keys (`Authorization: Bearer scope3_...`). All responses follow a standard envelope: `{ "data": <result>, "error": null }` on success, `{ "data": null, "error": {...} }` on failure.

## When to Use

**Buyer agents:** Create advertisers, plan campaigns, discover products from storefronts, execute media buys, manage creatives, track delivery, and report performance.

**Storefront operators:** Register inventory sources, create products, configure buyer routing, approve media buys, manage signals, and track analytics.

**Specific triggers:**
- User asks to "create a campaign" or "launch a media buy" → buyer campaign workflow
- User asks to "discover products" or "find inventory" → product discovery workflow
- User asks to "set up a storefront" or "add inventory" → storefront inventory workflow
- User asks to "check campaign status" or "get delivery metrics" → reporting workflow
- User asks to "approve a media buy" or "review creatives" → storefront approval workflow

## Quick Reference

### Buyer Core Objects

| Object | Purpose | Key Fields |
|--------|---------|-----------|
| **Advertiser** | Top-level container for a brand/agency | `id`, `name`, `domain`, `status` |
| **Campaign** | Media plan with flight dates and budget | `id`, `advertiserId`, `flightDates`, `budget`, `status` (DRAFT → ACTIVE → COMPLETED) |
| **Media Buy** | One ADCP transaction with one sales agent | `id`, `campaignId`, `salesAgentId`, `status` |
| **Package** | One product per pacing period per media buy | `id`, `mediaBuyId`, `productId`, `budget` |
| **Creative** | Ad asset (image, video, etc.) | `id`, `campaignId`, `format`, `manifest` |
| **Discovery Session** | Browse and select products from storefronts | `id`, `advertiserId`, `products[]`, `status` |

### Storefront Core Objects

| Object | Purpose | Key Fields |
|--------|---------|-----------|
| **Storefront** | Publisher's buyer-facing presence | `id`, `platformId`, `name`, `status` |
| **Inventory Source** | AdCP agent providing products | `id`, `storefrontId`, `type`, `status` |
| **Product** | Sellable inventory unit | `id`, `sourceId`, `name`, `pricing`, `targeting` |
| **Signal** | Audience/contextual targeting dimension | `id`, `sourceId`, `name`, `type` |
| **Media Buy Approval** | Pending buyer order for review | `id`, `mediaBuyId`, `status` (PENDING → APPROVED/REJECTED) |

### Authentication

| Method | When to Use | Setup |
|--------|-----------|-------|
| **OAuth** | Claude/ChatGPT MCP connectors | Automatic via login flow; no keys to manage |
| **API Key** | CLI, Cursor, custom agents, REST calls | Generate at `interchange.io/user-api-keys`; use `Authorization: Bearer scope3_...` |
| **M2M** | Deployed backend services | Create at `/api/v2/m2m-applications`; exchange client credentials for JWT tokens |

### MCP Tools (All Roles)

| Tool | Purpose |
|------|---------|
| `health` | Check API connectivity |
| `ask_about_capability` | Query skill.md to discover endpoints and schemas |
| `api_call` | Execute any REST endpoint with full request/response |
| `ask_murph` | Ask Murph (AI guide) for help, guidance, or to file support reports |

### Common Buyer Workflows

**Create and execute a campaign:**
1. `POST /advertisers` — create advertiser
2. `POST /campaigns` — create campaign in DRAFT status with flight dates, budget
3. `POST /discovery/discover-products` — search storefronts for products
4. `POST /discovery/:id/add-products` — select products for the campaign
5. `POST /campaigns/:id/creatives/create` — upload creatives (multipart)
6. `POST /campaigns/:id/execute` — launch campaign → creates media buys, submits to ADCP

**Pause and resume:**
- `POST /campaigns/:id/pause` — halt all media buys
- `POST /campaigns/:id/reactivate` — resume a paused campaign

**Check status:**
- `GET /campaigns/:id` — read campaign and media buy status
- `GET /campaigns/:id/delivery` — read delivery metrics

### Common Storefront Workflows

**Set up inventory:**
1. `POST /inventory-sources` — register an AdCP agent or ad-server connection
2. `POST /inventory-sources/:id/test-connection` — verify connectivity
3. `POST /inventory-sources/:id/products` — create products
4. `POST /inventory-sources/:id/signals` — define targeting dimensions

**Approve media buys:**
1. `GET /media-buy-approvals` — list pending approvals
2. `POST /media-buy-approvals/:id/decide` — approve or reject with reason

**Configure buyer routing:**
- `POST /buyer-routing/mappings` — map buyer domain to GAM advertiser
- `GET /buyer-routing/mappings` — list active mappings

## Decision Guidance

### When to Use Discovery vs. Direct Product Query

| Scenario | Use Discovery | Use Direct Query |
|----------|---------------|------------------|
| Buyer browsing storefronts for products | ✓ | |
| Buyer has specific product IDs to order | | ✓ |
| Storefront testing product availability | ✓ | |
| Programmatic product lookup by ID | | ✓ |

**Discovery** (`POST /discovery/discover-products`) returns grouped, browsable results with proposals. **Direct query** (`POST /products/query`) returns canonical products by ID or filter.

### When to Stage vs. Execute Media Buys

| Scenario | Stage First | Execute Directly |
|----------|------------|------------------|
| Need to review/customize media buys before submission | ✓ | |
| Simple one-call submission with no review | | ✓ |
| Changing pacing, creatives, or per-buy settings | ✓ | |
| Buyer wants to see generated buys before ADCP submission | ✓ | |

**Stage** (`POST /discovery/create-media-buys?mode=stage`) creates DRAFT media buys for review. **Execute** (`POST /discovery/create-media-buys?mode=execute` or `POST /campaigns/:id/execute`) submits directly to ADCP.

### When to Use Buyer Instructions vs. House Discounts

| Use Case | Buyer Instructions | House Discounts |
|----------|-------------------|-----------------|
| Apply rules to a specific buyer domain | ✓ | |
| Apply blanket discount to all buyers | | ✓ |
| Conditional pricing by buyer attributes | ✓ | |
| Simple percentage or fixed discount | | ✓ |

**Buyer instructions** are per-domain rules (pricing, targeting restrictions, creative requirements). **House discounts** are storefront-wide percentage/fixed discounts.

## Workflow

### Buyer Campaign Workflow

1. **Understand the request.** Parse the user's brief: budget, flight dates, target audience, channels, KPIs.

2. **Check existing state.** Call `GET /advertisers` to list existing advertisers; reuse if one matches, or create new with `POST /advertisers`.

3. **Create the campaign.** Call `POST /campaigns` with advertiser ID, flight dates, budget (gross, all-in), and optional brief/constraints. Campaign enters DRAFT status.

4. **Discover products.** Call `POST /discovery/discover-products` with the brief. Inspect results and call `POST /discovery/:id/add-products` to select products for the campaign.

5. **Upload creatives.** Call `GET /campaigns/:id/creatives/templates` to see required formats. Call `POST /campaigns/:id/creatives/create` (multipart) to upload manifest-based creatives. Verify `campaign.creativeFormats.missing` is empty.

6. **Stage or execute.** Call `POST /discovery/create-media-buys?mode=stage` to review DRAFT media buys, or `POST /campaigns/:id/execute` to submit directly. If staging, inspect returned media buys and call `POST /campaigns/:id/execute` when ready.

7. **Monitor delivery.** Poll `GET /campaigns/:id` for status and `GET /campaigns/:id/delivery` for metrics. Call `POST /campaigns/:id/pause` if needed.

### Storefront Inventory Workflow

1. **Understand the source.** Determine source type: ad-server (GAM, FreeWheel), modular (avails feed), or external agent (AdCP).

2. **Register the source.** Call `POST /inventory-sources` with source type and credentials/config. For ad-server, provide network code and service account email.

3. **Test connectivity.** Call `POST /inventory-sources/:id/test-connection` to verify the connection. Check `lastErrorCode` if it fails.

4. **Create products.** Call `POST /inventory-sources/:id/products` to define sellable units. Include pricing, targeting capabilities, and creative specs.

5. **Define signals.** Call `POST /inventory-sources/:id/signals` to expose audience/contextual dimensions buyers can target.

6. **Configure buyer routing.** Call `POST /buyer-routing/mappings` to map buyer domains to your ad-server advertisers. This ensures media buys route to the correct account.

7. **Approve media buys.** Poll `GET /media-buy-approvals` for pending orders. Call `POST /media-buy-approvals/:id/decide` to approve or reject with reason.

## Common Gotchas

- **Budget is always gross.** `campaign.budget.total` is the all-in amount the buyer pays, including Interchange fees. There is no separate fee model input at creation.

- **Media buys spawn at execute, not create.** Campaigns are DRAFT until you call execute. Only then does Interchange create one media buy per sales agent and submit to ADCP.

- **Creatives must match formats.** Check `campaign.creativeFormats.missing` after uploading. If it's not empty, the campaign is missing required creative formats and will fail on execute.

- **Discovery products are session-scoped.** Products selected in a discovery session are tied to that session. If you update the discovery query, call `create_media_buys` with `replace: true` to apply the new selection.

- **Pricing requires configuration.** If you get `PRICING_NOT_CONFIGURED`, the advertiser has no pricing rule for the selected sales agent. Configure pricing through the storefront or contact support.

- **Fee terms lock at media buy creation.** Once a media buy is created, its fee rate and budget breakdown are locked. You cannot change the fee terms on an existing buy; you must pause it and create a new one.

- **Storefront credentials are per-source.** If a storefront has multiple inventory sources and one requires OAuth, register credentials for that source only. Other sources do not need them.

- **Approval workflows are storefront-specific.** Media buy approval mode (auto-approve, manual review, per-buyer override) is configured per storefront. Check the storefront's approval routing before assuming a buy will auto-approve.

- **AdCP errors are in structuredContent.** MCP tool errors from downstream AdCP agents are returned in `structuredContent`, not HTTP status codes. Parse the ADCP error spec, not HTTP.

- **Validation errors enumerate all issues.** When you get `VALIDATION_ERROR`, `details.issues[]` lists every problem Zod found, with dotted field paths (e.g., `budget.amount`, `targeting.geos`). Fix all of them, not just the first.

- **Rate limits are per-org, not per-key.** All API keys for an organization share the same rate limit bucket. Hitting the limit returns `RATE_LIMITED` with a `Retry-After` header.

## Verification Checklist

Before submitting work:

- [ ] **Authentication works.** Test with `curl https://api.interchange.io/health` and `curl https://api.interchange.io/api/v2/accounts/current -H "Authorization: Bearer <key>"`.
- [ ] **Advertiser exists.** For buyer workflows, confirm the advertiser ID is valid with `GET /advertisers/:id`.
- [ ] **Campaign is in correct state.** Campaigns must be DRAFT before execute, ACTIVE before pause, PAUSED before reactivate.
- [ ] **Creatives cover all formats.** Check `campaign.creativeFormats.missing` is empty before executing.
- [ ] **Budget is sufficient.** Ensure `campaign.budget.total` covers all media buys. Check for `INSUFFICIENT_MEDIA_BUDGET` errors.
- [ ] **Products are selected.** For discovery workflows, confirm products were added with `GET /discovery/:id/products`.
- [ ] **Storefront is ready.** For storefront workflows, call `GET /storefront/readiness` to check for blockers.
- [ ] **Inventory source is connected.** For storefront setup, verify `GET /inventory-sources/:id` shows `status: ACTIVE`.
- [ ] **Error codes are actionable.** If an error occurs, check the `error.code` (not just the message) and handle it appropriately (retry on `RATE_LIMITED`, fix input on `VALIDATION_ERROR`, etc.).
- [ ] **Responses are unwrapped.** Remember that all responses are wrapped in `{ "data": <result>, "error": null }`. Read from `response.data`.

## Resources

**Comprehensive navigation:** Read the [llms.txt file](https://docs.interchange.io/llms.txt) for a page-by-page index of all documentation.

**Critical pages:**
- [Introduction](https://docs.interchange.io/v2/introduction) — What Interchange is and how it works
- [Built for Agents](https://docs.interchange.io/v2/setup/built-for-agents) — How to connect Claude, ChatGPT, Cursor, and custom agents
- [Buyer Onboarding](https://docs.interchange.io/v2/setup/buyer-onboarding) — End-to-end campaign-launch flow for buyer integrations

---

> For additional documentation and navigation, see: https://docs.interchange.io/llms.txt