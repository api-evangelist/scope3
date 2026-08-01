---
name: scope3-agentic-storefront
version: "2.0.0"
description: Scope3 Agentic Storefront API - Publisher and seller integrations
api_base_url: https://api.interchange.io/api/v2/storefront
auth:
  type: bearer
  header: Authorization
  format: "Bearer {token}"
  obtain_url: https://interchange.io/user-api-keys
---

# Scope3 Agentic Storefront API

This API enables partners to manage their storefronts and register agents through the Scope3 Agentic platform. A storefront is automatically created for each seller — partners configure and activate it through this API. Each seller has exactly one storefront. Agents are created as part of inventory source setup.

**Important**: Use the most specific MCP tool first. Session-level actions such as customer switching use dedicated tools like `customer_switch`; do not route them through `api_call`. For storefront REST operations, use `api_call` with the `operation` field — every call must supply a named operation; there is no raw `method` + `endpoint` mode.

## ⚠️ CRITICAL: Never Render Your Own UI When a Tool Is Already Returning One

**When a tool response already includes a UI component, display it as-is. Do NOT generate your own HTML artifacts, dashboards, charts, or visual components on top of it.** If the tool returns a UI, that is the UI — do not create a competing or duplicate visualization.

**Every value you present must trace to a tool response or user input.** When you show the user a storefront status, `platformId`, `sourceId`, `agentId`, readiness state, or any other data point, it must come from a prior `api_call` response, an `ask_about_capability` response, or something the user told you in this conversation. If you're unsure whether a value is still current (for example, storefront status may change after activation), qualify it — "last known status from `get_storefront`" — rather than stating it as a fresh fact.

## ⚠️ CRITICAL: Never Fabricate User Data

**Before making any API call that creates or modifies a resource, you MUST have explicit user input for all required fields.** Do NOT invent, guess, or auto-fill values the user hasn't provided.

**Rules:**
- **NEVER fabricate values** for required fields. If the user hasn't provided a value, ask for it.
- **Read-only calls are fine** — you can freely call GET endpoints (`get_storefront`, `readiness`, etc.) to fetch state and present it to the user.
- **Confirm before mutating** — Before any POST, PUT, or DELETE, verify you have user-provided (or user-confirmed) values for every required field.
- **Never make up IDs** — `platformId`, `agentId`, and similar identifiers must come from previous API responses. `sourceId` is user-chosen at creation time but must be echoed exactly on subsequent calls. `domain` values must come from the user.
- **Discovered agents are the source of truth for agent registration** — when adding an inventory source based on discovery, the `endpointUrl`, `protocol`, and agent metadata MUST come from the `discover-agents` response. Do NOT invent agent URLs or invent agents that did not appear in discovery results.
- **Only use what's documented** — Do NOT invent endpoints, fields, query parameters, or enum values that are not explicitly listed in this skill document (or returned by `ask_about_capability`). If you're unsure whether something exists, check first. If it's not here, don't use it.

## Before Every api_call: Verify Each Field Has a Source

Before invoking `api_call`, mentally walk through every field in `body`, `pathParams`, and `params` and name its source:

- "`platformId`: from the `get_storefront` response above" ✅
- "`domain`: user said acme.com" ✅
- "`sourceId`: user chose `acme-sales-agent`" ✅
- "`endpointUrl`: from the `discover-agents` response" ✅
- "`publisherDomain`: I assumed it's the same as operatorDomain" ❌ — STOP, ask the user
- "`status`: I think activation uses `ACTIVE`" ❌ — STOP, call `ask_about_capability`. Storefronts no longer have a writable `status` — toggle `transacting: true | false` on `PUT /storefront`, or call `POST /storefront/archive` to archive. The response carries a read-only `displayStatus` of `configuring | transacting | archived`.

If any field's source is "I assumed…", "it's probably…", "typically these are…", or "the docs usually say…" — do NOT send the call. Either ask the user for the value, or call `ask_about_capability` to find the correct schema.

## Notifications and readiness

When the storefront is not live yet, `ask_about_capability` prepends an "Open readiness blockers" section listing what is preventing buyers from transacting (for example, products that cannot be trafficked because a video product is missing its duration). When that section is present, address those blockers first — name them and explain how to fix them — before answering the rest of the user's question.

Notifications can be listed, marked as read, or acknowledged via the `/api/v2/notifications` endpoints — see the Notifications section below for details. Go-live blockers like un-traffickable products also raise an operator notification, so they appear in the notification feed and (for opted-in operators) by email.

Readiness has a compliance companion: `get_storefront_compliance` (`GET /api/v2/storefront/readiness/compliance`) returns the most recent compliance readiness check for the storefront, or `null` when no compliance check has run yet. Use it when an operator asks specifically about compliance status rather than the full go-live readiness picture (`GET /api/v2/storefront/readiness`).

## Getting Started

Storefront REST operations go through the generic `api_call` tool. Base path: `/api/v2/storefront/`. Session-level actions such as customer switching use dedicated MCP tools instead. All endpoints require authentication via the MCP session, except the platform-issued OAuth callback URL (`GET /api/v2/storefront/oauth/callback`), which is a token-gated callback URL the platform mints.

### `api_call` parameters

**Every call must supply the `operation` field** — each endpoint maps to a named operation (e.g. `list_partner_agents`, `get_storefront`, `update_storefront`, `create_esa_connection`, `respond_to_demand_signal`). Using the operation enum derives the HTTP method and endpoint automatically and prevents hallucinated endpoints, wrong methods, and misplaced path params. There is no raw `method` + `endpoint` mode.

- `operation` (string): Named operation to execute (required). `method` and `endpoint` are derived automatically.
- `pathParams` (object): Path parameters for the operation (e.g. `{ "esaId": "abc-123" }`). Required when the operation uses path params.
- `body` (object): Request body for POST/PUT/PATCH operations.
- `params` (object): Query parameters.

Call `ask_about_capability` first to learn the right operation name and field shapes for the user's request.

## When to call Murph (`ask_murph`)

Murph is your AI guide to Interchange. It's mounted on this server so you don't have to switch MCP servers when a publisher pivots from "do X" to "explain X."

**Route to Murph when the user asks**:
- "How do I…" / "Where do I start with onboarding" / "What's the right way to…"
- "Why is my X in state Y?" — storefront stuck in configuring, sync run failing, agent not registering, payout details not saving
- "What changed recently?"
- "This looks broken" / "I think there's a bug" — Murph files a report for the Scope3 team
- General help where the publisher doesn't know which endpoint applies

**Stay on `api_call` when the user asks for**:
- A specific CRUD action (`update storefront`, `sync inventory source`, `approve incoming demand signal`, etc.)
- A specific data fetch (`list my inventory sources`, `show me storefront reporting`)
- Any mutation — Murph is read-only

**Don't double-route.** If the user's intent is unambiguously mutate-this, call `api_call` directly. Don't ask Murph first.

**Murph parameters:**
- `prompt` (string, required): The user's question, verbatim. Plain English.
- `conversationUid` (string, optional): Pass back what Murph returned on the previous turn to continue a thread. Omit on first call.

**Murph caveats** (foundation mode):
- Murph answers from its own knowledge of Interchange. Live-state inspection and full docs citations are follow-up work.
- Murph CAN record an internal report (`escalated: true`). Do NOT mention any ticket ID or internal tracking reference — tell the user the Scope3 team has been notified.
- Murph is scoped to the caller's customer. One Murph conversation == one customer.

### Seller property, coverage, and key-value targeting docs

For seller questions about publisher properties, disclosed coverage, network
`adagents.json`, GAM custom targeting keys/values for site lists, or buyer
property-list briefs, route to Murph or retrieve the public docs. Use these
pages as pointers only; do not answer those topics from this skill:

- `storefront/inventory-sources/publisher-properties-coverage`
- `storefront/inventory-sources/adagents-json`
- `storefront/esa/custom-targeting-properties`
- `storefront/property-list-briefs`
- `guides/property-lists`

### Onboarding Flow

When a new seller is created on the platform, a storefront is **automatically created** with `transacting = false` (`displayStatus = "configuring"`). Each seller has exactly one storefront — there is no need to create one manually. The storefront stays in `configuring` until the operator finishes setup and explicitly flips `transacting = true`.

The onboarding flow is about **configuring** the auto-created storefront. The canonical AAO-driven setup:

```
1.  Get Storefront            → GET  /storefront                                                      → storefront (transacting=false, displayStatus=configuring)
2.  Resolve Brand (AAO)       → POST /storefront/resolve-brand     { domain }                        → { brandName, logoUrl, manifestUrl }
3.  Configure Storefront      → PUT  /storefront                   { name, publisherDomain,
                                                                      operatorDomain, brandName,
                                                                      logoUrl }                      → operatorDomainVerified server-computed
                                                                                                        from domain match + approval proof
4.  Discover Agents (AAO)     → GET  /storefront/discover-agents?domain={operatorDomain}             → { operator.agents[], publisher.agents[] }
5.  Add Inventory Source      → POST /storefront/inventory-sources (one per selected agent)          → source + agent (status=PENDING)
6.  Configure Auth            → PUT  /storefront/inventory-sources/{sourceId} { auth: ... }          → saves credentials; agent auto-activates
7.  Set Payout Details        → PUT /storefront/billing/payout-details { beneficiaryName,          → onboardingStatus flips to
       (OPTIONAL by default,         addressLine1, city, region, postalCode, countryCode,               "complete" once details
        REQUIRED if any agent         accountNumber, bankIdentifierType,                                 are on file
        supports `agent` billing)     bankIdentifierValue, currency }
                                      Optional when no connected agent advertises `agent` billing in
                                      its capabilities (`accounts.supported_billing`). REQUIRED once
                                      any connected agent declares `agent` billing — buyers can ask
                                      Scope3 to clear payments on those buys, and Scope3 finance
                                      needs the seller's payout bank details to remit by bank
                                      transfer. Without payout details on an optional storefront the
                                      storefront is limited to external agreements with buyers —
                                      Scope3 will not clear payments and the seller must bill the
                                      buyer directly. Account-admin only.
                                  After saving, read the billing config with GET /storefront/billing
                                  to confirm onboardingStatus and view the masked payoutDetails.
8.  Confirm Settlement        → PUT  /storefront                   { defaultCurrency: "USD" }        → required before go-live for
       Currency                                                                                         seller-settled storefronts;
       (seller-settled                                                                                  ISO-4217; never defaulted
        storefronts only)                                                                               silently. Direct sales adapter
                                                                                                        storefronts run by our expert
                                                                                                        agents skip this.
9.  Check Readiness           → GET  /storefront/readiness                                           → must be "ready"
10. Open for transactions     → PUT  /storefront                   { transacting: true }             → rejected if readiness is blocked
```

**Example walkthrough:**

**Step 1 — Retrieve the auto-created storefront:**
```http
GET /api/v2/storefront
```
Response: `{ "platformId": "acme-media", "name": "Acme Media", "transacting": false, "archivedAt": null, "displayStatus": "configuring", "plan": "basic", ... }`

**Step 2 — Resolve the brand from the AAO registry:**
```http
POST /api/v2/storefront/resolve-brand
{ "domain": "acme.com" }
```
Response (resolved): `{ "resolved": true, "domain": "acme.com", "brandName": "Acme", "logoUrl": "https://acme.com/logo.svg", "manifestUrl": "https://acme.com/.well-known/brand.json" }`
Response (unresolvable): `{ "resolved": false, "domain": "acme.com", "error": "No brand profile found", "builderUrl": "https://agenticadvertising.org/brand" }`

**Step 3 — Configure storefront details with brand info:**
```http
PUT /api/v2/storefront
{
  "name": "Acme Media Network",
  "publisherDomain": "acme.com",
  "operatorDomain": "acme.com",
  "brandName": "Acme",
  "logoUrl": "https://acme.com/logo.svg"
}
```
Response: `{ "platformId": "acme-media", "name": "Acme Media Network", "publisherDomain": "acme.com", "operatorDomain": "acme.com", "brandName": "Acme", "logoUrl": "https://acme.com/logo.svg", "operatorDomainVerified": true, "transacting": false, "archivedAt": null, "displayStatus": "configuring", ... }`

`operatorDomainVerified` is server-computed. It becomes `true` only when the supplied `operatorDomain` matches the authenticated customer's registered `customerDomain` and that `customerDomain` is approved by active-member email ownership or Scope3 admin attestation. Otherwise it remains `false` until manual KYC/admin verification.

**Step 4 — Discover agents registered for this operator in the AAO:**
```http
GET /api/v2/storefront/discover-agents?domain=acme.com
```
Response: `{ "domain": "acme.com", "operator": { "agents": [ { "url": "...", "name": "...", "type": "SALES", "compliance": {...}, "storyboards": [...] } ] }, "publisher": { "agents": [...] } }`

Pass `?refresh=true` to bypass the in-memory cache.

**Step 5 — Add an inventory source for each agent you want to sell through:**
```http
POST /api/v2/storefront/inventory-sources
{
  "sourceId": "acme-sales-agent",
  "name": "Acme Sales Agent",
  "executionType": "AGENT",
  "type": "SALES",
  "endpointUrl": "https://api.acme.com/adcp",
  "protocol": "MCP",
  "authenticationType": "API_KEY",
  "auth": { "type": "bearer", "token": "my-api-key" }
}
```
Response: `{ "sourceId": "acme-sales-agent", "agentId": "acmesalesagent_abc123", "status": "PENDING", ... }`

**Step 6 — Saving auth credentials auto-activates the agent:**
```http
PUT /api/v2/storefront/inventory-sources/acme-sales-agent
{ "auth": { "type": "bearer", "token": "my-api-key" } }
```
When an agent is `PENDING` and has no previous credentials, saving `auth` flips its status to `ACTIVE`. A separate `{ "status": "ACTIVE" }` call is no longer required.

**Step 7 — Set payout bank details (CONDITIONAL):**
```http
PUT /api/v2/storefront/billing/payout-details
{
  "beneficiaryName": "Acme Media Network Ltd",
  "addressLine1": "500 Market Street",
  "addressLine2": "Suite 400",
  "city": "San Francisco",
  "region": "CA",
  "postalCode": "94105",
  "countryCode": "US",
  "accountNumber": "12345678901234",
  "bankIdentifierType": "FEDWIRE_ABA",
  "bankIdentifierValue": "021000021",
  "currency": "USD"
}
```

Payout details are **optional** by default, and **required** once any connected agent advertises `agent` billing in `accounts.supported_billing` (buyers can ask Scope3 to clear payments on those buys, so Scope3 finance needs the seller's payout bank details to remit by bank transfer). Without payout details on an optional storefront, the storefront is limited to external agreements with buyers — Scope3 will not clear payments and the seller must bill the buyer directly. This call is **account-admin only**.

`accountNumber` is 6–34 alphanumeric characters (a bank account number or IBAN) and is **write-only** — it is never returned. It is required the first time; omit it on later edits to keep the stored value. `bankIdentifierType` is one of `FEDWIRE_ABA`, `CHIPS_ABA`, `SWIFT_BIC`, or `BANK_CODE`. `currency` is the seller's payout currency (ISO-4217).

Payouts are made by bank transfer in the seller's payout currency, run manually by Scope3 finance — there is no payment processor. After saving, read the billing config with `GET /storefront/billing` to confirm `onboardingStatus` (`complete` once payout details are on file) and view the masked `payoutDetails`.

**Multiple payout entities:** a seller org with more than one legal payout entity (e.g. an India entity and a Brazil entity) registers one bank account per entity per payout currency via `PUT /api/v2/storefront/billing/payees` (operation `set_payout_payee`). The body is the same as payout details plus `entityName` — the legal entity label; `beneficiaryName` must match the account holder, which is the legal entity itself. Upserts are keyed by `entityName` + `currency`: one bank account per entity per currency. List the payees (masked to last 4) with `GET /api/v2/storefront/billing/payees` (operation `list_payout_payees`) — each payee's `isPrimary` flag marks whether it's the org's PRIMARY payout entity.

The payout details set via Step 7 (`PUT /billing/payout-details`) ARE the org's PRIMARY payee — a payee row like any other, just always `isPrimary: true`. Editing via Step 7 always edits THE primary row in place (renaming the entity or changing its currency never creates a second row). The primary entity can hold multiple currencies too: register additional currencies for it through `PUT /billing/payees` with the same `entityName`/`beneficiaryName` as the primary entity, optionally passing `isPrimary: true` to (re-)designate which payee is primary. Exactly one payee per customer is primary — a customer's FIRST payee automatically becomes the primary, and promoting another demotes whichever payee held it before. Payee removal is finance-mediated — direct the seller to Scope3 support. Same account-admin rule as Step 7.

**Step 8 — Confirm the settlement currency (REQUIRED before go-live for seller-settled storefronts):**
```http
PUT /api/v2/storefront
{ "defaultCurrency": "USD" }
```
Sets the storefront's settlement currency (`storefront_config.default_currency`) and clears the `currency_confirmed` readiness blocker for seller-settled storefronts. The value is an ISO-4217 code (`^[A-Z]{3}$`, e.g. `"USD"`, `"EUR"`) and is **never defaulted silently** — the seller must confirm it. This is the same `PUT /storefront` endpoint used for the rest of storefront config; pass `defaultCurrency` alongside or after the other fields.

This is **distinct from payout details** — saving payout bank details does NOT set the settlement currency, and a seller-settled storefront stays blocked on `currency_confirmed` until you send `defaultCurrency` here. If you don't know which currency the seller settles in, ask them; do not guess. There is no separate currency endpoint, so on a blocked seller-settled storefront this `PUT` is how the currency gets confirmed.

Exception: direct sales adapter storefronts run by our expert agents (`routingMode: "ADAPTER"`, `adapterSourceKind: "sales"`, e.g. Meta/Google/TikTok/Snap/Pinterest/Reddit/Spotify/Amazon) do not have a seller payout currency in Interchange. The buyer pays the downstream platform directly through their selected platform account, so readiness skips `currency_confirmed` and `settlement_currency_match` for those storefronts. Do not nudge a seller to confirm a settlement currency for those adapter storefronts; use the platform account currency only where adapter budget/reporting calls need it.

**Multiple settlement currencies are supported for seller-settled storefronts.** `defaultCurrency` is the primary settlement currency and clears the required readiness blocker for those storefronts. If the seller wants to be paid directly in more currencies, also set `paymentCurrencies` to the full payout set, including the primary currency: `PUT /storefront { "defaultCurrency": "USD", "paymentCurrencies": ["USD", "EUR"] }`. If `paymentCurrencies` is omitted or empty, the storefront keeps single-currency behavior and falls back to `defaultCurrency`. The seller decides this direct-settlement set; the marketplace may still accept buyer currencies outside it via FX and convert them to one of the storefront's settlement currencies.

**Step 9 — Confirm readiness:**
```http
GET /api/v2/storefront/readiness
```
Response: `{ "status": "ready", "checks": [...] }`

**Step 10 — Activate the storefront:**
```http
PUT /api/v2/storefront
{ "transacting": true }
```

### Testing Discovery Before Launch

Sellers can test how buyer agents respond to a brief before the storefront goes live. Ask Murph to run a discovery preview: Murph will call the storefront's agents with the given brief and return what products buyers would see — even when the storefront is still in `configuring` status. This works for all storefront types including composition, passthrough, and adapter storefronts.

---

## Account Management

Some account management tasks are handled in the web UI at [interchange.io](https://interchange.io). Direct users to these pages for:

| Task | URL | Capabilities |
|------|-----|--------------|
| **API Access** | [interchange.io/user-api-keys](https://interchange.io/user-api-keys) | Organization admins manage WorkOS API keys; secrets appear only once at creation |
| **Team Members** | [interchange.io/user-management](https://interchange.io/user-management) | Invite members, manage roles, manage partner access |
| **Billing** | Available from user menu in the UI | Set payout bank details, view billing configuration and invoices |
| **Profile** | [interchange.io/user-info](https://interchange.io/user-info) | View and update user profile |

**Note:** Billing and member management require admin permissions.

Machine credentials require an organization admin and a human handoff. Organization
API keys are managed in **Settings → API Access**. M2M applications do not have a
management widget yet; a human admin or their developer uses the documented
`/api/v2/m2m-applications` lifecycle with an interactive human session. Never create,
retrieve, reveal, echo, copy, or transport an API key or M2M client secret in an MCP
conversation. Existing secrets cannot be retrieved. The admin must copy the one-time
secret directly into the workload's secret manager; if it is lost, replace it.
Prefer M2M `client_credentials` and short-lived access tokens for unattended
enterprise backends. Use a least-privilege organization API key for simpler scripts
and automation.

### Switch Account

Switching the active customer context is a session-mutating operation and is not available through `api_call`. Use the dedicated `customer_switch` MCP tool instead:

```json
{
  "tool": "customer_switch",
  "arguments": { "customerId": 123 }
}
```

Any user can switch to a customer they have an active membership on. SuperAdmin users can additionally switch to any customer. After the switch, all subsequent operations in the session are performed on behalf of the selected customer.

### Browser Origins for Custom Agents

Account admins can self-serve exact browser origins for browser-based MCP/OAuth integrations. Teach this as a two-part setup:

1. **OAuth redirect URI**: the full callback URL, including path, for example `https://mcp.example.com/oauth/callback`. Register it with dynamic client registration at `/auth/register`; the response returns the public `client_id` to use with OAuth/PKCE.
2. **Browser origin**: the exact origin only, for example `https://mcp.example.com`. Register it with the browser-origin operations below so browser CORS is allowed for MCP/OAuth calls.

Do not confuse these values. Redirect URIs include a path and must match exactly during OAuth. Browser origins are only `scheme://host[:port]` and never include a path, query, fragment, or wildcard.

```json
{ "operation": "list_browser_origins" }
```

```json
{
  "operation": "create_browser_origin",
  "body": {
    "origin": "https://mcp.example.com",
    "label": "Example Agent"
  }
}
```

Use `delete_browser_origin` with `pathParams: { "id": "<origin-id>" }` to remove one.

Register only exact HTTPS origins (`scheme://host[:port]`). Each production, staging, and development browser host needs its own origin. Do not register redirect URLs here.

Security posture:
- Customer-managed origins are exact-match and HTTPS-only.
- They enable CORS only for `/authorize`, `/auth/register`, `/auth/token`, `/auth/mcp/authorize`, `/.well-known/*`, `/mcp`, and `/mcp/*`; they do not open arbitrary REST API paths to that origin.
- Dynamic redirect registration is public, but it does not by itself authorize a client for a customer. For self-serve clients, Interchange issues an authorization code only when the redirect origin is also approved for the authenticated customer's account.
- They do not receive credentialed cookie CORS. Browser agents should use OAuth access tokens or explicit bearer tokens.
- Do not tell users that registering an origin shares their Interchange login cookie with that site.
- If a browser request is blocked, check: full redirect URI registered through `/auth/register`; exact origin registered by an admin; request goes to a supported browser MCP/OAuth path rather than a REST endpoint; browser fetch does not use `credentials: "include"`.

### Plan & billing (consolidated commercial account)

Named operation: `get_billing_account` (no path params — scoped to the caller's organization by auth context).

```http
GET /api/v2/billing/account
```

```json
{ "operation": "get_billing_account" }
```

Returns `{ "data": { ... } }` — one consolidated commercial-account document: plan and contract status, ToS acceptance, effective pricing, intelligence usage, credit/prepay standing and balance, agreements, child accounts, and the single next action (if any) needed to become or remain paid. Billing lives at the org level regardless of role, so this is the same document buyers see. Org-admin only; a child-account request resolves to the parent organization. This is the same read model as the `get_billing_account` hand tool (which additionally accepts a `section` argument to return just one section); the operation exists so the Plan & Billing MCP-app widget can hydrate through `api_call` on external MCP-UI hosts.

### Organization IU Rate Card and plan selection

When an administrator asks to see, choose, or inspect the organization's IU
plan, call `get_iu_rate_card_offer`. Do not route this task through `api_call`
or construct a substitute picker. The same typed Task and accepted binding
govern buyer, storefront, and Murph usage; workload attribution stays separate.

```json
{
  "tool": "get_iu_rate_card_offer",
  "arguments": {}
}
```

The response is the source of truth for:

- the legacy `commercialStatus` wire value, `eligibleWorkloads`, one
  `currentBinding`, and immutable `history`;
- the exact effective Rate Card revision and eligible plans, including list
  price, corporate discount, net price, included IUs, overage, billing timing,
  and `rolloverPolicy`;
- versioned `activityTerms`, including the activated social account-month;
  present that rate as planning information and state that it is not currently
  charged pending separate charging ratification;
- the governing agreement and `nextAction` (`ACCEPT_GOVERNING_AGREEMENT`,
  `SELECT_IU_PLAN`, or `NONE`); and
- any automatic `setupGrant`, including granted IUs, grant/expiry timestamps,
  status, no-rollover rule, and promotion/referral attribution kind.

Never quote a plan price, IU amount, expiry, discount, or rollover allowance
from memory; use the values returned by the current offer. The current
unpublished draft proposes one non-stacking 100-IU/60-day setup grant for a
genuinely new seller billing account, but that offer is unavailable until an
effective Rate Card makes it available. For an existing organization, say a
grant exists only when the response includes `setupGrant`, and use its returned
amount and timestamps. Its clock begins at the commercial signup commit even if
storefront provisioning reconciles asynchronously; later storefronts do not
mint another grant. A promotion or referral code only attributes that same
grant; it does not add credits, alter a price, or promise a referral reward.
Committed plans may carry at most the returned `rolloverPolicy` amount into the
following billing period only; pay-as-you-go usage and setup grants do not roll
over.

`accept_iu_rate_card_offer` is an **app-only** commit operation. It is
not model-visible and must not be called manually or imitated through
`api_call`. The bound MCP App supplies the exact displayed `offerVersion`,
selected `planId`, and idempotency key only after an authorized human
organization administrator confirms. A stale offer must be reloaded and
reviewed. Service tokens and impersonating staff cannot accept a paid plan.

For a storefront workload, no paid binding means pass-through to a third-party
sales agent at no cost; merchandising requires a paid plan. Buyer connection
and basic access remain available at no cost. Never invent a second side-specific
plan, discount, rollover policy, or wallet. Existing media-percentage pricing is
separate; this IU Rate Card does not amend existing accepted media terms.

Customer-facing details live in
`mintlify/v2/storefront/billing/overview.mdx`; the commercial source of truth is
`docs/spec/strategy/byoba-rate-card.md` and
`docs/spec/storefront/commercial-rate-card-binding.md`.

---

## Available Endpoints

### Storefront Management

Storefronts represent a partner's sellable inventory configuration. Each storefront is keyed by a `platformId` slug (e.g. "snapchat", "acme-media"). Each seller has exactly one storefront, which is **automatically created** when the seller is created. There is no "create storefront" endpoint — use `GET /storefront` to retrieve it and `PUT /storefront` to configure it.

#### Get Customer Storefront
```http
GET /api/v2/storefront
```

Returns the storefront for the authenticated customer, or 404 if none exists.

---

#### Update Storefront
```http
PUT /api/v2/storefront
{
  "name": "Acme Media Network",
  "publisherDomain": "acme.com",
  "operatorDomain": "acme.com",
  "brandName": "Acme",
  "logoUrl": "https://acme.com/logo.svg"
}
```

**Optional Fields:** `name`, `publisherDomain`, `operatorDomain`, `brandName`, `logoUrl`, `plan`, `transacting`, `defaultCurrency`, `paymentCurrencies`, `supportedLanguages`, `regions`, `description`

**Settlement currency:**
- `defaultCurrency` — the seller-confirmed currency the storefront settles in, as an ISO-4217 code (`^[A-Z]{3}$`, e.g. `"USD"`, `"EUR"`). **Required before go-live for seller-settled storefronts and never defaulted silently** — until it is set, `GET /storefront/readiness` returns the `currency_confirmed` check as `missing` with `isBlocker: true`. Set it here: `PUT /storefront { "defaultCurrency": "USD" }`. This is the ONLY way to set it — there is no dedicated currency endpoint. It is **independent of payout details**: the payout currency saved with `PUT /billing/payout-details` is a separate value and does NOT satisfy `currency_confirmed`. If you don't know the seller's currency, ask them; do not guess. Direct sales adapter storefronts run by our expert agents skip this readiness check because Interchange does not pay the seller on that path.
- `paymentCurrencies` — optional full set of currencies the storefront will be paid in directly. Include `defaultCurrency` when both fields are sent together. If omitted or empty, direct settlement falls back to `defaultCurrency` only.

**Brand fields (AAO-driven):**
- `operatorDomain` — the apex domain of the storefront operator. Populate from AAO resolution or user input.
- `brandName` — display brand name (e.g. "Acme Media"). Populate from `POST /resolve-brand` or user input.
- `logoUrl` — absolute URL to the brand logo. Populate from `POST /resolve-brand` or user input.

**Response-only fields (not accepted in body):**
- `operatorDomainVerified` — server-computed. Set to `true` only when the supplied `operatorDomain` matches the authenticated customer's registered `customerDomain` and that `customerDomain` is approved by active-member email ownership or Scope3 admin attestation; `false` otherwise. Clients cannot self-certify.

**Lifecycle model:**
A storefront has two stored driver fields and one derived display label:

| Stored | Type | Meaning |
|--------|------|---------|
| `transacting` | boolean | Operator-controlled toggle. `true` means the storefront is open to transactions; `false` means it is not. Default `false` on creation. |
| `archivedAt` | timestamp or null | Set by `POST /storefront/archive`. Once set, the storefront is read-only. |

| `displayStatus` (read-only, derived) | When |
|--------|------|
| `configuring` | `transacting = false` and not archived. Default state — visible to the marketplace but not accepting transactions. |
| `transacting` | `transacting = true` and not archived. Fully live, visible to buyers. |
| `archived` | `archivedAt IS NOT NULL`. Hidden from buyers, read-only on every mutation path. |

**Mutating the lifecycle:**
- `PUT /storefront { "transacting": true }` opens the storefront. Rejected with `400` if readiness is `blocked` (call `GET /storefront/readiness` and resolve blockers first).
- `PUT /storefront { "transacting": false }` pauses the storefront back to `configuring`.
- `POST /storefront/archive` archives the storefront. Atomic with `transacting = false`. Once archived, every subsequent mutation rejects with `Cannot modify an archived storefront`.

**Example — open the storefront for transactions:**
```http
PUT /api/v2/storefront
{ "transacting": true }
```

**Example — archive the storefront (irreversible):**
```http
POST /api/v2/storefront/archive
```

---

#### Resolve Brand from AAO Registry

Resolves a brand profile from the Agentic Advertising Organization (AAO) registry for a given domain. Use this during onboarding to auto-populate `brandName` and `logoUrl` before calling `PUT /storefront`.

```http
POST /api/v2/storefront/resolve-brand
{ "domain": "acme.com" }
```

**Required Fields:**
- `domain` (string): Apex domain to resolve (e.g., `acme.com`)

**Response (resolved):**
```json
{
  "resolved": true,
  "domain": "acme.com",
  "brandName": "Acme",
  "logoUrl": "https://acme.com/logo.svg",
  "manifestUrl": "https://acme.com/.well-known/brand.json"
}
```

**Response (unresolvable):**
```json
{
  "resolved": false,
  "domain": "acme.com",
  "error": "No brand profile found",
  "builderUrl": "https://agenticadvertising.org/brand"
}
```

When `resolved` is `false`, direct the user to the `builderUrl` to register their brand in the AAO registry, or have them supply `brandName` and `logoUrl` manually on the `PUT /storefront` call.

---

#### Patch Storefront Capabilities

Named operation: `update_storefront_capabilities`.

```http
PATCH /api/v2/storefront
{
  "capabilities": {
    "offersCreativeReview": true,
    "offersCampaignApproval": true,
    "offersProductComposition": true
  }
}
```

A capabilities-only shortcut to `PUT /storefront` — patch one or more of `offersCreativeReview`, `offersCampaignApproval`, `offersProductComposition` without sending the rest of the storefront configuration. Same capabilities-lock rule as `PUT /storefront`: ad-server-backed (ESA) storefronts have all three locked on, and a patch that would turn any of them off is rejected with a validation error.

**Omitted flags are left unchanged.** This is a true partial patch — sending `{ "capabilities": { "offersCreativeReview": true } }` updates only that flag and leaves the other two at their stored values. Send only the flags you intend to change.

---

---

### Inventory Sources (Agent Registration)

Inventory sources connect the storefront to product feeds. When `executionType` is `"AGENT"`, creating an inventory source also registers the agent. Storefronts can connect as many `AGENT` inventory sources as needed — no per-plan cap is enforced today. `MANAGED_SALES_AGENT` rows are always slot-exempt.

> **Enum values are UPPERCASE and case-sensitive.** `executionType` must be exactly `AGENT`, `MANAGED_SALES_AGENT`, `LINKED_STOREFRONT`, or `MODULAR_SOURCE`. Inventory-source `status` must be exactly `PENDING`, `ACTIVE`, or `DISABLED`. Lowercase values (e.g. `"agent"`, `"pending"`) are rejected with a validation error — send the uppercase value.

> **Two different paths.** An external AdCP sales agent reachable at a URL is `executionType: AGENT` — register it with `POST /inventory-sources` as shown below. An ad server (Google Ad Manager, FreeWheel, SpringServe) is `executionType: MANAGED_SALES_AGENT` and uses a **separate** connect flow — do NOT register an ad server as an `AGENT` with an `endpointUrl`. See "Connecting an ad server" below.

#### Connecting an ad server (GAM / FreeWheel / SpringServe)

An ad server is a `MANAGED_SALES_AGENT` source, not an `AGENT` URL. There is **no `endpointUrl`** — Interchange runs the sales-agent plumbing in front of the ad server. Use the `esa` connect operations, not `POST /inventory-sources`.

For Google Ad Manager specifically, Scope3 never takes a password or API token. Instead it provisions a per-customer service account and returns its email; a **human** grants that email access inside the GAM console. That grant is a prerequisite the agent cannot perform via API — surface the email and ask the operator to add it.

**Agent-facing sequence:**

1. **Ensure the service account** — `ensure_esa_service_account` (`POST /esa/service-account`, no body). Returns `{ "serviceAccountEmail": "scope3-sa-...@...iam.gserviceaccount.com" }`. This email is shared across all of the storefront's ad-server sources.
2. **Have a human grant it access in GAM** — the operator adds `serviceAccountEmail` as a user in their Google Ad Manager network and assigns the least-privilege role that can read inventory and traffic campaigns (`Trafficker`, or an equivalent custom role). The agent CANNOT do this step — present the exact email and wait for the operator to confirm the grant. For why this access is needed, the permission level, and the read-plus-traffic scope, point the operator to [Why Interchange needs access to your GAM network](https://docs.interchange.io/v2/storefront/inventory-sources/gam-access) — do not restate the rationale here.
3. **Create the connection with the network code** — `create_esa_connection` (`POST /esa`). The body is the ad-server config directly (not wrapped). For GAM: `{ "type": "google_ad_manager", "networkCode": "12345678" }`. `networkCode` is the numeric GAM network code (digits only) and is required for GAM. Provisioning probes GAM during this call, so run it only after the grant in step 2.
4. **Confirm reachability** — read the source with `get_esa_connection` (`GET /esa/{esaId}`) and check `deactivatedAt` (null = enabled) and `lastErrorCode`; re-probe with `test_esa_connection` (`POST /esa/{esaId}/test-connection`). On `lastErrorCode: "ADAPTER_PERMISSION_DENIED"` (or `test_esa_connection` returning `errorCode: "ADAPTER_PERMISSION_DENIED"`), the service-account email was not added or the grant has not propagated yet — confirm the email and role in GAM, wait a minute or two, and retry. On `ADAPTER_NETWORK_NOT_FOUND`, the `networkCode` is wrong — re-check it.

```http
POST /api/v2/storefront/esa/service-account
```
Response: `{ "serviceAccountEmail": "scope3-sa-acme@scope3-esa.iam.gserviceaccount.com" }`

```http
POST /api/v2/storefront/esa
{ "type": "google_ad_manager", "networkCode": "12345678" }
```
Response: `{ "id": "42", "serviceAccountEmail": "scope3-sa-acme@scope3-esa.iam.gserviceaccount.com", "lastErrorCode": null, ... }`

```http
POST /api/v2/storefront/esa/42/test-connection
```
Response (grant not yet propagated): `{ "success": false, "errorCode": "ADAPTER_PERMISSION_DENIED", "error": "...", "testedAt": "2026-06-21T10:00:00Z" }`

FreeWheel and SpringServe follow the same `esa` connect flow but pass their own credentials in the `create_esa_connection` body — they do not use the GAM service-account grant.

#### Browse ad-server inventory (`browse_ad_server_selectors`)

Once an ad server is connected, the dedicated **`browse_ad_server_selectors`**
MCP tool opens the inventory browser over its cached inventory. It is a separate
tool (not an `api_call`) so any MCP-UI host renders the result as the
inventory-selector widget.

**Inputs:**
- `esaId` (string, required) — the embedded sales agent's internal id (positive-integer string), from your inventory-source list / source diagnostics.
- `selectorType` (string, optional) — a selector type the connected adapter supports. **Omit** to return the adapter's *capabilities* (the list of valid selector types); **provide** to browse selectors of that type.
- `query` (string, optional) — search string.
- `parentId` (string, optional) — parent selector id (when the adapter supports it).
- `limit` (integer, optional, 1–100) — page size.
- `cursor` (string, optional) — page cursor from a prior call.

**Two modes:**
- `esaId` only → adapter **capabilities** (selector types differ per ad server: GAM exposes `ad_unit`/`placement`; FreeWheel `site`/`site_section`/`ad_unit_package`; SpringServe `supply_partner`/`supply_tag` — always read these before assuming a type).
- `esaId` + `selectorType` → the matching **selectors**, paginated via `cursor`.

**Result:** a conformant MCP-UI tool result — it declares
`_meta.ui.resourceUri = ui://agentic-api/inventory-selector/mcp-app.html` and
returns the inventory artifact as its own `structuredContent`
(`{ inventoryMcpui: { kind, resourceUri, view, canonicalSurface, esaId, data } }`),
so the host renders the inventory-selector widget directly from the result. The
model receives only a one-line summary; the full inventory rides
`structuredContent`. Read-only and customer-scoped — it resolves the ESA's
tenant from your session, so you never pass a customer id.

For modular inventory source setup, use the published guide at
`/v2/storefront/inventory-sources/modular-lifecycle`. Treat decks, media kits,
market indicators, co-viewing summaries, and screenshots as seller context, not
as commit-ready avails. A feed-backed modular source needs row-level avails with
stable ids, collection mapping, dates, net sellable capacity or gross capacity
plus upstream booked counts, pricing when available, and booking/reporting
ownership.

#### Create Inventory Source

```http
POST /api/v2/storefront/inventory-sources
{
  "sourceId": "retail-network-agent",
  "name": "Retail Network Agent",
  "executionType": "AGENT",
  "type": "SALES",
  "endpointUrl": "https://agent.example.com/adcp",
  "protocol": "MCP",
  "authenticationType": "API_KEY",
  "auth": { "type": "bearer", "token": "my-api-key" }
}
```

**Required Fields:**
- `sourceId` (string): Unique ID within the storefront
- `name` (string, 1-255 chars): Display name

**Required when `executionType` is `"AGENT"`:**
- `type` (string): Agent type — `"SALES"`, `"SIGNAL"`, `"CREATIVE"`, or `"OUTCOME"`. Cannot be changed after creation.
- `endpointUrl` (string): Agent endpoint URL (public HTTPS)
- `protocol` (string): `"MCP"` or `"A2A"`
- `authenticationType` (string): `"API_KEY"`, `"NO_AUTH"`, `"JWT"`, `"OAUTH"`, or `"BASIC_AUTH"`
- `auth` (object): Initial credentials. Required for `API_KEY`, `JWT`, and `BASIC_AUTH` agents. For Basic auth, use `{ "type": "basic", "username": "...", "password": "..." }`. Omit for `NO_AUTH` and `OAUTH`.

**Optional Fields:**
- `executionType` (default: `"AGENT"`) — UPPERCASE enum; see the casing note above
- `description` (string): Agent description

**Note on OAUTH agents:** When `authenticationType` is `"OAUTH"`:
- No `auth` field needed — the platform handles OAuth automatically
- The response includes `oauth.authorizationUrl` — present this to the user
- The platform discovers OAuth endpoints from the agent's AdCP endpoint URL

**Completing OAuth (API-only clients — no polling required):**

The OAuth dance happens out-of-band: a human opens `oauth.authorizationUrl`, signs in to the agent, and the platform's hosted callback writes the resulting token back. **API clients do NOT need to poll for completion.** Instead:

1. `POST /inventory-sources` with `authenticationType: "OAUTH"` → response returns `oauth.authorizationUrl`.
2. Hand that URL to a human (email, Slack, CLI prompt, agent message). Have them complete it on any device.
3. **Just retry your next call.** If the human hasn't finished, the next call (e.g. `PUT /storefront { transacting: true }`) returns a deterministic error and you retry on backoff.

When OAuth is incomplete, `GET /storefront/readiness` reports the `agent_status` check as `missing` (the agent stays `PENDING` until the token lands), and `PUT /storefront { transacting: true }` returns `400 INVALID_STATE` with `readiness: blocked`. There is no separate "OAuth status" endpoint — the agent's `status` and the storefront's `readiness` ARE the status. `GET /storefront/inventory-sources/{sourceId}` also exposes `authConfigured: boolean` if you want a single-field check (`true` once OAuth completes for that source).

This means a fully scriptable end-to-end onboarding flow for OAUTH agents looks like:

```
POST /storefront/inventory-sources { authenticationType: "OAUTH", ... }
   ↳ get oauth.authorizationUrl, hand to human
   ↳ human approves on their own device
PUT  /storefront { transacting: true }
   ↳ on first call: 400 INVALID_STATE if human still pending — retry
   ↳ when readiness flips to "ready": 200, storefront is live
```

No polling endpoint. No webhook. Just retry-on-error.

**Response (201):**
```json
{
  "sourceId": "retail-network-agent",
  "name": "Retail Network Agent",
  "executionType": "AGENT",
  "status": "PENDING",
  "agentId": "retailnetworkagent_abc123",
  "type": "SALES",
  "endpointUrl": "https://agent.example.com/adcp",
  "protocol": "MCP",
  "authenticationType": "API_KEY",
  "authConfigured": true,
  "createdAt": "2026-03-17T10:00:00Z",
  "updatedAt": "2026-03-17T10:00:00Z"
}
```

**Response (OAUTH agent):**
```json
{
  "sourceId": "oauth-agent",
  "agentId": "oauthagent_x1y2z3",
  "status": "PENDING",
  "authenticationType": "OAUTH",
  "oauth": {
    "authorizationUrl": "https://seller.example.com/oauth/authorize?client_id=abc&...",
    "agentId": "oauthagent_x1y2z3",
    "agentName": "OAuth Agent"
  }
}
```

---

#### List Inventory Sources
```http
GET /api/v2/storefront/inventory-sources
```

---

#### Get Inventory Source
```http
GET /api/v2/storefront/inventory-sources/{sourceId}
```

---

#### Get Inventory Source Diagnostics

Named operation: `get_source_diagnostics`.

```http
GET /api/v2/storefront/inventory-sources/{sourceId}/diagnostics?windowHours=24
```

Recent call health, latency, timeout, failure, and prior-window movement diagnostics for one inventory source — the same data backing the Source Diagnostics widget. `windowHours` (optional, default set server-side, max 720) sizes both the current window and the immediately-preceding comparison window.

---

#### Run Inventory Source Discovery Test

Named operation: `run_inventory_source_discovery_test`.

```http
POST /api/v2/storefront/inventory-sources/{sourceId}/tests/discovery
{
  "brief": "Return representative products available to a US buyer."
}
```

Runs a direct, read-only `get_products` check against one external sales-agent
source. It creates no media buy and spends nothing. The response distinguishes a
successful product response, a successful empty response, an upstream failure,
and a request that never reached the source; the durable test run has no
synthetic Murph conversation. `brief` is optional and uses a bounded diagnostic
default when omitted.

---

#### Update Inventory Source

```http
PUT /api/v2/storefront/inventory-sources/{sourceId}
{
  "status": "ACTIVE"
}
```

**Optional Fields:** `name`, `executionType`, `status`

**Status Transitions:**
- `pending → active`: Activates the source and the linked agent. Requires auth configured.
- `pending → disabled`: Disables the source and the linked agent.
- `active → disabled`: Disables the source and the linked agent.

---

#### Delete Inventory Source
```http
DELETE /api/v2/storefront/inventory-sources/{sourceId}
```

Deletes the source and disables the linked agent. **Response:** 204 No Content

---

### Ad-server source — Products

Use these endpoints when a storefront operator asks to list, inspect, edit, or remove wholesale products for an embedded sales agent. First list ESAs with `GET /api/v2/storefront/esa`, choose the active ESA, then call the product endpoint.

#### List products

Named operation: `list_esa_products`.

```http
GET /api/v2/storefront/esa/{esaId}/products
```

List response:
```json
{
  "products": [
    { "productId": "premium-display", "name": "Premium Display", "status": "ACTIVE" }
  ]
}
```

If the list is empty, say that no products were returned for that ESA. If the call fails, surface the error instead of guessing.

#### Get a product

Named operation: `get_esa_product`.

```http
GET /api/v2/storefront/esa/{esaId}/products/{productId}
```

The response body is the upstream adapter's product payload (varies by ad server), plus an `updateDraft` field: a request-shaped snapshot of the editable authoring fields (`wholesale_product_id`, `name`, `description`, `status`, `delivery_type`, `inventory`), snake_cased and with nulls dropped. Seed an edit form from `updateDraft` and PUT it back via `update_esa_product` — the upstream endpoint requires a complete replacement body, not a partial patch.

#### Update a product

Named operation: `update_esa_product`.

```http
PUT /api/v2/storefront/esa/{esaId}/products/{productId}
```

- `pathParams`: `{ "esaId": "123", "productId": "premium-display" }`
- `body`: the full replacement product request, generally seeded from a prior `get_esa_product` call's `updateDraft`. Unknown/extra fields are forwarded to the upstream adapter as-is.

Returns the updated product (upstream adapter payload, varies by ad server).

#### Delete a product

Named operation: `delete_esa_product`.

```http
DELETE /api/v2/storefront/esa/{esaId}/products/{productId}
```

- `pathParams`: `{ "esaId": "123", "productId": "premium-display" }`

No request body. Returns `204 No Content` on success.

---

### Ad-server source — Inventory Selector Search (Browse Ad Units & Placements)

Use these read-only endpoints when a storefront operator needs to browse or search the ad-server inventory — ad units, placements — behind an embedded sales agent before drafting products. First list ESAs with `GET /api/v2/storefront/esa`, choose the active ESA, then discover the valid selector types before searching.

```http
GET /api/v2/storefront/esa/{esaId}/inventory/capabilities
GET /api/v2/storefront/esa/{esaId}/inventory/selectors?selectorType=ad_unit&q=homepage&limit=50
GET /api/v2/storefront/esa/{esaId}/inventory/publisher-properties
GET /api/v2/storefront/esa/{esaId}/creative-formats
```

Capabilities response:
```json
{
  "adapter": "google_ad_manager",
  "selectorTypes": [
    { "selectorType": "ad_unit", "label": "Ad units", "supportsParentFilter": true },
    { "selectorType": "placement", "label": "Placements" }
  ],
  "pricingModels": ["cpm"],
  "targetingDimensions": ["geo", "device"]
}
```

Selector search query parameters:
- `selectorType` or `selector_type` — required; use a value from `selectorTypes[].selectorType` or `selectorTypes[].id`.
- `q` or `query` — optional search text, for example `homepage`, `ctv`, or `youtube`.
- `parentId` or `parent_id` — optional parent selector id for hierarchical browsers when `supportsParentFilter` is true.
- `limit` — optional page size, 1-100.
- `cursor` — optional page cursor from the prior response.

Selector response:
```json
{
  "items": [
    {
      "selectorType": "ad_unit",
      "externalId": "123456",
      "name": "Homepage",
      "path": ["Root", "News", "Homepage"],
      "parentId": "root"
    }
  ],
  "nextCursor": null
}
```

When using Murph directly, the equivalent tools are `get_esa_inventory_capabilities`, `search_ad_server_selectors`, `lookup_publisher_properties`, and `list_esa_creative_formats`. Do not guess paths such as `/esa/{esaId}/selectors`; the selector route is `/api/v2/storefront/esa/{esaId}/inventory/selectors`.

When a selector response includes ad sizes such as `728x90`, `300x250`, or `970x250`, treat them as ad-server inventory slot sizes. Do not present them as ADCP creative format IDs and do not ask the operator to choose legacy format catalog IDs while drafting an ESA product. Use the observed sizes to explain the selected inventory and to validate product constraints against the ESA/storefront capabilities.

---

### Ad-server source — Sync Streams

Use the regular ESA status operation first when diagnosing a managed ad-server source:

```http
GET /api/v2/storefront/esa/{esaId}/status
```

The default response is the compact health snapshot. For drill-down diagnostics, keep the same operation and add bounded recent sync runs:

```http
GET /api/v2/storefront/esa/{esaId}/status?includeSyncHistory=true&syncHistoryLimit=5
```

Query parameters:
- `includeSyncHistory` — optional boolean. `true` includes recent runs under each sync block's `history.recentRuns`.
- `syncHistoryLimit` — optional page size for each sync stream, 1-10. Defaults to 5 when `includeSyncHistory=true`.

In MCP, call `api_call` with `operation: "get_esa_status"`, `pathParams: { "esaId": "..." }`, and put these values in `params`.

For cursor-paginated drill-down beyond the bounded `recentRuns` on the status snapshot, use the dedicated sync-history operation instead:

```http
GET /api/v2/storefront/esa/{esaId}/sync-history
```

Named operation: `get_esa_sync_history`, `pathParams: { "esaId": "..." }`. Query parameters — `syncType`, `status`, `limit`, and `cursor` for paging. This is a connection-id alias of the REST-only `GET /inventory-sources/{sourceId}/sync-history` route. Use this when the operator needs to page further back than the status snapshot's `recentRuns`.

Use this endpoint when `signalCoverage` or `pricingAvailability` sync streams are stuck in `never_run` or have not run despite the ESA being active. These streams are platform-derived and are not triggered by the operator-facing refresh tool — use this admin endpoint instead.

```http
POST /api/v2/admin/embedded-sales-agents/{esaId}/sync/refresh
```

No request body. The call triggers all enabled sync streams for the tenant, including `signalCoverage` and `pricingAvailability`.

Response:
```json
{
  "data": {
    "status": "started",
    "syncRunIds": {
      "signalCoverage": "run_abc123",
      "pricingAvailability": "run_def456"
    },
    "runningSyncTypes": ["signalCoverage", "pricingAvailability"],
    "startedAt": "2026-06-09T12:00:00.000Z"
  }
}
```

`status: "already_running"` means a sync started within the last 60 seconds is still in flight — not an error. The `syncRunIds` in that case identify the existing runs. Safe to call again; the upstream deduplicates within a 60-second window.

Get the `esaId` from `GET /api/v2/storefront/esa` (the `id` field on the active ESA connection), or from `GET /api/v2/admin/embedded-sales-agents/health` (each item has an `esaId` field).

---

### Managed GAM Buyer Routing

These endpoints apply to embedded sales agents backed by Google Ad Manager. Use them when a storefront operator asks about GAM advertisers, buyer routing, the default GAM advertiser blocker, or creating an Interchange catch-all/default advertiser.

Do not confuse these with buyer-side Interchange advertiser records from the buyer API. In a storefront setup context, "what advertisers do I have?" usually means "what GAM advertiser records are available for this managed sales agent." First list ESAs with `GET /api/v2/storefront/esa`, choose the active GAM ESA, then use the endpoints below.

#### List Cached GAM Advertisers

```http
GET /api/v2/storefront/esa/{esaId}/gam/advertisers?q=Interchange&limit=50
```

Query parameters:
- `q` or `query` (optional): case-insensitive substring on advertiser name, or exact ID if numeric.
- `limit` (optional): page size, max 500.
- `cursor` (optional): cursor from the prior page.

Response:
```json
{
  "advertisers": [
    {
      "id": "123456",
      "name": "Interchange Default",
      "status": "ACTIVE",
      "currencyCode": "USD"
    }
  ],
  "nextCursor": null,
  "syncedAt": "2026-06-05T12:00:00Z"
}
```

`syncedAt` is the cache timestamp. If the operator just created an advertiser directly in GAM and it does not appear yet, refresh or wait for the sync before telling them it is missing.

#### Ensure a GAM Advertiser Exists

Use this when the operator asks to create an advertiser such as "Interchange Default" or "Wonderstruck Programmatic" for buyer routing. Confirm the exact name before calling this mutation.

```http
POST /api/v2/storefront/esa/{esaId}/gam/advertisers/ensure
{
  "name": "Interchange Default"
}
```

Optional field:
- `dryRun` (boolean): validates what would happen without creating.

Response:
```json
{
  "advertiser": {
    "id": "123456",
    "name": "Interchange Default",
    "status": "ACTIVE",
    "currencyCode": "USD"
  },
  "created": true,
  "dryRun": false
}
```

If the response is a validation error saying the GAM credential cannot create advertisers, explain that the configured GAM user lacks create permission. The operator can create the advertiser in GAM, then return to buyer routing and select it after the cache syncs.

Important: ensuring an advertiser creates or finds the GAM company record; it does not by itself select that advertiser as the tenant default. If the readiness blocker is specifically "Default GAM advertiser", set it directly with `PUT /api/v2/storefront/esa/{esaId}/gam/default-advertiser` using the advertiser id returned by list/ensure. Do not send the operator to the embedded sales-agent UI for this.

#### Set the Default GAM Advertiser

Use this after the operator has chosen a GAM advertiser from the list or after `ensure` returns the advertiser to use as the catch-all/default.

```http
PUT /api/v2/storefront/esa/{esaId}/gam/default-advertiser
{
  "gamAdvertiserId": "123456"
}
```

Response:
```json
{
  "defaultGamAdvertiserId": "123456"
}
```

For "create the Interchange default advertiser", the normal no-UI sequence is:
1. `POST /api/v2/storefront/esa/{esaId}/gam/advertisers/ensure` with the confirmed name.
2. `PUT /api/v2/storefront/esa/{esaId}/gam/default-advertiser` with `advertiser.id` from the ensure response.
3. Re-check readiness/status and report whether the default advertiser blocker cleared.

#### Buyer-to-GAM Advertiser Mappings

Mappings route a specific buyer/operator/brand tuple to a specific GAM advertiser instead of the default advertiser.

```http
GET /api/v2/storefront/esa/{esaId}/buyer-advertiser-mappings
GET /api/v2/storefront/esa/{esaId}/buyer-advertiser-mappings?operatorDomain=wpp.com
POST /api/v2/storefront/esa/{esaId}/buyer-advertiser-mappings
PATCH /api/v2/storefront/esa/{esaId}/buyer-advertiser-mappings/{mappingId}
DELETE /api/v2/storefront/esa/{esaId}/buyer-advertiser-mappings/{mappingId}
```

Create body:
```json
{
  "operatorDomain": "wpp.com",
  "brandHouse": "nike.com",
  "gamAdvertiserId": "123456"
}
```

Optional create fields: `brandHouse`, `brandId`, `principalId`.

#### Recent Buyers

```http
GET /api/v2/storefront/esa/{esaId}/recent-buyers?days=30&limit=100
```

Use this to show which recent buyers resolved through the default advertiser versus a specific mapping. The response includes `resolvedVia` and `resolvedGamAdvertiserId`.

---

### GAM Inventory Selector Search — Browse Ad Units & Placements (Product Drafting)

Use these endpoints to browse inventory selectors from the ESA's cached ad-server inventory when building products from an ESA. The selector types available depend on the connected ad server (GAM, FreeWheel, SpringServe, …) — discover them via capabilities first. Each selector has an `externalId` — call these endpoints **before** authoring a product to find the correct `externalId` values to include.

**If `search_ad_server_selectors` is available** (Murph sessions only): use `get_esa_inventory_capabilities` to discover valid selector types, then `search_ad_server_selectors` to browse inventory. Both tools require an explicit `esaId` parameter — get it from a prior `list_esas` call. **If you are on the external `api_call` path**, use the REST endpoints below.

#### Get Inventory Adapter Capabilities

Returns the adapter type and the selector types the ESA supports. **Call this first** to determine which `selectorType` values are valid before running a selector search. In Murph context, call `get_esa_inventory_capabilities` instead.

```http
GET /api/v2/storefront/esa/{esaId}/inventory/capabilities
```

Response:
```json
{
  "adapter": "gam",
  "selectorTypes": [
    { "id": "ad_unit", "selectorType": "ad_unit", "label": "Ad Units", "supportsParentFilter": true },
    { "id": "placement", "selectorType": "placement", "label": "Placements", "supportsParentFilter": false }
  ],
  "pricingModels": ["CPM"],
  "targetingDimensions": ["geo", "device"],
  "optimizationGoals": ["CPM"]
}
```

All fields except `adapter` are optional — `selectorTypes` may be absent if the adapter has not published its capabilities yet. The valid selector types **differ by ad server** (GAM: `ad_unit`/`placement`; FreeWheel: `site`/`site_section`/`series`/`video_group`/`ad_unit_package`; SpringServe: `supply_partner`/`supply_tag`), so always pass a value the adapter actually declared. When `selectorTypes` is absent or empty, do NOT guess a type name (in particular, do not default to `placement` — it only exists on GAM and a FreeWheel/SpringServe search will be rejected); instead tell the operator the adapter has not declared its supported selector types and use `lookup_publisher_properties` `allowedSelectors[]` to find valid values.

Each `selectorTypes` entry may include both `id` and `selectorType` string fields — use `selectorType` as the value to pass to the selector search query parameter (e.g. `"ad_unit"`). `supportsParentFilter: true` means you can pass `parentId` on the selector search to browse children of a node (for example, ad units under a specific network node).

#### Search Inventory Selectors

Search the cached ad-server inventory for a specific selector type. Paginated via cursor.

```http
GET /api/v2/storefront/esa/{esaId}/inventory/selectors?selectorType=ad_unit&q=homepage
```

Query parameters:
- `selectorType` (required): Must be a `selectorType` value the connected adapter declared in the capabilities response — these differ per ad server, so never hardcode or guess one (e.g. `placement` is valid on GAM but rejected by FreeWheel).
- `q` or `query` (optional): Case-insensitive substring match on name.
- `parentId` (optional): Filter to children of this selector node (only when `supportsParentFilter` is `true` for the type).
- `limit` (optional): 1–100, defaults to 50.
- `cursor` (optional): Cursor from a prior page.

Response:
```json
{
  "items": [
    {
      "selectorType": "ad_unit",
      "externalId": "12345",
      "name": "Homepage Takeover",
      "path": ["Network", "Video"],
      "parentId": null,
      "status": "ACTIVE"
    }
  ],
  "nextCursor": null
}
```

`externalId` is the value to use when referencing this selector in a product definition. Iterate with `cursor` when `nextCursor` is non-null. `path`, `status`, and `parentId` are optional — treat their absence as no error. `parentId` may be `null` (root node) or absent entirely. `path` is a breadcrumb of ancestor node labels from the root (display-only; not used as an API input).

Selector metadata may include ad sizes such as `728x90`, `300x250`, or `970x250`. These are GAM slot sizes, not ADCP creative format IDs. When drafting a product from selected inventory, show them as observed ad sizes and do not ask the operator to bind legacy creative format catalog IDs; validate any product-format requirements through the ESA/storefront capability response instead.

**Note:** The selector cache is populated during the inventory sync run. If results are empty and the operator's GAM inventory sync has completed recently, the ESA's inventory may genuinely contain no matching selectors. If sync has not run, results may be empty as a cache artifact — the operator should confirm sync ran successfully before concluding a selector is absent.

---

### Storefront Readiness

Check whether a storefront meets go-live requirements.

```http
GET /api/v2/storefront/readiness
```

**Response:**
```json
{
  "platformId": "acme-media",
  "status": "blocked",
  "checks": [
    {
      "id": "inventory_sources",
      "name": "Inventory Sources",
      "description": "At least one inventory source must be configured before the storefront can start transacting.",
      "category": "inventory",
      "status": "missing",
      "isBlocker": true
    },
    {
      "id": "agent_status",
      "name": "Agent Status",
      "description": "All agent-backed inventory sources must have their agents activated.",
      "category": "agents",
      "status": "missing",
      "isBlocker": true,
      "details": "1 agent not yet active"
    },
    {
      "id": "agent_auth",
      "name": "Agent Authentication",
      "description": "Non-OAuth agents must have authentication credentials configured.",
      "category": "agents",
      "status": "complete",
      "isBlocker": true,
      "details": "1 agent with auth configured"
    },
    {
      "id": "currency_confirmed",
      "name": "Currency",
      "description": "Confirm the currency your storefront settles in before going live — it is never chosen for you.",
      "category": "billing",
      "status": "missing",
      "isBlocker": true
    }
  ]
}
```

**Status values:** `ready` (all checks pass), `blocked` (blockers present)

**Checks:**
- `inventory_sources`: At least one source configured
- `agent_status`: All agent-type sources have active agents
- `agent_auth`: All non-OAUTH agents have auth configured
- `billing_setup`: Payout bank details on file. **Optional** when no connected agent advertises `agent` billing in its capabilities — returns `status: optional`/`partial` with `isBlocker: false` and warns the seller their storefront is limited to external agreements with buyers (Scope3 will not clear payments). **Required (blocker)** once any connected agent declares `agent` in `accounts.supported_billing` — returns `status: missing`/`partial` with `isBlocker: true` until payout details are saved with `PUT /storefront/billing/payout-details`.
- `currency_confirmed`: The seller-confirmed primary settlement currency. **A blocker for seller-settled storefronts** (`isBlocker: true`). Returns `status: missing` until `storefront_config.default_currency` is set, then `complete`. Set it with `PUT /storefront { "defaultCurrency": "USD" }` (ISO-4217, never defaulted silently). To configure direct settlement in additional currencies, include `paymentCurrencies` with the full set. This is **separate from `billing_setup`** even though both fall under `category: "billing"` — saving payout details does NOT satisfy it. Direct sales adapter storefronts run by our expert agents skip this check because Interchange does not pay the seller on that path; do not nudge a seller for adapter settlement currency.
- `settlement_currency_match`: Verifies the confirmed `defaultCurrency` matches the payout currency declared with the storefront's payout details. Blocks go-live when they differ (every payout would otherwise FX-convert). Non-blocking when there are no payout details on file, the payout currency hasn't been captured yet, or `defaultCurrency` is unset (`currency_confirmed` owns that). Each payout account clears one currency — if they can't be made to match, the seller needs payout details in a currency they can directly settle.

---

### Publisher List Management

Declare the publisher domains your storefront covers. Buyers use this to evaluate reach before purchasing.

#### Endpoints

- `GET /api/v2/storefront/publishers` -- list all publisher rows (declared + crawled + discovered) with adagents.json resolution status and authorization verdict
- `PUT /api/v2/storefront/publishers` -- replace the declared list atomically. Body: `{ "publishers": ["bbc.com", "nytimes.com"] }`. Max 5,000 domains. Invalid domains fail the whole request.
- `POST /api/v2/storefront/publishers` -- add one domain. Body: `{ "domain": "bbc.com" }`. Accepts full URLs (`https://www.bbc.com/news`) and normalizes them automatically.
- `DELETE /api/v2/storefront/publishers/:domain` -- remove one declared domain. Returns 204 (idempotent). Returns 409 if the domain was not seller-declared (e.g. crawled or discovered).

#### Response shape (GET / PUT / POST)

```json
{
  "publishers": [
    {
      "domain": "bbc.com",
      "provenance": "declared",
      "adagentsStatus": "resolved",
      "authorizationStatus": "authorized",
      "propertyCount": 12,
      "lastSyncedAt": "2026-07-10T00:00:00Z"
    }
  ],
  "total": 1,
  "verified": 1
}
```

- `provenance`: `declared` (seller added), `crawled` (found via adagents.json scan), or `discovered` (surfaced through a buyer brief)
- `adagentsStatus`: `resolved` | `no_adagents` | `error` | `pending`
- `authorizationStatus`: `authorized` | `unauthorized` | `unknown` -- `unauthorized` domains are hidden from buyer-facing surfaces

#### Buyer visibility

Buyers see a summary (`total`, `verified`, and a sample of top domains) on the storefront detail, and can page through the full list via `GET /api/v2/buyer/storefronts/{storefrontId}/publishers` with optional `?q=&verification=verified&limit=&offset=` filters. Buyers can also filter the storefront list with `?publisherDomain=bbc.com` to find sellers that represent a given publisher (vendor selection). This is a coverage filter — it answers "which sellers cover this publisher," not "what can I buy on this domain."

---

### Storefront Billing (Payout Details)

Payout details are **optional by default** and **required** once a connected agent advertises `agent` billing in its capabilities (`accounts.supported_billing` contains `"agent"`). When optional and a seller does not add payout details, their storefront is limited to external agreements with buyers — Scope3 will not clear payments and the seller must bill the buyer directly. When required, the readiness check `billing_setup` returns `isBlocker: true` until payout details are on file.

Payouts are made by bank transfer in the seller's payout currency, run manually by Scope3 finance — there is no payment processor. For Scope3 to clear a buyer-cleared (`agent`) payment on a media buy, the storefront must have payout bank details on file. Two operations cover this: read the saved billing config, and set the payout bank details. A separate parent-only endpoint lists per-child payout status. Commercial terms (platform fee, additional fees, currency, payment terms) are set by Scope3 via `PUT /storefront/billing` (superadmin).

> If a seller previously set up payouts through a payment processor, they re-enter their bank details once with `PUT /storefront/billing/payout-details` — the prior connection does not carry over.

| Operation | Endpoint | Purpose |
|-----------|----------|---------|
| Get billing config | `GET /storefront/billing` | Payout setup status, masked payout details, fee schedule, currency, payment terms |
| Set payout details | `PUT /storefront/billing/payout-details` | Seller sets payout bank details (account-admin only) |
| Set commercial terms | `PUT /storefront/billing` | Platform fee, additional fees, currency, net days (superadmin) |
| List child payout status | `GET /storefront/billing/accounts` | Parent: per-child payout status |

#### Get Billing Config

```http
GET /api/v2/storefront/billing
```

Returns the storefront billing configuration. `payoutDetails` is `null` until the seller sets them.

**Response:**
```json
{
  "billing": {
    "onboardingStatus": "complete",
    "platformFeePercent": 3.5,
    "fees": [{ "name": "Data fee", "feePercent": 1.0 }],
    "currency": "USD",
    "country": "US",
    "defaultNetDays": 30,
    "inherited": false,
    "payoutDetails": {
      "beneficiaryName": "Acme Media Network Ltd",
      "addressLine1": "500 Market Street",
      "addressLine2": "Suite 400",
      "city": "San Francisco",
      "region": "CA",
      "postalCode": "94105",
      "countryCode": "US",
      "accountNumberLast4": "1234",
      "bankIdentifierType": "FEDWIRE_ABA",
      "bankIdentifierValue": "021000021",
      "completedAt": "2026-03-19T10:00:00Z"
    },
    "createdAt": "2026-03-19T10:00:00Z",
    "updatedAt": "2026-03-19T10:00:00Z"
  }
}
```

**Fields:**
- `onboardingStatus` — payout setup status: `pending` (no payout details yet), `complete` (payout details on file), or `restricted`. Derived from whether payout details are on file.
- `platformFeePercent`, `fees`, `defaultNetDays` — commercial terms set by Scope3; may be `null` before terms are configured.
- `currency` — payout currency (ISO-4217), or `null` until set.
- `country` — ISO 3166-1 alpha-2 country used for billing, or `null`.
- `inherited` — `true` means this storefront is a child customer that inherits its parent's billing configuration; payouts go to the parent account. A child can set its own payout details at any time, which then take precedence over the parent's.
- `payoutDetails` — masked payout bank details, or `null` when none are on file. `accountNumberLast4` is the last 4 characters only — the full account number is write-only and never returned.

---

#### Set Payout Details

Seller sets the payout bank details Scope3 uses to remit payouts by bank transfer. **Account-admin only.**

```http
PUT /api/v2/storefront/billing/payout-details
{
  "beneficiaryName": "Acme Media Network Ltd",
  "addressLine1": "500 Market Street",
  "addressLine2": "Suite 400",
  "city": "San Francisco",
  "region": "CA",
  "postalCode": "94105",
  "countryCode": "US",
  "accountNumber": "12345678901234",
  "bankIdentifierType": "FEDWIRE_ABA",
  "bankIdentifierValue": "021000021",
  "currency": "USD"
}
```

**Required Fields:**
- `beneficiaryName` (string): Payout beneficiary legal name.
- `addressLine1` (string): Beneficiary street address.
- `city` (string): Beneficiary city.
- `postalCode` (string): Beneficiary postal/ZIP code.
- `countryCode` (string): Beneficiary country, ISO 3166-1 alpha-2 (e.g. `"US"`, `"GB"`).
- `accountNumber` (string, 6–34 alphanumeric): Bank account number OR IBAN. **Write-only** — never returned; the response and `GET /billing` expose only `accountNumberLast4`. Required the first time payout details are set; **omit on later edits** to keep the stored value.
- `bankIdentifierType` (string): One of `FEDWIRE_ABA` (9-digit US Fedwire/ABA routing number), `CHIPS_ABA` (9-digit CHIPS ABA), `SWIFT_BIC` (8/11-character SWIFT-BIC), or `BANK_CODE` (local/domestic bank code).
- `bankIdentifierValue` (string): The identifier matching `bankIdentifierType`.
- `currency` (string): Payout currency, ISO-4217.

**Optional Fields:** `addressLine2`, `region`.

**Response:** the updated billing config (same shape as `GET /billing`), with `onboardingStatus: "complete"` and masked `payoutDetails` once details are on file.

---

#### List Child Payout Status (parent customers)

```http
GET /api/v2/storefront/billing/accounts
```

For a parent customer, lists each child customer's payout status.

**Response:**
```json
{
  "accounts": [
    {
      "customerId": 456,
      "name": "Acme West",
      "company": "Acme Media Network Ltd",
      "inherited": false,
      "onboardingStatus": "complete"
    }
  ]
}
```

`onboardingStatus` is `pending`, `complete`, `restricted`, or `none` (when the child inherits the parent's billing).

`GET /billing` and `PUT /billing/payout-details` accept `?targetCustomerId=<childId>` for parent customers operating on a child's billing.

---

### Plan & Billing (`open_plan_and_billing` / `get_billing_account`)

Two read-only tools cover the organization's consolidated commercial-account
view — plan/contract status, effective pricing, intelligence usage,
credit/prepay standing and balance, agreements, and the single next action (if
any) needed to become or remain paid. This is org-level, not the storefront
payout config above — a storefront pays out via `Storefront Billing`; every
organization (buyer or seller role) also has one Plan & Billing document for
its own commercial standing.

- **`open_plan_and_billing`** opens the Plan & Billing Page (the widget) —
  call this when the user asks to see their account status, plan, billing, or
  usage in the chat surface. Takes no arguments.
- **`get_billing_account`** returns the document as JSON — call this when you
  need the actual figures (balance, next action, usage) to answer a question
  or reason about them. Optional `section` param (one of `organization`,
  `scope`, `standing`, `plan`, `pricing`, `usage`, `payment`, `contracts`,
  `accounts`, `nextAction`, `permissions`) returns just that section to keep
  the response small; omit it for the full document. `section: "accounts"`
  also returns `totalAccounts` — the `accounts` array is capped at 50
  entries, and `totalAccounts` is the uncapped count.

Both tools are read-only — no mutation Tasks exist yet (accepting ToS, funding
a balance, applying for credit are separate, deferred flows; `accept_tos`
already exists standalone). Pricing, standing, and credit figures are ground
truth from this document — never state a balance, limit, or next action from
memory or a prior turn; call `get_billing_account` fresh.

### Adding a payment method (`add_payment_authority`)

When the organization needs a payment method for its own commercial standing
(this is org-level Plan & Billing, not storefront payout config), use the
`add_payment_authority` bounded Task. Card data never passes through you: the
Task returns a secure link the organization's own cardholder opens in a
browser (no sign-in needed) to enter the card directly with the payment
processor. Three steps: `action: "request"` → single-use `confirmationToken`
(5 min); `action: "confirm"` + that token → `{ method: "capture_link", url,
expiresAt }` (30 min, one verified use — save the URL, it is returned exactly
once); hand the URL to the cardholder, then poll `action: "status"` every
15-30 seconds until `verified` (or `expired` — start over). Requires
org-admin access or an org-scoped service token.

---

### Agent Discovery (Read-Only)

Agents registered through storefront inventory sources can be queried for buyer discovery. Two discovery modes are available:

- **AAO Discovery** (`GET /storefront/discover-agents`) — queries the AAO registry for agents declared by the operator/publisher of a domain. Used during onboarding to suggest agents to register.
- **Local List** (`GET /storefront/agents`) — lists agents already registered on the Scope3 platform, with account and status metadata. Used after agents have been added.

#### Invoke Agent Tool

Call an ADCP protocol tool directly on a connected agent. Use this to drive deal validation flows, test agent integrations, or invoke any ADCP tool the agent advertises — without needing to create a buyer-side media buy in Interchange.

```http
POST /api/v2/storefront/agents/{agentId}/invoke
```

**Request body:**
```json
{
  "tool": "update_media_buy",
  "arguments": {
    "media_buy_id": "SANDBOX-431E45DEC3FB41E7",
    "status": "active"
  }
}
```

**Supported tools:** `get_products`, `create_media_buy`, `update_media_buy`, `get_media_buys`, `get_media_buy_delivery`, `get_adcp_capabilities`

**Response:**
```json
{
  "tool": "update_media_buy",
  "result": { ... }
}
```

`result` is the raw `TaskResult` returned by the agent. The shape depends on the tool invoked.

**Notes:**
- Only ACTIVE agents owned by the caller (or marketplace agents with an active account) can be invoked
- The tool must appear in the agent's declared capabilities; if capabilities haven't been fetched yet, the call proceeds
- This is a passthrough relay — Interchange does not track the resulting deal or operation in its own database
- No webhook is registered; the response is the agent's synchronous reply

**Errors:**
- 400 if `tool` is not in the supported list or not advertised by the agent
- 404 if the agent does not exist or the caller does not have access to it

---

#### Refresh Agent Capabilities

Cached agent capabilities (including `require_operator_auth` and `accounts.supported_billing`) are served from a cache that revalidates lazily. When an agent changes what it advertises, the cache can lag — e.g. an agent that stops requiring operator auth may still show its sources as needing buyer credentials until the cache catches up. Force an immediate re-read:

```http
POST /api/v2/storefront/agents/{agentId}/capabilities/refresh
```

No request body. Re-reads the live agent and rewrites the cached capabilities and the denormalized `require_operator_auth`. A seller may refresh only agents their own storefront owns; refreshing an agent owned by another customer returns 403.

#### Discover Agents from AAO Registry

Queries the Agentic Advertising Organization (AAO) registry for agents declared by the operator and publisher of a given domain. Returns candidate agents that the storefront operator can register via `POST /inventory-sources`. Does not return Scope3-registered agents — use `GET /storefront/agents` for that.

```http
GET /api/v2/storefront/discover-agents?domain={operatorDomain}
```

**Query Parameters:**
- `domain` (string, required): Apex domain of the operator to discover agents for
- `refresh` (boolean, optional): Pass `true` to bypass the in-memory cache and re-fetch from AAO

**Response:**
```json
{
  "domain": "acme.com",
  "operator": {
    "agents": [
      {
        "url": "https://agent.acme.com/adcp",
        "name": "Acme Sales Agent",
        "type": "SALES",
        "compliance": {
          "status": "passing",
          "storyboards_passing": 8,
          "storyboards_total": 8,
          "headline": "All AAO conformance checks passed."
        },
        "storyboards": ["..."]
      }
    ]
  },
  "publisher": {
    "agents": [
      { "url": "...", "name": "...", "type": "SIGNAL", "compliance": {...}, "storyboards": [...] }
    ]
  }
}
```

`compliance.status` is one of `passing`, `pending`, `not-passing`, `not-registered`. See the [storefront onboarding guide](https://docs.interchange.io/v2/setup/storefront-onboarding) for the full conformance gate.

**Errors:**
- 400 if `domain` is missing or malformed
- 404 if no AAO manifest exists for the domain

---

#### List Agents

List all agents visible to the authenticated user. Supports filtering by type, status, and relationship.

```http
GET /api/v2/storefront/agents
```

**Query Parameters (all optional):**
- `type` (string): Filter by agent type — `SALES`, `SIGNAL`, `CREATIVE`, or `OUTCOME`
- `status` (string): Filter by status — `PENDING`, `ACTIVE`, or `DISABLED`
- `relationship` (string): Filter by relationship — `SELF` (owned by you), `MARKETPLACE` (all other marketplace agents)

**Response:**
```json
{
  "data": [
    {
      "agentId": "premiumvideo_a1b2c3d4",
      "type": "SALES",
      "name": "Premium Video Exchange",
      "description": "Premium CTV and video inventory provider",
      "endpointUrl": "https://api.premiumvideo.com/adcp",
      "protocol": "MCP",
      "authenticationType": "API_KEY",
      "status": "ACTIVE",
      "relationship": "MARKETPLACE",
      "customerAccounts": [
        {
          "accountIdentifier": "my-account-123",
          "status": "ACTIVE"
        }
      ],
      "requiresAccount": false,
      "authConfigured": true,
      "createdAt": "2026-01-15T10:00:00Z"
    }
  ]
}
```

**Notes:**
- All non-DISABLED agents are visible to everyone
- PENDING agents from other owners appear as `"status": "COMING_SOON"` with minimal info (name only)
- `customerAccounts` lists the caller's own accounts (excludes marketplace accounts)
- `requiresAccount` is true when the agent supports per-account registration and the caller has no accounts
- `authConfigured` indicates whether the agent has authentication credentials configured. For OAUTH agents: `true` means the OAuth flow is complete, `false` means the user still needs to authorize. Use this field (NOT `status`) to determine if OAuth is complete.
- `oauth` is present ONLY for owner's PENDING OAUTH agents where `authConfigured` is `false` (OAuth flow not yet completed)

**Display Requirements — ALWAYS include when listing agents:**
- `customerAccounts`: Show the caller's registered accounts for each agent
- `requiresAccount`: Highlight agents that require account registration
- `authConfigured`: Show whether authentication is set up
- Group agents by status (ACTIVE first, then COMING_SOON)
- For each agent, clearly indicate: name, type, status, number of caller's accounts, and whether registration is needed

---

#### Get Agent Details

Retrieve detailed information about a registered agent, including capabilities and account counts.

```http
GET /api/v2/storefront/agents/{agentId}
```

**Path Parameters:**
- `agentId` (string): The agent ID

**Response:**
```json
{
  "agentId": "acme-sales-agent",
  "name": "Acme Sales Agent",
  "type": "SALES",
  "status": "ACTIVE",
  "endpointUrl": "https://agent.acme.com/adcp",
  "protocol": "MCP",
  "authenticationType": "API_KEY",
  "authConfigured": true,
  "requiresAccount": true,
  "hasMasterAccount": false,
  "hasCustomerAccount": true,
  "customerAccountCount": 3,
  "capabilities": { "discover_products": true, "create_media_buy": true }
}
```

---

### OAuth Endpoints

#### Start Setup OAuth Flow

Initiate the OAuth flow for **agent-level setup**. Tokens are stored in the agent's configuration. The `redirectUri` is optional — if omitted, the platform-hosted callback URL is used automatically.

```http
POST /api/v2/storefront/agents/{agentId}/oauth/authorize
{}
```

#### Start Account OAuth Flow

Initiate the OAuth flow for **per-account registration**. Tokens are stored per-account. The `redirectUri` is optional — if omitted, the platform-hosted callback URL is used automatically.

```http
POST /api/v2/storefront/agents/{agentId}/accounts/oauth/authorize
{}
```

#### Platform OAuth Callback (Public - No Auth)

Platform-hosted callback that automatically completes the OAuth flow. AI agents should use this URL as their `redirectUri`.

```
GET /api/v2/storefront/oauth/callback?code={code}&state={state}
```

- **Production:** `https://api.interchange.io/api/v2/storefront/oauth/callback`
- **Staging:** `https://api.staging.interchange.io/api/v2/storefront/oauth/callback`

#### Exchange OAuth Code

Exchange an OAuth authorization code for tokens manually. Only needed if you have your own callback server.

```http
POST /api/v2/storefront/agents/{agentId}/oauth/callback
{
  "code": "authorization-code-from-redirect",
  "state": "state-parameter-from-redirect"
}
```

---

### Signals

Manage signals registered by signal agents. Signals represent data feeds (audience segments, contextual signals, etc.) that can be made available to buyers.

#### Create Signal

Register a signal with key types, regions, and access configurations.

```http
POST /api/v2/storefront/signals
{
  "signalId": "my-signal-id",
  "name": "Premium Audience Segment",
  "description": "High-intent users in NORAM",
  "keyType": ["property"],
  "regions": ["NORAM"],
  "agentId": "optional-agent-id",
  "isLive": false,
  "access": [
    {
      "advertiserId": 12345,
      "visibility": "PUBLIC",
      "price": { "model": "CPM", "price": 2.50 }
    }
  ]
}
```

**Required fields:** `signalId`, `name`, `keyType` (at least one)

#### List Signals

```http
GET /api/v2/storefront/signals?agentId={agentId}&visibility={PUBLIC|PROPRIETARY}&isLive={true|false}&advertiserId={advertiserId}&limit={20}&offset={0}&includeArchived={false}
```

All query parameters are optional filters.

#### Get Signal

```http
GET /api/v2/storefront/signals/{signalId}?advertiserId={advertiserId}&includeArchived={false}
```

**Path Parameters:**
- `signalId` (string): The signal identifier

#### Update Signal

Update signal metadata and manage access records. Cannot change `signalId`, `keyType`, or `regions` after creation.

```http
PUT /api/v2/storefront/signals/{signalId}
{
  "name": "Updated Name",
  "description": "Updated description",
  "isLive": true,
  "addAccess": [{ "advertiserId": 67890, "visibility": "PROPRIETARY" }],
  "updateAccess": [{ "accessId": 1, "visibility": "PUBLIC" }],
  "archiveAccess": [2, 3]
}
```

At least one field must be provided.

#### Delete Signal

Soft-deletes (archives) a signal and all its access records.

```http
DELETE /api/v2/storefront/signals/{signalId}
```

#### Discover Signals

Query signal agents via AdCP to discover available signals. Does not persist results.

```http
POST /api/v2/storefront/signals/discover
{
  "agentId": "optional-filter-to-specific-agent",
  "signalSpec": "natural language description of desired signals",
  "signalIds": ["specific-signal-id-1"],
  "limit": 20,
  "offset": 0
}
```

All fields are optional.

### Activity Feed

Lists audit log events for the storefront — agent registrations, inventory source changes, credential updates, and other storefront-side actions. Customer-scoped.

```http
GET /api/v2/storefront/audit-logs?take=50&skip=0
```

Query parameters (all optional):
- `startDate`, `endDate` — ISO timestamps to bound the range
- `inventorySourceId` — filter to a single source's activity
- `resourceTypes` — repeated (`?resourceTypes=INVENTORY_SOURCE&resourceTypes=AGENT`) or comma-separated (`?resourceTypes=INVENTORY_SOURCE,AGENT`)
- `take` (max 500, default 50 over REST; default 100 and max 100 through MCP), `skip`

Response:
```json
{
  "data": {
    "logs": [
      {
        "id": 1,
        "timestamp": "2026-04-01T00:00:00Z",
        "action": "CREATE",
        "resourceType": "INVENTORY_SOURCE",
        "resourceId": "source_abc123",
        "resourceName": "Example Source",
        "userEmail": "user@example.com",
        "description": "Created inventory source Example Source"
      }
    ],
    "total": 123
  },
  "meta": { "pagination": { "skip": 0, "take": 10, "total": 123, "hasMore": true } }
}
```

### Demand Signals (incoming buyer briefs)

> Behind the `demand-supply-signals` feature flag. If any of these endpoints return `FEATURE_NOT_ENABLED`, the feature is not turned on for this customer — tell the user demand-signal routing isn't available yet and don't fabricate a queue.

The demand-signals surface is the **seller-facing view** of buyer planning briefs that have been routed to a storefront the caller owns. Each row is a `demand_signal` joined to the per-storefront `demand_signal_target` that put it in this queue. A seller responds with one of four actions per signal: `QUOTE`, `CLARIFY`, `DECLINE`, `BOOK`.

**Seller-facing terminology:** call these "demand signals" or "incoming briefs". The buyer-facing label is "planning brief" / "prospective brief" — only relevant when the seller needs to talk back to the buyer.

> Don't confuse this live action queue with **Brief history (intelligence-runs)** below — that's the read-only record of briefs the agent has *already* answered, not demand waiting on the seller.

> **You do NOT supply `storefrontId` for these operations.** Use the named operations (`list_storefront_products`, `list_storefront_demand_signals`, `get_storefront_dashboard_summary`, `get_storefront_demand_signal`, `respond_to_demand_signal`) with **no `storefrontId` path param** — the server resolves the caller's storefront from auth (there is exactly one storefront per account). Never put a customer id, a slug, or a guessed number in `storefrontId`. The `{storefrontId}` shown in the raw paths below is filled in for you. (`signalId` is still required and comes from the list response.)

#### List demand signals routed to a storefront

```http
GET /api/v2/storefront/storefronts/{storefrontId}/demand-signals
```

Query parameters (all optional):
- `status` — `SEARCHING` / `QUOTED` / `BOOKED` / `ABANDONED` / `DECLINED`
- `dispatchStatus` — `QUEUED` / `DISPATCHED` / `ACKNOWLEDGED` / `ON_HOLD` / `FAILED` / `DECLINED`
- `channel` — `display` / `native` / `video` / `audio` / `ctv` / `dooh` / `newsletter` / `podcast`
- `minMatchPct` — 0..100, filters on the per-target match percentage
- `limit` (max 100, default 20), `offset`

Response items extend the buyer view with the per-target row and the **seller-internal** `matchScore` (0..100). The seller surface is the only place `matchScore` is exposed — never echo it back to a buyer.

#### Get a single demand signal

```http
GET /api/v2/storefront/storefronts/{storefrontId}/demand-signals/{signalId}
```

Returns `NOT_FOUND` if the signal exists but isn't routed to this storefront, or if the caller doesn't own the storefront (`ACCESS_DENIED`).

#### Respond to a demand signal

```http
POST /api/v2/storefront/storefronts/{storefrontId}/demand-signals/{signalId}/respond
```

The body is a discriminated union keyed by `kind`:

```json
{ "kind": "QUOTE",   "payload": { "proposedCpm": 12.5, "currency": "EUR", "notes": "best fit on Sat–Sun" } }
{ "kind": "CLARIFY", "payload": { "question": "Is iOS-only acceptable?" } }
{ "kind": "DECLINE", "payload": { "reason": "category conflict with current advertiser" } }
{ "kind": "BOOK",    "payload": { "dealId": "deal_abc123" } }
```

The endpoint is transactional:
1. Writes a `demand_signal_response` row.
2. Updates the per-storefront target's `dispatch_status` to `ACKNOWLEDGED` (for `QUOTE` / `CLARIFY` / `BOOK`) or `DECLINED` (for `DECLINE`).
3. Re-evaluates the parent `demand_signal.status` from the union of all its targets: any `BOOK` → `BOOKED` (terminal), any `QUOTE` → `QUOTED`, all targets declined → `DECLINED`. `BOOKED` is terminal and never walked backward.

Returns the created response row with status 201.

#### Preview the agent's response (dry-run)

```http
POST /api/v2/storefront/storefronts/{storefrontId}/demand-signals/{signalId}/test-fit
```

The test-fit endpoint runs the matcher + LLM reply drafter against the signal **without persisting anything**. It's safe to call repeatedly — same inputs return cached LLM output within the 60s TTL. Use it to preview what the seller agent would propose before calling `/respond`.

Body is optional; pass `overrides` to nudge the matcher:

```json
{
  "overrides": {
    "includeProductIds": ["pmp-climate-display"],
    "excludeProductIds": ["video-only-bundle"],
    "cpmOverrides": { "pmp-climate-display": 5.5 },
    "notesOverride": "Hi — we can do this at €5.50 CPM. Confirming creative specs by EOD."
  }
}
```

- `includeProductIds` **force-includes** the listed productIds in the output even if they fail the channel gate. Layered on top of the natural matched set — does NOT replace it, and force-included rows do NOT count toward `matchScore`. Use when an operator manually picks a catalogue product the matcher didn't surface.
- Pair this with `GET /api/v2/storefront/storefronts/{storefrontId}/products` to discover the full catalogue (every productId reachable from the storefront's active sales agents).

Response shape:

```json
{
  "matchScore": 88,
  "verdict": "AUTO-QUOTE",
  "factors": [
    { "id": "channel", "label": "Channel coverage", "weight": 25, "score": 25, "detail": "Covers display & native" }
  ],
  "products": [
    {
      "productId": "pmp-climate-display",
      "name": "Climate Vertical · Display & Native PMP",
      "channel": "display",
      "cpm": 5.2,
      "currency": "EUR",
      "estimatedImpressions": 9615385,
      "fitScore": 92,
      "factors": [ /* same shape as the signal-level factors but per product */ ]
    }
  ],
  "productsConsidered": 12,
  "draftReply": { "text": "Hi GroupM NL — strong fit…", "source": "gemini" },
  "headlineCpm": 5.2,
  "headlineCurrency": "EUR"
}
```

- `verdict` is `AUTO-QUOTE` (score ≥ 70), `REVIEW` (40–69), or `PASS` (< 40).
- `draftReply.source` is `gemini` (LLM-drafted), `template` (LLM unavailable, deterministic fallback), or `human` (caller supplied `notesOverride`).
- `headlineCpm` / `headlineCurrency` are the server's recommended quote rate. Both are `null` when no priced product matched AND the brief carried no `priceExpect` — in that case do NOT call `/respond` with a fabricated CPM. Either submit a `cpmOverrides` value and re-run, or `/respond` with `kind: 'DECLINE'`.
- The endpoint has zero side effects. Buyer never sees it.

#### List the storefront's catalogue (for the Adjust picker)

```http
GET /api/v2/storefront/storefronts/{storefrontId}/products
```

Returns every product reachable from the storefront's active sales agents in a flat, lightweight shape: `productId`, `name`, `description`, `publisherDomain`, `channel`, `cpm`, `currency`. No factors, no fitScore — this isn't the matcher; it's the universe of products you can hand to `includeProductIds` in `/test-fit` to force-include them in a quote.

```json
{
  "items": [
    { "productId": "pmp-climate-display", "name": "Climate Vertical · Display & Native PMP", "channel": "display", "cpm": 5.2, "currency": "EUR", "publisherDomain": "nrc.nl", "description": null }
  ]
}
```

### Brief history (intelligence-runs)

This is the **settled-outcome record** of buyer briefs the storefront agent has already auto-answered — distinct from Demand Signals above, which is the *live action queue* a seller still works (QUOTE/CLARIFY/DECLINE/BOOK). Use brief history to answer "how did my agent do?" / "show me the briefs that booked"; use demand signals to act on open demand.

```http
GET /api/v2/storefront/intelligence-runs
```

Scoped to the caller's own storefront by auth context — there is **no `{storefrontId}`** in the path. It follows the v2 paginated-list contract: the response `data` is a **bare array** of runs and the count is in `meta.pagination.total` (not a `{ items, total }` object).

Query parameters (all optional):
- `buyingMode` — `brief` (the demand inbox; excludes wholesale/refine runs) / `wholesale` / `refine`
- `disposition` — `responded` / `declined_fit` (no matching inventory) / `declined_policy` (acceptance-policy violation). (`not_live` is reserved in the schema but never emitted today — filtering by it returns zero rows.)
- `commercialResult` — `booked` (forwarded upstream or delivering — a confirmed commitment) / `pending` (awaiting the operator's own acceptance-policy approval, still rejectable) / `rejected` (the storefront rejected the buy). Runs with no attributed media buy yet match no value.
- `q` — case-insensitive substring over the brief text
- `view` — `full` (default; every field) or `summary` (omits the heavy LLM payload columns the inbox doesn't render — prefer it for listing)
- `skip`, `take` (max 100, default 25)

A single run is at `GET /api/v2/storefront/intelligence-runs/{id}` (always full). The decline `reason` on a run is operator-only — never echo it to a buyer.

### Seller analytics (how am I doing)

The aggregated "how is my agent selling" rollup over recent brief history — use it for questions like "how am I doing?" or "which negotiation posture is winning deals?", not for inspecting individual briefs (that's intelligence-runs above).

Named operation: `get_seller_analytics` (no path params — scoped to the caller's own storefront by auth context).

```http
GET /api/v2/storefront/seller-analytics
```

Query parameters:
- `limit` — how many of the most recent intelligence runs to aggregate over (1–200, default 100)

The response is the full seller-analytics payload: `generatedAt`, the aggregation `window`, summary KPIs, commercial outcomes (booked budget, delivery), per-posture conversion (`postureConversion`: run count, booked count, win rate, and booked budget per negotiation posture), recommendation adherence, learned defaults, and recent runs. This is the same payload the seller Dashboard widget renders. Runs where no composition ran are excluded from the conversion denominators. Returns `NOT_FOUND` if no storefront exists for the calling operator.

### Approval queues (creative review + media-buy approval)

The two operator work queues for storefronts that gate buyer submissions before forwarding upstream. A storefront enrolls in creative review when `capabilities.offersCreativeReview = true`, and in media-buy approval when `capabilities.offersCampaignApproval = true` — until then these lists are empty. Each queue exposes a list, an advisory AI evaluation (decision-support only, records nothing), and an operator decide; media-buy approvals add a retry-forward for a decided buy whose upstream forward did not complete. All are scoped to the caller's own storefront by auth context — you never pass a `storefrontId`.

> The approval-gate dials (auto-approve on/off, per-buyer carve-outs) are POLICY controls, not queue actions — they live in the acceptance-policy / buyer-auto-approvals surfaces, not here.

#### List creative reviews

Named operation: `list_creative_reviews`.

```http
GET /api/v2/storefront/creative-reviews
```

Query parameters (all optional):
- `status` — `pending` (default) / `approved` / `rejected` / `revoked`

Returns a bare array of creative-review rows, newest first. Each row carries `id`, `creativeId`, `mediaBuyId` (nullable), `buyerCustomerId`, the verbatim `submittedPayload` (buyer webhook credentials stripped), `status`, `reviewedBy`/`reviewedAt`/`reviewerNotes` (null while pending), `createdAt`, and `updatedAt`.

#### Evaluate a creative review (advisory)

Named operation: `evaluate_creative_review`.

```http
POST /api/v2/storefront/creative-reviews/{creativeId}/evaluate
```

- `pathParams`: `{ "creativeId": "cr_abc123" }` (the buyer-supplied creative id from the list row).
- `body` (all fields optional; send `{}` for a default evaluation):
  - `acceptance_policy_text` — policy text to evaluate the creative against
  - `expected_brand` — the brand the creative should represent (inferred from the payload when omitted)
  - `questions` — up to 8 specific questions for the evaluator to answer

Returns an advisory verdict: `status` (`pass`/`warn`/`fail`), `recommendation` (`approve`/`manual_review`/`reject`), `policyDecision`, `summary`, `checks[]`, and `answers[]`. **It never records a decision** — call decide separately.

#### Decide a creative review

Named operation: `decide_creative_review`.

```http
POST /api/v2/storefront/creative-reviews/{creativeId}/decide
```

- `pathParams`: `{ "creativeId": "cr_abc123" }`
- `body`:
  - `status` (required) — `approved` or `rejected` (`revoked` is a separate gesture, not valid here)
  - `reviewer_notes` (optional, **snake_case**) — reason surfaced back to the buyer

Only valid on a `pending` row. Approved creatives are forwarded to the underlying sales agent(s) by the wire layer. Returns the updated review row.

#### List media-buy approvals

Named operation: `list_media_buy_approvals`.

```http
GET /api/v2/storefront/media-buy-approvals
```

Query parameters (all optional):
- `status` — `pending` (default) / `approved` / `rejected` / `revoked`

Returns a bare array of approval-queue rows, newest first. Each row carries `id`, `kind` (`create` or `update`), `mediaBuyId`, `buyerCustomerId`, the raw `submittedPayload` (buyer webhook credentials stripped), `status`, `reviewedBy`/`reviewedAt`/`reviewerNotes`, `forwardedAt` (null until the approved buy is forwarded upstream — an approved row with a null `forwardedAt` has not been forwarded yet), `createdAt`, and `updatedAt`.

#### Evaluate a media-buy approval (advisory)

Named operation: `evaluate_media_buy_approval`. **No request body.**

```http
POST /api/v2/storefront/media-buy-approvals/{mediaBuyId}/evaluate
```

- `pathParams`: `{ "mediaBuyId": "mb_abc123" }`

Returns a policy classification: `policyDecision` (`definitely_on_policy` / `needs_human_approval` / `definitely_not_on_policy`), `recommendation` (`auto_approve` / `escalate_to_human` / `auto_reject`), `rejectionReason` (nullable), `summary`, and `checks[]`. Decision-support only — it records nothing.

#### Decide a media-buy approval

Named operation: `decide_media_buy_approval`.

```http
POST /api/v2/storefront/media-buy-approvals/{mediaBuyId}/decide
```

- `pathParams`: `{ "mediaBuyId": "mb_abc123" }`
- `body`:
  - `status` (required) — `approved` or `rejected`
  - `reviewerNotes` (optional, **camelCase** — note this differs from the creative-review decide, which uses `reviewer_notes`) — surfaced back to the buyer alongside the status

Only valid on a `pending` row (double-decide is rejected). On approve, the storefront forwards the buy upstream; a partial or failed forward still records the approval, so check the returned `forwardOutcome` — anything other than `all_completed` means the buy is not fully live and may need a retry-forward. Returns the updated approval row.

#### Retry forwarding a media-buy approval

Named operation: `retry_forward_media_buy_approval`. **No request body.**

```http
POST /api/v2/storefront/media-buy-approvals/{mediaBuyId}/retry-forward
```

- `pathParams`: `{ "mediaBuyId": "mb_abc123" }`

Re-attempts the upstream forward for an already-approved buy whose previous forward did not complete (a transient, non-terminalized failure). Idempotent — the storefront reuses the same idempotency key, so a source that already received the buy is not double-booked. Returns the approval row with its refreshed forwarding state. Structural/terminalized failures are not retryable here and must be escalated.

### Own product catalog (approval matching)

Named operation: `list_own_storefront_products` (no path params — scoped to the caller's own storefront by auth context).

```http
GET /api/v2/storefront/products
```

Returns `{ "data": [ ... ] }` — the storefront's products aggregated across all active ESA connections. Each product carries `productId`, `name`, `description` (nullable), `status`, `creativeFormatIds[]`, and `adUnitNames[]`. The approvals widget uses this to show which products a pending media buy is eligible for. This is distinct from `list_storefront_products` (the demand-signals catalogue keyed by an explicit `{storefrontId}`).

### Sandbox test runs (agent E2E diagnostics)

Named operation: `list_agent_test_runs` (no path params — scoped to the caller's own storefront by auth context).

```http
GET /api/v2/storefront/test-runs
```

Query parameters (all optional):
- `take` — max rows to return, 1-20 (default 5)
- `storefrontId` — narrow the listing to a single storefront (for multi-storefront sellers); omit to get every run for the caller's customer

Returns `{ "runs": [ ... ] }` — the durable Murph sandbox test-run history: each row is one end-to-end diagnostic run (tool name, status, stage, sandbox flag, storefront/advertiser/campaign/discovery ids where applicable, summary, artifacts, diagnostics, and the per-step timeline). This is the same data as the `GET /v2/murph/test-runs` route the Murph chat surface uses; this operation exposes it on the storefront surface so the `test-runs` MCP-app widget can hydrate on external MCP-UI hosts, which cannot reach `/murph/*` paths.

---

## Key Concepts

### Partners, Storefronts, and Agents

- **Agents** have a `type` (SALES, SIGNAL, CREATIVE, or OUTCOME)
- Buyers register agent credentials (`POST /api/v2/buyer/sales-agents/{agentId}/accountCredentials`) then link accounts to advertisers (`POST /api/v2/buyer/sales-agents/{agentId}/accounts`)

### Authentication Types

| Type | Auth Object Format | Notes |
|------|-------------------|-------|
| `API_KEY` | `{ "type": "bearer", "token": "..." }` | Most common. Simple token-based auth. |
| `NO_AUTH` | `{}` | Agent endpoint is open. |
| `JWT` | `{ "type": "jwt", "privateKey": "...", ... }` | JSON Web Token with private key signing. |
| `OAUTH` | Managed by OAuth flow | Credentials obtained through OAuth redirect flow. |

## Notifications

Notifications are events about resources you manage — agents registering, status changes, etc. They follow a `resource.action` taxonomy (e.g., `salesagent.registered`, `signalsagent.signal_activated`).

### List Notifications

**Operation:** `list_notifications`
```http
GET /api/v2/notifications?unreadOnly=true&limit=20&offset=0
```
**As operation:**
```json
{ "operation": "list_notifications", "params": { "unreadOnly": "true", "limit": "20", "offset": "0" } }
```

**Query Parameters (all optional):**
- `unreadOnly` (`true`/`false`): Show only unread notifications
- `brandAgentId` (number): Filter by brand agent
- `types` (comma-separated): Filter by event types (e.g., `salesagent.registered,signalsagent.updated`)
- `limit` (number): Results per page (default: 50, max: 100)
- `offset` (number): Pagination offset

### Mark Notification as Read

**Operation:** `mark_notification_read` with `pathParams: { "id": "<notificationId>" }`
```http
POST /api/v2/notifications/{notificationId}/read
```

### Mark Notification as Acknowledged

**Operation:** `mark_notification_acknowledged` with `pathParams: { "id": "<notificationId>" }`
```http
POST /api/v2/notifications/{notificationId}/acknowledge
```

### Mark All Notifications as Read

**Operation:** `mark_all_notifications_read`
```http
POST /api/v2/notifications/read-all
```

**Optional body:**
```json
{ "brandAgentId": 123 }
```

### Proactive Notification Setup

Unread notifications are automatically included in `help` and `ask_about_capability` tool responses. To ensure your AI agent surfaces them to users at the start of every session, add the following to your client configuration:

- **Claude Desktop**: Create a Project and add to the project instructions: `When using Scope3 tools, always start by calling the help tool. The response includes unread notifications — summarize those for the user before answering their question.`
- **Claude Code**: Add the same instruction to your `CLAUDE.md` or project instructions.
- **API / Custom Agent**: Add it to your system prompt.
- **ChatGPT Custom GPT**: Add it to your Custom GPT's instructions.

---

## Error Handling

### REST API Error Format

All REST error responses use a standard envelope:

```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable error description",
    "field": "start_date",
    "details": {}
  }
}
```

- `code` — machine-readable error code (see table below)
- `message` — human-readable description
- `field` — (optional) the specific field that caused the error
- `details` — (optional) additional context

### MCP Tool Error Format

Tool errors return `isError: true` with a structured error object in `structuredContent` and a human-readable message in `content`:

```json
{
  "content": [{ "type": "text", "text": "Agent not found" }],
  "structuredContent": {
    "code": "NOT_FOUND",
    "message": "Agent not found"
  },
  "isError": true
}
```

- `code` — machine-readable error code (see table below)
- `message` — human-readable description
- `field` — (optional) field path that caused the error
- `suggestion` — (optional) suggested fix

### Error Codes

| Code | HTTP Status | Resolution |
|------|-------------|------------|
| `VALIDATION_ERROR` | 400 | Check request body against schema |
| `UNAUTHORIZED` | 401 | Verify API key/auth |
| `ACCESS_DENIED` | 403 | Check permissions |
| `NOT_FOUND` | 404 | Verify resource ID exists |
| `CONFLICT` | 409 | Resource already exists |
| `RATE_LIMITED` | 429 | Wait and retry |
| `INVALID_STATE` | 400 | Operation not allowed in current state |
| `INTERNAL_ERROR` | 500 | Contact support |

## Common Mistakes to Avoid

1. **Asking the user for OAuth credentials** - NEVER ask for client_id, client_secret, authorization URL, token URL, or scopes for OAUTH agents. The platform discovers everything automatically. Just set `authenticationType: "OAUTH"` with no `auth` field.
2. **Registering credentials/accounts on the Storefront API** — Agent credential registration and account linking are on the Buyer API (`POST /api/v2/buyer/sales-agents/{agentId}/accountCredentials` and `POST /api/v2/buyer/sales-agents/{agentId}/accounts`), not the Storefront API.
3. **Missing `advertiserId` when linking an account** — Buyers must specify which advertiser to link the account for.
4. **Missing `auth` for credentialed agents** - Initial credentials are required for testing and validation when `authenticationType` is `API_KEY`, `JWT`, or `BASIC_AUTH`
5. **No `redirectUri` needed for OAuth registration** - The platform automatically uses its own callback URL
6. **Using GET for OAuth authorize** - The OAuth authorize endpoints are POST, not GET
7. **Calling endpoints without auth** - All endpoints require authentication except `GET /oauth/callback`.
8. **Wrong base path** - Storefront API endpoints use the `/api/v2/storefront/` prefix; shared endpoints (notifications, accounts, accept-tos, communication-channel, service-tokens) use `/api/v2/` with no `/storefront/` segment.
9. **Registering accounts on PENDING agents** - Agent must be ACTIVE before accounts can be registered
10. **Activating a storefront without checking readiness** — `PUT /storefront { transacting: true }` returns 400 if readiness status is `"blocked"`. Always check readiness first with `GET /storefront/readiness` and resolve all blocking checks before activating.
11. **Trying to create a storefront** — Storefronts are automatically created when a seller is created. `POST /storefront` is idempotent and returns the existing storefront if one is already provisioned.
12. **Polling for OAuth completion** — There is no OAuth status endpoint to poll. After receiving `oauth.authorizationUrl` from `POST /inventory-sources`, hand the URL to a human and just retry your next call (e.g. `PUT /storefront { transacting: true }`). Incomplete OAuth surfaces as `400 INVALID_STATE` with `readiness: blocked`; on success the same call returns 200.
