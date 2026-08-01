---
name: scope3-agentic-buyer
version: "2.0.0"
description: Scope3 Agentic Buyer API - AI-powered programmatic advertising
api_base_url: https://api.interchange.io/api/v2/buyer
auth:
  type: bearer
  header: Authorization
  format: "Bearer {token}"
  obtain_url: https://interchange.io/user-api-keys
---

# Scope3 Agentic Buyer API

This API enables AI-powered programmatic advertising with inventory discovery, campaign management, and creative orchestration.

**Important**: Use the most specific MCP tool first. Session-level actions such as customer switching use dedicated tools like `customer_switch`; do not route them through `api_call`. For buyer REST operations, use `api_call` with the `operation` field — every call must supply a named operation; there is no raw `method` + `endpoint` mode.

## ⚠️ CRITICAL: Presentation Rules

**Tool responses return JSON data.** For most endpoints (advertisers, sales agents, campaigns, etc.), YOU are responsible for presenting the data clearly in your message. Follow the Display Requirements for each endpoint — they tell you exactly what fields to show and how to structure the output. Never summarize into vague prose — always show the specific data points listed in the display requirements for each item.

**Exception: Product discovery and reporting endpoints render interactive UI components.** When those tools return a UI, display it as-is — do not generate your own competing visualization.

**Every value you present must trace to a tool response or user input.** When you show the user a status, ID, count, date, or any other data point, it must come from a `api_call` response, an `ask_about_capability` response, or something the user told you in this conversation. If you are unsure whether a value is still current (for example, a status that may have changed since the last fetch), qualify it — "last known value from `get_advertiser`" — rather than stating it as a fresh fact. Do not restate values "from memory" without re-fetching.

## ⚠️ CRITICAL: Show Operational Details, Not Just Names

When listing entities (advertisers, sales agents, storefronts), you MUST display the operational fields from the response — account status, credential requirements, linked accounts, sandbox status — not just names. The `ask_about_capability` response tells you which fields to show for each operation.

**Common mistake:** Listing advertisers or sales agents and showing only names. This is WRONG. Show account status, credential requirements, linked accounts, and other operational details for each item.

## ⚠️ CRITICAL: Exact Field Names Required

**DO NOT GUESS FIELD NAMES.** Use these exact camelCase names:

| Field | Type | Notes |
|-------|------|-------|
| `advertiserId` | string | NOT `advertiser_id` |
| `brand` | string | Required on advertiser create (e.g., `"nike.com"`) |
| `saveBrand` | boolean | Optional on advertiser create/update. Set `true` only after reviewing enrichment or when the user confirms they want to save the brand to the registry |
| `sandbox` | boolean | Optional on advertiser create. When `true`, all ADCP operations use sandbox accounts (no real spend). Immutable after creation |
| `optimizationApplyMode` | string | Optional. `"AUTO"` or `"MANUAL"` (default `"MANUAL"`). Do not ask for this during advertiser setup; only set it when the user explicitly asks to configure approval mode or volunteers a clear preference. |
| `flightDates` | **object** | NOT `startDate`/`endDate` at root level |
| `flightDates.startDate` | string | ISO 8601: `"2026-02-05T00:00:00Z"` |
| `flightDates.endDate` | string | ISO 8601: `"2026-02-10T23:59:59Z"` |
| `budget` | **object** | NOT a number |
| `budget.total` | number | e.g., `1000` |
| `budget.currency` | string | Three-letter ISO code. Every campaign is created in the advertiser's currency; an advertiser is single-currency. Pass the advertiser's currency here or omit it — never a different currency, the server rejects mismatches (400, field `budget.currency`). Preserve the user's stated currency (`€`/`euro`/`euros`/`Euro` -> `"EUR"`, `£`/`pound`/`pounds`/`sterling` -> `"GBP"`, `USD`/`US dollar`/`US$` -> `"USD"`); do not let a non-USD request fall back to USD. Ask if ambiguous, including standalone `$`, `dollar`, `dollars`, or `dólares`. |
| `constraints` | object | Optional. Channel filter plus AdCP-shaped targeting overlay (geo, language, device). Targeting fields flow into every media-buy package; include lists intersect with package targeting, exclude lists union. |
| `constraints.channels` | array | e.g., `["display"]`, `["ctv"]` |
| `constraints.geo_countries` / `geo_countries_exclude` | string[] | ISO 3166-1 alpha-2 codes. Replaces the deprecated `countries` field. |
| `constraints.geo_regions` / `geo_regions_exclude` | string[] | ISO 3166-2 subdivision codes (e.g. `"US-CA"`). |
| `constraints.geo_metros` / `geo_metros_exclude` | `{system, values}[]` | Metro targeting, e.g. `{ system: "nielsen_dma", values: ["501"] }`. Always use numeric string codes. |
| `constraints.geo_postal_areas` / `geo_postal_areas_exclude` | `{country, system, values}[]` | Postal-area targeting. Use ISO 3166-1 alpha-2 `country` plus country-local `system`, e.g. `{ country: "NL", system: "postal_code", values: ["1011"] }`. Legacy `{system, values}` is accepted only for deprecated country-fused systems. |
| `constraints.language` | string[] | ISO 639-1 codes (e.g. `["en"]`). |
| `constraints.device_type` / `device_type_exclude` | string[] | `desktop \| mobile \| tablet \| ctv \| dooh`. |
| `constraints.device_platform` | string[] | `ios \| android \| roku_os \| ...` |
| `performanceConfig` | object | Optional. Contains `optimizationGoals` array. Each goal has `kind` (`"event"` or `"metric"`). Event goals have `eventSources` array + optional `target`. Metric goals have `metric` + optional `target`. |

## Nielsen DMA Resolution

When using `geo_metros` with `system: "nielsen_dma"`, always send numeric string codes — never market names. If a buyer provides a market name, casual label, or spreadsheet/export row, call `buyer_api_call` with `operation: "resolve_targeting_dimension"`, `pathParams: { "system": "nielsen_dma" }`, and `params: { "q": "<buyer text>" }` for each label. Do not list the full geo-metro table to do fuzzy matching yourself unless the resolver returns no candidates or an ambiguous result.

If the response has `ambiguous: true`, multiple high-confidence candidates, or no candidates, ask a concise clarification before creating/updating the campaign. If the buyer says "LA DMA", resolve it within `nielsen_dma` and use the returned Los Angeles DMA code.

When you resolve a market name to a DMA code, send only the code in `constraints.geo_metros`, for example:

```json
{
  "constraints": {
    "geo_metros": [{ "system": "nielsen_dma", "values": ["532"] }]
  }
}
```

For browsing display labels, use `operation: "list_targeting_dimension_values"` with `pathParams: { "system": "nielsen_dma" }`. For campaign display, pass `params: { "fields": "geo_metro_names" }` to `list_campaigns` or `get_campaign` when human-readable metro labels are needed. Included metro labels return in `geo_metro_names`; excluded metro labels return in `geo_metro_names_exclude`. The returned labels are derived from the same local English label table. If a code has no label, show the code and ask for clarification instead of guessing a DMA name.

## ⚠️ CRITICAL: Planning Briefs vs Product Discovery — Pick The Right Flow

**Planning briefs (a.k.a. prospective briefs) and product discovery are different flows. Do NOT default to product discovery when the buyer is sharing a planning brief with publishers.**

Buyer side: planning briefs. Seller side (separate API): demand signals — same row in the DB, different vantage point. Never say "RFP" to the buyer.

Two distinct workflows, two different operations:

| The buyer wants to… | Flow | Operation |
|---|---|---|
| Browse / select inventory to attach to a campaign right now | **Product Discovery** | `discover_products` |
| Send a planning / prospective brief to publishers and gather quotes or clarifications | **Planning Briefs** | `share_planning_brief` |

**Use the Planning Brief flow when the buyer language sounds like:**
- "we're planning a [launch / campaign / push]…"
- "I want to reach some storefronts to see if they're around"
- "send this brief to [publishers]"
- "share a prospective brief with [publishers]"
- "see if [publisher X / Y] is interested"
- "test fit with these publishers"
- "planning brief", "prospective brief", "forward signal", "planning note"
- the user is describing intent at a brand/campaign level — audience, geo, dates, budget, KPI — and is asking which publishers can meet it, NOT which products to buy
- the user names specific publishers/storefronts and asks whether to engage them

**Use Product Discovery when the buyer language sounds like:**
- "find products for my campaign"
- "what inventory matches this brief — I want to attach it"
- "show me products on [storefront X]"
- the user is already in the campaign-build flow and has chosen the "browse products" path (see Campaigns → Required flow)

**If unsure, ASK.** Don't silently jump to discovery just because a brief is involved. A one-line clarifier — "Do you want to (a) share this as a planning brief with publishers and collect their fit/quotes, or (b) browse products to add to a campaign?" — is correct.

See **Planning Briefs (Prospective Briefs)** section below for the full flow.

## ⚠️ CRITICAL: Always Send the ENTIRE Client Brief

**When sending a `brief` field to ANY endpoint, you MUST include the COMPLETE brief text the client provided — word for word, in full.** Do NOT summarize, truncate, paraphrase, or shorten the brief under any circumstances.

The brief is used by sales agents and the discovery system to match relevant inventory. Sending a partial or summarized brief degrades match quality and loses important context. Even if the brief is long, always pass it through in its entirety.

**Rules:**
- **Always include the brief** — if the client provided a brief, you MUST send it in every API call that accepts a `brief` field. Never omit it or leave it out.
- **Copy the full brief verbatim** — include every detail the client provided
- **Never summarize** — "Premium CTV for tech enthusiasts" is NOT an acceptable substitute for a multi-paragraph brief
- **Never truncate** — if the client gave you 500 words, send all 500 words
- **Applies everywhere `brief` is used** — product discovery, campaign creation, and any other endpoint that accepts a brief

## ⚠️ CRITICAL: Never Fabricate User Data

**Before making any API call that creates or modifies a resource, you MUST have explicit user input for all required fields.** Do NOT invent, guess, or auto-fill values the user hasn't provided.

**Rules:**
- **NEVER fabricate values** for required fields. If the user hasn't provided a value, ask for it.
- **Read-only calls are fine** — you can freely call GET endpoints to fetch data and present it to the user.
- **Confirm before mutating** — Before any POST, PUT, or DELETE call, verify you have user-provided (or user-confirmed) values for all required fields.
- **Inferring is OK when obvious** — If the user says "optimize for purchases with a 4x ROAS target", you can infer an event goal with `eventType: "purchase"` and `target: { kind: "per_ad_spend", value: 4.0 }`. But if intent is ambiguous, ask.
- **Never make up IDs** — IDs (advertiserId, discoveryId, campaignId, etc.) must come from previous API responses or the user. Never generate them.
- **Account IDs for linking MUST come from the correct discovery surface** — For an official adapter connection, use only an `accountId` returned by `list_storefront_connection_accounts` or `list_storefront_connection_account_mappings` for that `connectionId`. For a legacy or third-party external AdCP inventory source, use only an `accountId` returned by `list_available_accounts`. Never use an account ID or name supplied only in conversation. If the account is absent from the appropriate response, tell the user it was not found; never pretend it was linked.
- **Only use what's documented** — Do NOT invent endpoints, fields, query parameters, or enum values that are not explicitly listed in this skill document. If you're unsure whether something exists, check this document first. If it's not here, don't use it.

## Before Every api_call: Verify Each Field Has a Source

Before invoking `api_call`, mentally walk through every field in `body`, `pathParams`, and `params` and name its source:

- "`advertiserId`: from the `list_advertisers` response above" ✅
- "`brief`: verbatim from the user's message" ✅
- "`budget.total`: user said $5,000" ✅
- "`flightDates.startDate`: I assumed next Monday" ❌ — STOP, ask the user
- "`optimizationGoals[0].metric`: typically these APIs use `CTR`" ❌ — STOP, call `ask_about_capability`

If any field's source is "I assumed…", "it's probably…", "typically these are…", or "the docs usually say…" — do NOT send the call. Either ask the user for the value, or call `ask_about_capability` to find the correct schema.

This check is cheap; a failed or wrong-data API call is expensive.

---

## Notifications

The `help` and `ask_about_capability` tools include unread notifications in their responses. When a response contains a "Unread Notifications" section, summarize those notifications for the user before answering their question.

Notifications can be listed, marked as read, or acknowledged via the `/api/v2/notifications` endpoints — see the Notifications section below for details.

**Setup:** To receive notifications proactively at the start of every session, add this to your Claude Desktop Project instructions, CLAUDE.md, or system prompt: `When using Scope3 tools, always start by calling the help tool. The response includes unread notifications — summarize those for the user before answering their question.`

## Plan & Billing (`open_plan_and_billing` / `get_billing_account`)

Two read-only tools cover the organization's consolidated commercial-account
view — plan/contract status, effective pricing, intelligence usage,
credit/prepay standing and balance, agreements, and the single next action (if
any) needed to become or remain paid.

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

## Adding a payment method (`add_payment_authority`)

When the organization needs a payment method — `get_billing_account` shows no
verified payment method, or billing setup is blocking a paid action — use the
`add_payment_authority` bounded Task. Card data NEVER passes through you or
this API: the Task produces a secure link the organization's own cardholder
opens in a browser (no Scope3 sign-in needed) to enter the card directly with
the payment processor.

**The three-step contract:**

1. `add_payment_authority` with `action: "request"` — stages the request and
   returns a single-use `confirmationToken` (expires in 5 minutes). Nothing
   is created yet.
2. `add_payment_authority` with `action: "confirm"` and that
   `confirmationToken` — issues the capture link:
   `{ method: "capture_link", url, status: "pending", expiresAt }`. The link
   is valid for 30 minutes and serves one verified card capture. SAVE THE
   URL — it is returned exactly once and cannot be re-fetched.
3. **Hand the `url` to the human cardholder** (paste it in the chat, an
   email, wherever the user directed) and tell them: open it, enter the
   card, done — no account or sign-in is needed.

**Polling etiquette:** after handing off the link, poll
`add_payment_authority` with `action: "status"` to learn the outcome:
`pending` → `opened` (page visited) → `verified` (card confirmed) or
`expired`. Poll no more than every 15-30 seconds — the human step usually
takes minutes. On `verified`, the organization has an active payment
authority (it appears in `get_billing_account` → `payment`). On `expired`,
start over with a fresh `request`.

Requires org-admin access (a human admin session or an org-scoped service
token; advertiser-scoped tokens are refused). `method` is always
`"capture_link"` today; switch on the field rather than assuming it — a
future agent-held payment token method would be an additive value.

## Quick Start

1. **Use the most specific MCP tool first**: For session state, use dedicated tools such as `customer_switch`; do not route those actions through `api_call`.
2. **Use `ask_about_capability` before REST work when needed**: Ask about the user's request to learn the correct workflow, endpoints, and field names.
3. **Use `api_call` with the `operation` field for REST operations**: **Always use the `operation` enum** — it prevents hallucinated endpoints and wrong HTTP methods.
4. **Use `ask_murph` when the user wants explanation, not execution**: see "When to call Murph" below.
5. **Authentication**: Handled automatically by the MCP session

### `api_call` parameters

- `operation` (string): Named operation — each endpoint section below shows the operation name
- `pathParams` (object): Path parameters (e.g. `{ "advertiserId": "abc-123" }`)
- `body` (object): Request body for POST/PUT operations
- `params` (object): Query parameters as key-value pairs

## When to call Murph (`ask_murph`)

Murph is your AI guide to Interchange. It's mounted on this server so you don't have to switch MCP servers when a user pivots from "do X" to "explain X."

**Route to Murph when the user asks**:
- "How do I…" / "What's the right way to…" — onboarding, workflow, "where do I start"
- "Why is my X in state Y?" — campaign paused, sync not running, sales agent not responding
- "What changed recently?" — they noticed something different
- "This looks broken" / "I think there's a bug" — Murph files a report for the Scope3 team
- General help where the user doesn't know which tool / endpoint applies

**Stay on `api_call` when the user asks for**:
- A specific CRUD action (`create advertiser`, `pause campaign`, `add product to campaign`, etc.)
- A specific data fetch (`list my campaigns`, `show me reporting`)
- Any mutation — Murph is read-only

**Don't double-route.** If the user's intent is unambiguously mutate-this, just call `api_call` and tell the user what you did. Don't ask Murph first; that wastes a Claude round-trip and confuses the user.

**Murph parameters:**
- `prompt` (string, required): The user's question, verbatim. Plain English.
- `conversationUid` (string, optional): Pass back what Murph returned on the previous turn to continue a thread. Omit on first call.

**Murph caveats** (foundation mode):
- Murph answers from its own knowledge of Interchange. Live-state inspection and full docs citations are follow-up work — Murph will tell you when it's guessing vs sourced.
- Murph CAN record an internal report (`escalated: true` in the response). Do NOT mention any ticket ID or internal tracking reference — tell the user the Scope3 team has been notified.
- Murph is scoped to the caller's customer. One Murph conversation == one customer; don't reuse `conversationUid` across `customer_switch`.

---

## Planning Briefs (Prospective Briefs)

> Behind the `demand-supply-signals` feature flag. If `share_planning_brief` returns `FEATURE_NOT_ENABLED`, the feature is not turned on for this customer — tell the user planning briefs aren't available yet and fall back to discussing inventory via `list_storefronts` only, without pretending to share a brief.
>
> If `share_planning_brief` returns `CUSTOMER_SCOPE_REQUIRED`, call `customer_switch` first with one of the `customerId`s in `details.availableCustomers`, then retry the original request unchanged. Don't guess at the cause — this error is always about session scoping, never about the flag, schema, or permissions.

This is the buyer workflow for **sharing a planning / prospective brief with publishers** — passing along a brand-level brief (audience, geo, dates, budget, KPI) to one or more publisher storefronts so they can react with fit signals, quotes, clarifications, declines, or booked deals. It is NOT the same as product discovery. See the "Planning Briefs vs Product Discovery" CRITICAL section above for which flow to use.

Buyer-facing name is **planning brief** or **prospective brief**. Don't say "RFP" or "demand signal" to the buyer (the latter is the seller-side framing).

### When to use this flow

Trigger phrases (non-exhaustive):
- "we're planning a [campaign / launch / push]"
- "I want to reach some [publishers / storefronts]"
- "send this brief to [publisher names]"
- "share a planning / prospective brief"
- "see if [publisher X] is interested / has fit"
- "test fit", "forward signal", "planning note"
- the user describes audience + geo + dates + budget at a brand level and asks which publishers can carry it

### The 3-step flow

Each step is one assistant turn. Do not collapse them — the buyer needs to see and confirm the matched publisher set before the brief is shared with anyone.

**Step 1 — Intake (parse the brief).** When the user gives you a planning brief, extract these fields from their message. Do NOT fabricate values; if a required field is missing, ASK. Echo back what you extracted before moving on (e.g. "Got it. Parsing your brief and looking for matching publishers across Interchange…").

Required (must be present before dispatch):
- `buyerName` (string) — the buyer's display name. **Buyer-name resolution rule — ASK, NEVER GUESS, NEVER INFER (no exceptions):**
  - The ONLY two legal sources for `buyerName` are: (a) the `advertiserName` the user just named in THIS brief, or (b) a name the user explicitly typed in response to your direct question. There is no third source.
  - If the user named an advertiser in this brief (e.g. "for my advertiser Marabou"), set `buyerName = advertiserName` verbatim. The advertiser IS the buyer. Do not invent a separate agency name and do not append/prepend anything.
  - If the user did NOT name an advertiser in this brief, STOP and ASK: "Who is the buyer for this brief?" Wait for their answer before calling `share_planning_brief`.
  - **NEVER infer `buyerName` from:** prior turns in this session, a previous session's brief, an earlier tool call's arguments, the authenticated user, the customer/advertiser name, a notification, a memory, a hunch, or anything else outside this brief. If the source is not (a) or (b) above, it is forbidden — full stop.
  - If you are about to call `share_planning_brief` and cannot point to exactly which words in the user's current brief produced `buyerName`, you are guessing. STOP and ASK before dispatching.
- `advertiserName` (string) — the brand the brief is for (e.g. "Volvo"). If the user did not name one, ASK before continuing. Same rule: never carry over from a previous turn or session — only use what the user named in THIS brief, or what they answer when you ask.
- `audience` (string) — verbatim from the brief (e.g. "climate-conscious Dutch audiences").
- `geo` (string) — verbatim (e.g. "Netherlands", "NL", "US + CA").
- `channels` (array of `display | native | video | audio | ctv | dooh | newsletter | podcast`) — lower-case wire format. Map "display + native" → `["display", "native"]`.
- `startDate` / `endDate` (ISO 8601 dates, `YYYY-MM-DD`) — convert relative dates ("March 1–31") into absolute ISO dates using the current year if not given.
- `statedBudget` (number) — the headline budget the buyer named.
- `currency` (3-letter, uppercase) — derive from the budget symbol or currency word (`€`/`euro`/`euros`/`Euro` -> `EUR`, `£`/`pound`/`pounds`/`sterling` -> `GBP`, `USD`/`US dollar`/`US$` -> `USD`). Ask if ambiguous, including standalone `$`, `dollar`, `dollars`, or `dólares`.

**Resolve `advertiserId` before Step 3.** Whenever the user named an advertiser in the brief, call `list_advertisers` (filter by `name` matching `advertiserName`) and set `advertiserId` to the numeric `advertiserId` from the response on the share body. The wire field is OPTIONAL in the schema, but for any brief that names an advertiser it should be set — without it the planning brief lands on the table with `advertiser_id = NULL`, which orphans the brief from the advertiser's planning history. Do this between Step 1 (intake) and Step 2 (match + select).

- If `list_advertisers` returns exactly one match (case-insensitive substring or exact match on `name`), use that `advertiserId`. Confirm in your echo-back: "Resolved advertiser 'Ematini Inc.' → advertiserId 25."
- If it returns multiple matches, surface the candidates to the user and ask which one — never pick silently.
- If it returns zero matches, ask the user: "I don't see '<advertiserName>' in your advertisers. Want to create it first (`create_advertiser` requires `brand`), or share the brief without an advertiser link?"
- Do not fabricate or guess an advertiserId. Only use values returned by `list_advertisers`.

**Backend safety net (not a substitute for the rule above).** If you omit `advertiserId` but provide `advertiserName`, the backend will attempt one exact case-insensitive lookup on the customer's active BUYER advertisers. On a unique match it auto-fills `advertiserId`; on zero or multiple matches it leaves `NULL` and logs the miss. Treat this as defense-in-depth — do not rely on it. Always resolve client-side so the user sees the resolution and can disambiguate.

Optional but pass through if the brief mentions them:
- `campaignName` (string) — e.g. "Volvo EX30 Q1".
- `primaryKpi` — `{ raw: "<verbatim KPI string>", parsed?: { kpi: "attention_seconds" | "viewability_pct" | "ctr" | "vcr" | "vtr" | "reach" | "frequency_cap" | "sov" | "brand_lift" | "cpa", op: "gt" | "gte" | "eq" | "lte" | "lt", value: number } }`. Always include `raw` verbatim. Only fill `parsed` when the KPI is unambiguous (e.g. "attention > 4s" → `{ kpi: "attention_seconds", op: "gt", value: 4 }`).
- `exclusions` — array of `{ type: "category" | "adjacency" | "competitor" | "domain" | "keyword" | "raw", values: string[], raw?: string }`. "No auto adjacency" → `[{ type: "adjacency", values: ["auto"], raw: "no auto adjacency" }]`.
- `flexibility` — `{ mode: "firm" | "flexible" | "open", days?: number }`. Default to omitting if the brief doesn't talk about flex.
- `priceExpect` (number) — only if the buyer named a CPM expectation.
- `creativeReady` (boolean) — only if the buyer states it.
- `rawPrompt` (string) — the FULL original brief verbatim. Always include this when present.

**Step 2 — Match + select.** Call `list_storefronts` (filter by `name` or visibility as needed) to enumerate available publishers. From the response, group each candidate into one of three buckets:

- `LIVE` — storefront `status === "ACTIVE"` AND it has connected inventory sources (`connectedSourceCount > 0`). Publishers the buyer can share the brief with right now and expect an acknowledgment back.
- `COMING_SOON` — storefront `status === "PENDING"` (or ACTIVE with no connected sources / configuring). The buyer can still register interest; the publisher will see the brief when they come online.
- `NOT_ON_INTERCHANGE` — the user named a publisher (e.g. "NU.nl", "De Volkskrant", "RTL") that does NOT appear in `list_storefronts`. Capture as `externalPublisherName` (+ `externalPublisherDomain` if known). A planning note will be generated for them.

Present the matched set to the buyer with the bucket labels and any heuristic match info you can show (channel coverage, geo overlap from the storefront record). DO NOT share the brief yet. Ask the buyer which publishers to share the planning brief with. The user picks the subset.

**Step 3 — Share + confirm.** Call `share_planning_brief` ONCE with the parsed brief plus a `targets[]` array containing one entry per selected publisher. Describe this to the buyer as "sharing the prospective brief" or "sending the planning brief". Each target must follow exactly one of these shapes — never mix:

```jsonc
// LIVE / COMING_SOON publisher (on Interchange)
{ "storefrontId": 42, "bucket": "LIVE", "matchPct": 86 }

// NOT_ON_INTERCHANGE publisher
{ "externalPublisherName": "NU.nl", "externalPublisherDomain": "nu.nl", "bucket": "NOT_ON_INTERCHANGE" }
```

After the call returns, surface the share summary (one line per publisher with its bucket + status) and tell the buyer: "I'll surface any quotes or clarifying questions here when publishers respond." Do NOT invent or imply a response time (no "within an hour", "shortly", "soon", etc.) — there is no SLA and response cadence varies by publisher. Then END YOUR TURN — do NOT poll responses immediately; the buyer will check back, or you can call `list_planning_brief_responses` next turn.

### Operations

#### Share a planning brief

**Operation:** `share_planning_brief`
```http
POST /api/v2/buyer/planning-briefs
{
  "buyerName": "Acme Agency",
  "advertiserName": "Volvo",
  "advertiserId": 25,  // MUST be set whenever advertiserName resolves to an existing advertiser via list_advertisers
  "campaignName": "Volvo EX30 Q1",
  "audience": "climate-conscious Dutch readers",
  "geo": "NL",
  "channels": ["display", "native"],
  "startDate": "2026-03-01",
  "endDate": "2026-03-31",
  "statedBudget": 50000,
  "currency": "EUR",
  "primaryKpi": {
    "raw": "attention > 4s",
    "parsed": { "kpi": "attention_seconds", "op": "gt", "value": 4 }
  },
  "exclusions": [
    { "type": "adjacency", "values": ["auto"], "raw": "no auto adjacency" }
  ],
  "rawPrompt": "<<< full verbatim brief from the user >>>",
  "targets": [
    { "storefrontId": 42, "bucket": "LIVE", "matchPct": 86 },
    { "storefrontId": 57, "bucket": "COMING_SOON" },
    { "externalPublisherName": "NU.nl", "externalPublisherDomain": "nu.nl", "bucket": "NOT_ON_INTERCHANGE" }
  ]
}
```

**As operation:**
```json
{ "operation": "share_planning_brief", "body": { "buyerName": "...", "advertiserName": "Volvo", "advertiserId": 25, "audience": "...", "geo": "NL", "channels": ["display","native"], "startDate": "2026-03-01", "endDate": "2026-03-31", "statedBudget": 50000, "currency": "EUR", "targets": [{ "storefrontId": 42, "bucket": "LIVE" }] } }
```

**Response (201):** the created signal with its `targets[]` array (each with `dispatchStatus: "QUEUED"`), plus `status: "SEARCHING"`.

**Rules:**
- Each target must have EITHER `storefrontId` (for LIVE / COMING_SOON) OR `externalPublisherName` (for NOT_ON_INTERCHANGE) — never both.
- Up to 200 publishers per planning brief.
- One planning brief shared per turn. After the share call, END YOUR TURN.

#### List my planning briefs

**Operation:** `list_planning_briefs`
```http
GET /api/v2/buyer/planning-briefs?status=SEARCHING&limit=20&offset=0
```

**Query params (all optional):** `status` (`SEARCHING` | `QUOTED` | `BOOKED` | `ABANDONED` | `DECLINED`), `advertiserId`, `limit` (max 100), `offset`.

**Response:** paginated `items[]` of the buyer-facing brief shape (no internal `matchScore`). Refer to each row as a "planning brief" in user-facing copy.

#### Get one planning brief + its publisher targets

**Operation:** `get_planning_brief` (path: `briefId`)

Returns the brief plus its `targets[]` — use this to inspect which publishers it was shared with and the per-publisher status.

#### Check publisher responses

**Operation:** `list_planning_brief_responses` (path: `briefId`)

Returns `items[]` where each row is one publisher's response: `kind` is `QUOTE`, `CLARIFY`, `DECLINE`, or `BOOK`, with a `payload` keyed to the kind:
- `QUOTE` → `{ proposedCpm, currency, products?, notes? }`
- `CLARIFY` → `{ question, gaps? }`
- `DECLINE` → `{ reason }`
- `BOOK` → `{ dealId, notes? }`

Each row also includes a top-level `proposalCode` field (e.g. `"PRP-XK4A29"`). It is set on `QUOTE` rows and is the buyer's redemption handle for the seller's offer — pass it back into `discover_products` to pull the seller's saved snapshot into a campaign (see "Redeeming a QUOTE via `discover_products`" below). `proposalCode` is `null` on `DECLINE`, `CLARIFY`, and `BOOK` rows, and also on the rare `QUOTE` row where operator resolution failed at quote time.

Summarize back to the buyer grouped by publisher. For `CLARIFY` responses, surface the question and ask the buyer how they want to answer. For `QUOTE` responses, mention the proposal code naturally — e.g. "Acme quoted $12 CPM; I can pull their offer into a campaign whenever you're ready" — and offer to redeem it via discovery when the user wants to act on it.

### Redeeming a QUOTE via `discover_products`

When a publisher responds to a planning brief with a `QUOTE`, the server mints a **proposal code** (`PRP-XXXXXXXX`) and returns it on the response row. That code is the buyer's handle to pull the seller's exact saved offer (the products/pricing the seller quoted) into a campaign — without re-running open discovery and hoping the same products surface.

**Flow:**

1. Buyer has shared a planning brief and a publisher has responded with `QUOTE`. The proposal code is on the response row (`proposalCode`) returned by `list_planning_brief_responses`.
2. When the buyer says "let's go with [publisher]'s offer" / "pull that into a campaign" / "accept the quote", call `discover_products` with `proposalCode` set. The server short-circuits open discovery and returns the seller's saved snapshot.
3. From there the flow rejoins the standard discovery → `add_discovery_products` → `create_campaign`/`update_campaign` path.

**Rules:**
- `proposalCode` is mutually exclusive with open-discovery filters in spirit. When `proposalCode` is set, the server **ignores** `brief`, `budget`, `channels`, `countries`, `refine`, `storefrontIds`, etc. and returns the seller's snapshot as-is. Don't try to "narrow" a redeemed snapshot — if the buyer wants different inventory, drop the proposal code and run open discovery instead.
- The caller's resolved operator (brand binding on the advertiser) must match the proposal's operator binding, or the redemption is rejected. If you get an operator-mismatch error, check that the `advertiserId` on the discovery call is the same advertiser the planning brief was shared from.
- Proposal codes expire **14 days** after the QUOTE was sent. After that, redemption returns an expiry error — tell the buyer the offer has expired and offer to share a fresh planning brief.
- One redemption per code. Once a proposal code has been redeemed against a media buy, attempting to redeem it again is rejected. If the buyer wants to apply the same offer to a second campaign, ask the publisher for a fresh quote.
- Don't expose the raw `PRP-XXXXXXXX` string to the buyer unless they ask. Refer to it as "the publisher's offer" or "the quote" in user-facing copy.

### Common mistakes to avoid

1. **Jumping to `discover_products` for a planning brief.** If the buyer is asking who can carry their launch, that is NOT discovery. Use the planning-brief flow.
2. **Sharing the brief without showing the matched publisher set.** Step 2 (Match + select) is required — the buyer must confirm which publishers to share with before the brief goes out.
3. **Fabricating or inferring buyer/advertiser names.** Apply the buyer-name resolution rule from Step 1: `buyerName` may ONLY come from (a) the `advertiserName` named in THIS brief, or (b) a name the user typed in response to your direct question. NEVER infer it from prior turns, prior sessions, prior tool calls, the customer/advertiser, the authenticated user, or any other context. If the user named an advertiser, `buyerName === advertiserName`. If they did not, STOP and ASK — do not call `share_planning_brief` with a guessed value. If you cannot quote the exact words in the current brief that produced `buyerName`, you are guessing.
4. **Truncating `rawPrompt`.** Send the buyer's full original brief verbatim — same rule as `brief` elsewhere.
5. **Wrong target shape.** `storefrontId` is for LIVE/COMING_SOON; `externalPublisherName` is for NOT_ON_INTERCHANGE. Never both on one target.
6. **Polling responses immediately after sharing.** Acknowledgments take real time — END YOUR TURN once the brief is shared. Check `list_planning_brief_responses` on a later turn when the user asks.
7. **Echoing seller-side framing to the buyer.** Use "planning brief" or "prospective brief". Don't echo seller-side framing ("demand signal", "incoming demand") to the buyer.
8. **Skipping advertiser resolution.** When the brief names an advertiser, you MUST call `list_advertisers` and set the numeric `advertiserId` on the share body. Sending `advertiserName` without `advertiserId` orphans the brief from the advertiser's planning history on the DB. The `advertiserId` field being marked OPTIONAL in the schema is a wire-level concession, not a behavioral one — resolve it whenever the brief names an advertiser.

---

## Campaigns

Every campaign has a `mode`: `discovery`, `performance`, or `directed`.
`discovery` and `performance` are platform-managed. `directed` means the
connected seller/ad platform is the source of truth and Interchange keeps a
one-to-one mirror. The mirror is read-only by default; upstream writes require
separate canary enrollment.

Create a platform-managed campaign via the `create_campaign` operation. Campaigns are configured at creation or update time with:

- **Products**: Select products via the `discover_products` and `add_discovery_products` operations, then attach it via `discoveryId` at campaign creation or update.
- **Performance optimization**: Set via the `performanceConfig` field at campaign creation (`create_campaign`) or update (`update_campaign`).
- **Audience targeting**: Target or suppress audiences via the `audienceConfig` field at campaign creation or update. Audiences are synced to the advertiser first via `sync_audiences`.
- **Auto-select products (pick for me)**: Use the `auto_select_products` operation to let the system automatically choose products and allocate budget using AI. Requires a performance campaign with discovered products.

### External AdCP buyers (alpha)

When a user asks whether their own AdCP SDK/client can control a seller through
Interchange, teach from the public
`/v2/concepts/adcp-on-the-buy-side` documentation. The inbound protocol flow is a
directed campaign because the client addresses one storefront:

- **Storefront-endpoint provenance** (`campaign.directed.provenance =
  "storefront_endpoint"`): an external client calls one Interchange-hosted storefront.
  The storefront manages execution while Interchange anchors contract, governance,
  ledger, and reporting evidence and dispatches the full zero-fee media budget. The
  campaign projects as `mode: "directed"`.
- **Connected-account provenance** (`campaign.directed.provenance =
  "connected_account"`): an Interchange buyer connects a seller/platform
  account and Interchange mirrors or forwards its 1:1 campaign. It also projects as
  `mode: "directed"`.

Ask whether the buyer is addressing one storefront directly or asking Interchange to
perform cross-storefront discovery/allocation. Direct is directed. A managed campaign
may still end with one seller, but the realized count does not change its mode.
“Seller-managed campaign” is explanatory copy for directed mode, not a replacement
API value. “Inbound” describes provenance, not a fourth campaign mode.

This is an external-client integration, not a sequence to drive by improvising
buyer `api_call` operations. Explain that the client discovers and transacts on
`/storefront/{platformId}/mcp` using standard AdCP tasks. The authenticated
credential must resolve to the buyer and advertiser, and enrollment requires a
currency policy plus per-buy and aggregate caps. Do not offer to provision the
PostHog flag or policy from a buyer conversation.

#### Meta creative image upload (gated)

On a Meta `ADAPTER` Storefront with private creative ingress configured,
`upload_creative_asset` opens a portable upload Task in compatible MCP App
hosts. Call the model-visible tool with the selected delegated Meta account as
`{ "account_id": "act_..." }`. The account must match the connection scope;
never invent an account ID or substitute credentials, URLs, or image bytes.

The Task accepts one JPEG or PNG up to 30,000,000 bytes. It checks the file
signature, size, and SHA-256 digest in the browser, then performs a direct
private upload. `prepare_creative_asset_upload` and
`finalize_creative_asset_upload` are app-only tools. Never call them from the
model, relay their signed URL, or persist that URL. Successful finalization
returns a stable `scope3-asset://v1/...` reference for the creative workflow;
use that reference rather than the temporary storage capability.

If `upload_creative_asset` is absent, the capability is not configured for
that Storefront. Explain that it is unavailable and do not invent an app-only
bypass. Treat fixture coverage as protocol evidence only: staging and
production readiness require successful Claude and ChatGPT live canaries.

State the money and reporting boundary precisely: the external client authorizes
a GROSS buyer budget; Interchange pins zero-fee AdCP entitlement terms, so GROSS and
NET are equal and the full media budget reaches the seller. `get_media_buys` and live
`get_media_buy_delivery` stay correlated by the dual-key media-buy identity.
Governed `update_media_buy` and cancellation are supported for storefront-endpoint
campaigns when the customer/storefront policy is active and the addressed storefront
declares the operation. They share the durable directed-write journal, budget-delta
caps, SCD2 activation, commitment reconciliation, and ambiguous-outcome fence. Adding
packages on this path remains unsupported. Webhooks, scheduled delivery, and
invoice-grade delivery are not part of the current alpha. Direct `ADAPTER` ingress
also requires the delegated provider account. Never describe the alpha delivery read
as a cached reporting history or a billing statement.

### Seller-managed campaigns (API mode: `directed`) (alpha)

> Treat enrollment as one customer-facing directed-campaign capability. Internal
> PostHog kill switches protect individual paths but are not separate products to
> teach buyers. If a directed operation returns `FEATURE_NOT_ENABLED`, tell the buyer
> the alpha is not enabled for their
> account. Upstream creates/updates require the separate
> `directed-campaign-writes` customer flag; mirror access never implies write
> access. Every active storefront is eligible for directed mode. Treat
> enumeration, delivery, products, create, and update as independent operations:
> report an unsupported or unhealthy operation precisely without hiding the
> storefront or inventing an untyped bypass.

Use **seller-managed campaign** in buyer-facing language. `directed` is the
stable API mode, not a second product name; never invent a `seller_managed`
mode or campaign type.

Seller-managed campaigns are storefront-authoritative 1:1 projections. Do not run
discovery or execute them, and never edit the shell independently. A directed
campaign is exactly one storefront campaign, one AdCP media buy, and one seller. Its
shell fields are projections of the buy. Connected-account mirrors are read-only by
default; separately enrolled customers may forward strict AdCP creates/updates
through `create_campaign` and `update_campaign`. A `brief` on
`mode: "directed"` is a validation error, never an ignored field.
To pull a fresh upstream snapshot without writing to the seller, call
`update_campaign` with `{ "mode": "directed", "refresh": true }`.

Choose the mode by management responsibility:

- `directed`: the buyer addresses one storefront directly, including an inbound AdCP
  endpoint or an existing connected seller/platform campaign;
- `discovery`: the buyer has a brief and wants Interchange to find/transact;
- `performance`: the buyer delegates cross-seller allocation to an objective.

A discovery campaign can finish with one seller and remain platform-managed when
Interchange performed the selection. Without the separate connected-account write
alpha, a new seller-owned campaign must be created in the seller platform first and
then mirrored. The inbound AdCP buyer edge has its own governed flag and policy.

Required setup and flow, one operation per turn:

1. Identify the provider path. For an official social adapter, use
   `list_storefronts` with `limit: 100`, inspect `adapterConnection`, and follow
   steps 2–3. For a registered external AdCP source, use
   `get_storefront_capabilities` and `list_agent_credentials`, then follow the
   registered-source branch below. Never accept an arbitrary endpoint URL or
   raw secret for seller-managed campaigns.
2. **Official adapter:** if it is not connected, call `connect_storefront`, give
   its `connectionUrl` to the human, and END YOUR TURN while they complete
   OAuth. Never ask them to paste provider credentials into chat. After OAuth,
   use `list_storefront_connections`, then
   `list_storefront_connection_accounts`, and let the buyer choose a buyable
   advertiser account. Organization/manager and publisher-identity rows cannot
   be subscribed.
3. **Official adapter mapping:** use
   `list_storefront_connection_account_mappings`. If the account is not mapped,
   use `list_advertisers`, ask which advertiser owns it, then call
   `map_storefront_connection_account_to_advertiser`. Re-list mappings later
   and do not proceed until the mapping is visible.
4. **Registered AdCP source:** register credentials only through
   `register_source_credentials`, discover the source account with
   `list_available_accounts`, and link it to the selected advertiser using the
   advertiser account flow. Then call `connect_adcp_storefront` with the exact
   `storefrontId`, `sourceId`, `advertiserId`, discovered `accountId`, and its
   `credentialId` when present. The returned `connectionId` and
   `connectionAccountId` enter the same subscription flow. Do not use this
   operation for official adapters, managed sources, unknown storefronts, or
   sources whose seller identity and authorization have not been registered.
   Reported tools are stored as readiness evidence, not an admission gate. A
   source without plural `get_media_buys` cannot enumerate the account, but it
   may still connect and support other directed operations.
5. Call `subscribe_directed_campaigns` with the mapped IDs. Do not send
   governance: `maxMediaBuyBudget`, `maxAccountBudget`, and `currency` are
   operator-owned controls that ordinary buyers cannot set or raise. All three
   are required before an upstream write; the Interchange account team records
   them after pilot authorization. Missing governance keeps the subscription
   read-only. Subscription starts
   enumeration and a one-year campaign-metadata backfill attempt. Enrollment is
   durable even when that operation is unsupported or temporarily unhealthy;
   the response then reports subscription status `ERROR`, `lastSyncError`, and
   zero reconciliation counts. Do not claim that delivery history is imported
   at subscribe time.
6. On a later turn, call `get_directed_campaign_subscription` with the same
   connection/account path params to inspect `lastSyncedAt`, `lastSyncStatus`,
   `lastSyncError`, and the fixed metadata-history boundary `backfillStart`.
   This is sync health, not a running backfill percentage or delivery-import
   status. It works even if feature exposure is later disabled.
7. On the next turn, call `list_campaigns` with `params: { "mode":
"directed", "advertiserId": "..." }` to show the mirrors.
8. Call `get_campaign_delivery` with `pathParams.campaignId` to read delivery
   through the connected seller. Optional `params.startDate` / `endDate` use
   `YYYY-MM-DD`; omitted dates default to the one-year window through yesterday.

Recovery guidance:

- `FEATURE_NOT_ENABLED`: say the customer is not enrolled in the requested read
  or write alpha. Mirror access does not imply write access. Do not attempt an
  adapter-tool bypass.
- Account or mapping not found: re-run the corresponding official-adapter list
  operation and use only IDs it returns. A connected account is not subscribed
  until it is mapped to the intended advertiser.
- `lastSyncStatus: "ERROR"`: show `lastSyncError` and `lastSyncedAt` as the time
  of that failed attempt. Existing mirrors are retained on a failed or partial
  snapshot. If the error is authentication-related, reconnect the adapter;
  otherwise ask whether the buyer wants to retry a read-only refresh.
- `mirrored: 0` with `lastSyncStatus: "SUCCESS"` is a successful empty account,
  not a failed backfill. Say that no campaigns were eligible in the one-year
  metadata window.

When the buyer wants to stop mirroring, call `unsubscribe_directed_campaigns`
with the connection/account path params. It pauses the subscription and retires
the mirrored shells; it does not delete campaigns in the seller platform.

The default worker sweep re-enumerates campaign metadata every 15 minutes.
Subscription status is the account-level source of truth. Campaign summaries and
the campaign widget show mode, provider, mirror state, and
`campaign.directed.lastSyncedAt` for row-level freshness.

**Delivery is always live read-through.** Every `get_campaign_delivery` call
invokes the seller's `get_media_buy_delivery`. The response is returned to that
caller but its raw payload and time series are not persisted or served later.
Only the latest aggregate media-buy/package metrics, exact requested
`windowStart`/`windowEnd`, and refresh timestamp are persisted. When presenting
the snapshot from `get_campaign`, show its window and `lastUpdated`; never call
it lifetime delivery. Subscription and metadata sweeps do not pull delivery.
Never describe Phase 1 delivery as a cache hit or completed reporting backfill.

```json
{
  "operation": "connect_adcp_storefront",
  "pathParams": { "storefrontId": "42", "sourceId": "publisher-sales" },
  "body": {
    "advertiserId": "12345",
    "accountId": "seller-account-7",
    "credentialId": "81"
  }
}
```

```json
{
  "operation": "subscribe_directed_campaigns",
  "pathParams": { "connectionId": "42", "accountId": "81" },
  "body": {
    "advertiserId": "12345",
    "sourceId": "tiktok"
  }
}
```

```json
{
  "operation": "get_directed_campaign_subscription",
  "pathParams": { "connectionId": "42", "accountId": "81" }
}
```

```json
{
  "operation": "get_campaign_delivery",
  "pathParams": { "campaignId": "campaign_123" },
  "params": { "startDate": "2026-01-01", "endDate": "2026-07-10" }
}
```

### Directed writes (narrower alpha)

Use these only after the mapped subscription and provider are write-enabled.
The provider remains source of truth; the campaign operation records durable
intent, forwards one AdCP write, then reconciles the mirror. Webhooks and
scheduled-delivery integration are still unavailable.

Before collecting package IDs, call `list_directed_campaign_products` with the
discovered `connectionId`/`accountId` in `pathParams` and the mapped
`advertiserId` in `params`. Show the returned product names, `product_id`, and
each `pricing_option_id`, then let the buyer choose. This is a no-brief provider
catalog read; it does not create a discovery session. Never substitute a
product ID from another storefront or one supplied only in conversation.

```json
{
  "operation": "list_directed_campaign_products",
  "pathParams": { "connectionId": "42", "accountId": "81" },
  "params": { "advertiserId": "12345" }
}
```

For `create_campaign`, send exactly this root structure:

- `mode: "directed"`
- `advertiserId`, `connectionId`, `accountId`, and `name`, all sourced from
  prior list/mapping responses or explicit user confirmation
- a caller-generated `idempotencyKey` of 16–255 allowed characters, stable for
  retries of this exact request
- `currency`, matching the subscription governance currency
- `mediaBuy`, an explicit AdCP create payload with `brand`, `start_time`,
  `end_time`, optional `total_budget`, and explicit `packages`

Never put `account` or `idempotency_key` inside `mediaBuy`. Never put a `brief`,
campaign `flightDates`, or campaign `budget` on a directed request. Obtain
product and pricing-option IDs from the connected provider; do not invent them.
For the TikTok alpha canary, require explicit user confirmation and use future
dates. The server requires `paused: true` and rejects creative assignments or
inline creatives until the proof advances. Each package must include at least
one provider-supported ISO country in `targeting_overlay.geo_countries`.
For the `tiktok_reach` product, each package must also include an explicit
`targeting_overlay.frequency_cap` with `max_impressions` from 1 through 1000
and a `window` of 1 through 30 `days`; never invent this buyer policy.
Confirm the requested amount and currency are within both stored governance
caps before calling.

```json
{
  "operation": "create_campaign",
  "body": {
    "mode": "directed",
    "advertiserId": 12345,
    "connectionId": "42",
    "accountId": "81",
    "name": "TikTok reach canary",
    "idempotencyKey": "buyer-2026-07-canary-001",
    "currency": "USD",
    "mediaBuy": {
      "brand": { "domain": "example.com" },
      "total_budget": { "amount": 60, "currency": "USD" },
      "packages": [{
        "product_id": "tiktok_reach",
        "pricing_option_id": "tiktok_reach_cpm",
        "budget": 60,
        "targeting_overlay": {
          "geo_countries": ["US"],
          "frequency_cap": {
            "max_impressions": 3,
            "window": { "interval": 7, "unit": "days" }
          }
        }
      }],
      "start_time": "2026-07-20T00:00:00Z",
      "end_time": "2026-07-21T00:00:00Z",
      "paused": true
    }
  }
}
```

For an upstream update, call `update_campaign` with the Interchange campaign ID
in `pathParams` and `{mode, idempotencyKey, currency, expectedRevision?,
mediaBuy}` in the body. The URL supplies account and media-buy identity; reject
or remove nested `account`, `media_buy_id`, `idempotency_key`, and `revision`.
Use root `expectedRevision` from the latest campaign read when available. The
`mediaBuy` patch must be non-empty.
For registered AdCP sellers, a budget or package mutation requires a non-null
`expectedRevision` from the latest mirror. Refresh once if it is missing; if the
seller still supplies no revision, do not attempt an exposure mutation. Pause or
cancel may remain available because they cannot increase spend.

```json
{
  "operation": "update_campaign",
  "pathParams": { "campaignId": "campaign_123" },
  "body": {
    "mode": "directed",
    "idempotencyKey": "buyer-2026-07-cancel-001",
    "currency": "USD",
    "expectedRevision": 1,
    "mediaBuy": {
      "canceled": true,
      "cancellation_reason": "Canary completed"
    }
  }
}
```

A synchronous response returns `{campaign, directedWrite}`;
`directedWrite.status: "CONFIRMED"` means the provider result has been reconciled
and the campaign object is current. A `202` raw write result with
`AWAITING_CONFIRMATION` means the provider may have accepted the write but the
mirror has not proved it yet; show that state and stop. On a later turn, use
`get_campaign` or the read-only refresh shape. Reuse an idempotency key only for
an identical retry. Never retry an ambiguous outcome with a new key, because
that can create a duplicate upstream campaign.

Some providers, including TikTok, omit a deleted campaign from the next complete
account snapshot. When that absence follows the pending cancellation for the known
mirror, the sweep confirms the operation, records the projected buy as canceled,
and keeps its terminal shell readable. Report the campaign as canceled and the
write as confirmed; do not describe the provider-omitted shell as lost or failed.
The retained campaign and its media buy both report `status: "CANCELED"`.
Only an identical replay with the original idempotency key is valid afterward;
never invent a new key or attempt another mutation on the terminal shell.

The sync sweep marks a write left dispatching for more than five minutes as
ambiguous. A known-ID update can still reconcile; a create without a returned
upstream ID requires operator inspection of the connected account before any
further mutation.

TikTok alpha limitation: the first canary accepts only a paused create,
`{paused:true}`, or `{canceled:true,cancellation_reason?}`. Resume, package
mutations, schedule/bid changes, and every other update field fail closed.
On create, a package may include canonical `catalogs[]` of type `offering` or
`job`; TikTok expands those items into promoted offerings. Do not send other
catalog types—the provider capability guard rejects them before dispatch.
TikTok does not yet provide authoritative revision/CAS protection for a safe
platform-side budget projection. Tell the buyer to make other changes in TikTok
until the post-canary capability review; never invent a bypass.

**⚠️ HARD RULE: One API Call Per Turn**

Only make ONE mutating or discovery API call per turn. After that call, present the results and END YOUR TURN. Do not paginate, re-discover, or chain additional calls — wait for the user to tell you what to do next.

**Required flow when user says "create a campaign":**
1. First determine who should steer: use `directed` when the buyer addresses one
   storefront directly or a connected seller/platform is authoritative; use
   `discovery` when Interchange should act from a brief; use `performance` when
   Interchange should allocate toward an objective. A managed campaign can still
   realize one seller. If the user chooses a directed write, follow the narrower
   alpha flow above and do not collect a brief or continue through steps 2–7 below.
2. For a platform-managed campaign, collect campaign details: name, advertiser, budget, any stated budget currency, flight dates, brief, and any targeting constraints. Routing (decisioned vs routed) is not a campaign input — each media buy derives it from the storefront it executes against, so never ask the user to choose. The budget is always GROSS — `budget.total` is the all-in amount the customer pays, and the Scope3 fee is carved out of it server-side. There is no separate `mediaBudget` field in the response and no fee-model choice to collect.
3. Ask whether they want to target or suppress any audiences. If yes, list the advertiser's audiences (`list_audiences` operation) and let them choose. If the advertiser has no audiences, let the user know and offer: "You don't have any audiences synced yet. I can help you sync audiences to Scope3 — just ask me how to get started."
4. Ask if they want to attach a catalog — if yes, list their catalogs via `list_catalogs` operation and let them pick one
5. Optionally collect `performanceConfig.optimizationGoals` — if the user wants Scope3 to optimize toward specific performance goal(s), ask what they want to optimize toward. This is optional; campaigns can be created without it.
6. Ask if they also want to browse and attach specific products (discovery). Based on their choice:
   - **Yes**: Run discovery ONCE, present results, END YOUR TURN. The user drives what happens next.
   - **No**: Proceed to campaign creation.
7. Include `audienceConfig` if the user selected audiences in step 3
8. When ready, launch: use `execute_campaign` operation

**Required fields for platform-managed campaign creation:**
- `advertiserId` (number) — NOT `advertiser_id`
- `name` (string)
- `flightDates` (object) — NOT `startDate`/`endDate` at root level
  - `flightDates.startDate` (ISO 8601 datetime)
  - `flightDates.endDate` (ISO 8601 datetime)
- `budget` (object) — NOT a number
  - `budget.total` (number)
  - `budget.currency` (string, optional). Every campaign is created in the advertiser's currency; an advertiser is single-currency. Pass the advertiser's currency or omit it — never a different currency, the server rejects mismatches (400, field `budget.currency`). Preserve the user's stated currency as a three-letter ISO code (`€`/`euro`/`euros`/`Euro` -> `"EUR"`, `£`/`pound`/`pounds`/`sterling` -> `"GBP"`, `US dollar`/`USD` -> `"USD"`); do not let a non-USD request fall back to USD. Ask if ambiguous, including standalone `$`, `dollar`, `dollars`, or `dólares`. Currency is immutable after campaign creation.

The budget is always GROSS — `budget.total` is the all-in amount the customer pays, fee included. Media buy budgets are gross too and are capped against `budget.total` directly (`Σ media buy budgets ≤ budget.total`, same denomination — there is no separate post-fee media ceiling). Use `unallocatedBudget` (see **Get Campaign** below) for the remaining headroom rather than re-deriving it yourself. There is no fee-model field on the campaign to send or ask the buyer about.

**Optional fields at creation:**
- `discoveryId`: Attach an existing discovery session
- `productIds`: Product IDs to pre-select (requires discoveryId)
- `performanceConfig`: For performance optimization. Contains `optimizationGoals` array. Each goal has `kind` (`"event"` or `"metric"`). Event goals have `eventSources` array (each with `eventSourceId`, `eventType`, optional `valueField`) + optional `target` object (`kind: "per_ad_spend"` or `kind: "cost_per"` with `value`). Metric goals have `metric` string + optional `target`. Goals can include `attributionWindow` and `priority`.
- `optimizationApplyMode`: `"AUTO"` or `"MANUAL"` (default). Controls whether Scope3 AI model optimizations to media buys are applied automatically or require manual approval. Overrides the advertiser-level default.
- `catalogId` (number, optional): Attach a single catalog to the campaign. Only **one** catalog per campaign. The catalog must belong to the same advertiser. Get available catalogs via `list_catalogs` operation. When attached, the catalog is automatically included in product discovery requests — referenced by ID for agents that have the catalog syndicated, or sent inline (feed URL or items) otherwise.
- `constraints.channels`: Target channels (display, olv, ctv, social)
- `constraints.geo_countries` / `geo_countries_exclude`: ISO 3166-1 alpha-2 country codes (`countries` is accepted as a deprecated alias and normalized to `geo_countries`)
- `constraints.geo_regions` / `geo_regions_exclude`: ISO 3166-2 subdivisions (e.g. `"US-CA"`)
- `constraints.geo_metros` / `geo_metros_exclude`: metro targeting, `{ system, values }` objects (e.g. Nielsen DMAs)
- `constraints.geo_postal_areas` / `geo_postal_areas_exclude`: postal-area targeting, preferably `{ country, system, values }` objects (e.g. `{ country: "NL", system: "postal_code", values: ["1011"] }`). Use legacy `{ system, values }` only for deprecated country-fused systems.
- City names are not a supported targeting field. When a buyer names cities/towns, preserve the exact city list in `brief` and use a machine-readable overlay only when you can resolve the intent safely: `geo_postal_areas` for known postal areas, `geo_proximity` for radius/travel-time/store-area intent, or `geo_metros` only for declared metro systems. Do not invent `geo_cities`, do not pass raw city names as targeting, and call out ambiguous places (for example, Bergen or Laren) for confirmation.
- `constraints.language`, `constraints.device_type`, `constraints.device_platform`: AdCP language/device overlays
- `brief`: Campaign brief. **MUST be the ENTIRE brief from the client — never summarize or truncate.**
- `audienceConfig`: Audience targeting and suppression. Contains `targetAudienceIds` (audiences to **include**) and `suppressAudienceIds` (audiences to **exclude**). Audience IDs come from `list_audiences` operation.

---

## Catalogs (product feeds)

A catalog is a feed of an advertiser's **products** (items, prices, links, images) synced onto the advertiser account and used for catalog-based buying / dynamic creative. Catalogs are per-advertiser and distinct from creative assets.

**Operations** (all scoped to `/api/v2/buyer/advertisers/:advertiserId`):
- `list_catalogs` — `GET .../catalogs`. List the advertiser's catalog feeds with health, sync status, item count, and latest version.
- `refresh_catalog` — `POST .../catalogs/:catalogId/refresh`. Re-pull a URL-backed feed (`executeActivation` optional).
- `preview_catalog_activation_plan` — `POST .../catalogs/:catalogId/activation-plan/preview`. Dry-run the plan (campaign groups, syndication targets, and — currently gated — creative assets) without writing; `save` optional.
- `execute_catalog_activation_plan` — `POST .../catalogs/:catalogId/activation-plan/execute`. Persist the plan's downstream jobs as `pending` (`dryRun` optional). A runner that carries out those jobs (live campaigns, creative, seller push) is not yet available, so executing does not by itself make anything go live; the creative leg is additionally gated on brand identity + per-format output.

**As an in-chat widget:** `open_buyer_catalogs` opens the Catalogs widget — a chat-first card that lists the advertiser's feeds, drills into one feed for detail, and supports sync (hosted feeds) / reupload (uploaded feeds) + preview and save-plan inline. Open it for "show me my catalogs", "sync my product feed", or "set up this catalog's activation plan"; answer a one-off catalog question directly instead.

---

## Browsing Products Before Creating a Campaign

**When a user wants to browse products but hasn't created a campaign yet:**

Users may want to explore available inventory before committing to a campaign. Use the `discover_products` operation which discovers products based on the advertiser's context and returns a `discoveryId` along with the discovered products.

**Interactive flow:**
1. **Discover products** — Use the `discover_products` operation with advertiser context
   - Returns `discoveryId` and product groups — save the `discoveryId` for later use
2. **Present products** — Show available inventory in a user-friendly way
3. **Refine (optional)** — If the user wants changes ("more video", "remove that one", "more like this"), use `discover_products` operation again with `discoveryId` and a `refine` array
4. **Select products** — When the user likes products, add them via `add_discovery_products` operation
5. **Attach to a campaign** — Create a campaign with the `discoveryId` via `create_campaign`, or attach it to an existing campaign via `update_campaign` with `discoveryId`

**Request Parameters (Filtering):**
- `publisherDomain` (optional): Filter products by publisher domain (exact domain component match). Example: "example" matches "example.com", "www.example.com" but "exampl" does not match
- `storefrontIds` (optional, array of integers): **Highly encouraged when the user wants results scoped to specific sellers.** Filter discovery to these storefronts (from `list_storefronts`).
- `storefrontNames` (optional, array of strings): Filter discovery to storefronts whose name matches (case-insensitive substring). Use when the user names a seller (e.g. "Acme", "Acme Exchange").
- `pricingModel` (optional): Filter by pricing model (`cpm`, `vcpm`, `cpc`, `cpcv`, `cpv`, `cpp`, `flat_rate`). Use when a user wants inventory with a specific pricing type.
- `proposalCode` (optional): Storefront-issued proposal code (e.g. `"PRP-XK4A29"`) returned on a `QUOTE` response to a planning brief. When set, the server short-circuits discovery and returns the seller's saved snapshot; all other discovery filters (`brief`, `budget`, `channels`, `refine`, etc.) are ignored. See "Redeeming a QUOTE via `discover_products`" in the Planning Briefs section for the full flow.

> **Note:** Empty arrays for `storefrontIds` / `storefrontNames` are treated the same as omitting the field — both mean "no request-level filter." If a `campaignId` is also provided and the campaign was created with `storefrontIds`, the campaign-level pin is used as the fallback in that case.

See the Campaign Workflow below for the full step-by-step with HTTP examples.

---

## Adding Products to a Campaign

**When the user wants to choose specific inventory:**

Product discovery and selection is done via the discovery endpoints. Discover products, select the ones you want, then attach them to the campaign.

1. **Discover products** — Use `discover_products` operation
   - Show product groups, publishers, channels, and price ranges in a user-friendly way
2. **Present results and let the user choose** — Show the discovered products and ask which ones they want to add. Do NOT auto-select products for the user.
3. **Refine (optional)** — If the user wants to iterate ("more like this", "remove that", "more video options"), use `discover_products` operation again with `discoveryId` and a `refine` array
4. **Add their selections** — Use `add_discovery_products` operation with the products the user chose
   - Show the updated product list and budget allocation
5. **Attach to campaign** — Create the campaign with `discoveryId` via `create_campaign`, or update an existing campaign via `update_campaign` with `discoveryId`
5. **Confirm readiness** — "Your campaign has X products selected with $Y allocated. Ready to launch?"
6. **Launch** — Use `execute_campaign` operation

See the Discovery Workflow (Pre-Campaign Product Discovery) section below for the full step-by-step with HTTP examples.

### Setting Performance Optimization

**When the user wants the system to optimize for business outcomes:**

1. **Check conversion events** — Use `get_event_summary` operation with `eventType: "conversion"` to see what events are available for optimization
   - If none exist, help the user configure event sources first
   - **Note:** Event data is aggregated hourly. Newly reported events may take up to 1 hour to appear in the summary.
2. **Set performance config** — Include `performanceConfig` at campaign creation (`create_campaign`) or update (`update_campaign`)
   - Required: `optimizationGoals` array with at least one goal object
   - Each goal has `kind` (`"event"` or `"metric"`)
   - Event goals: `eventSources` array (each with `eventSourceId`, `eventType`, optional `valueField`), optional `target` (`kind: "per_ad_spend"` or `kind: "cost_per"` with `value`), optional `attributionWindow`, optional `priority`
   - Metric goals: `metric` string, optional `target`, optional `priority`
3. **Launch** — Use `execute_campaign` operation

### Auto-Selecting Products (Pick For Me)

**When the user wants the system to choose products automatically:**

Instead of manually browsing and selecting products, performance campaigns can use auto-selection:

1. **Ensure products are discovered** — The campaign must have discovered products (via `discover_products` operation or auto-discovery at campaign creation with `performanceConfig` + `constraints.channels`)
2. **Auto-select** — Use `auto_select_products` operation (no request body needed)
   - The system uses AI to select the best products based on the campaign brief, budget, constraints, and optimization goals
   - Budget is allocated across selected products based on strategic fit
   - Any previous product selections in the discovery session are replaced
3. **Review selections** — Present the selected products, budget allocations, and rationale to the user
4. **Refine (optional)** — The user can adjust selections:
   - `discover_products` operation with `discoveryId` and `refine` — Iterate on results (e.g., "more like this", "omit that", "more video")
   - `add_discovery_products` operation — Add products
   - `remove_discovery_products` operation — Remove products
   - Or use `auto_select_products` operation again to re-select
5. **Launch** — When the user confirms, use `execute_campaign` operation

**Response includes:**
- `selectedProducts`: Array of products with budget allocations
- `budgetContext`: Campaign budget vs allocated amount
- `selectionRationale`: AI-generated explanation of the selection strategy
- `selectionMethod`: `"scoring"`, `"measurability"`, or `"cpm_heuristic"`
- `testBudgetPerProduct`: Test budget allocated per product (when using measurability or scoring strategy)
- `productCount`: Number of products selected

---

## Discovery Workflow (Pre-Campaign Product Discovery)

**When to use:** User wants to browse, select, or control which specific inventory/products to include before or independently of campaign creation.

**Prerequisites:** Advertiser exists with a linked brand (set during advertiser creation via `brand`).

**Routing and discovery:**
- Discovery always returns both decisioned and routed inventory — there is no routing choice to make before discovering, and a campaign does not filter discovery by type. If the user wants execution through specific storefronts, narrow results via `storefrontIds` / `storefrontNames`.
- Routing (`DECISIONED` vs `ROUTED`) is a property of each **media buy**, derived server-side from the storefront it executes against — never a client input, and a single campaign can mix both. Never ask the user to choose DECISIONED vs ROUTED.
- Adapter storefronts (for example Snap, Google, Meta, Spotify, TikTok, Pinterest, Reddit, Amazon) are **routed-only**: a media buy through one passes through the buyer's registered provider credentials without Scope3 decisioning. Media buys on every other storefront are decisioned and use Scope3 optimization.

**Highly encouraged: ask the user which storefronts to target.** Discovery returns far more relevant results — and the buyer keeps control of seller mix — when you scope to specific storefronts via `storefrontIds` (preferred, IDs from `list_storefronts`) or `storefrontNames`. Only run unscoped discovery if the user explicitly wants to see everything available. If a campaign was created with `storefrontIds`, that filter is auto-applied to every discovery run on that campaign — you don't need to resend it.

### Interactive Flow

Follow these steps in order. **Do NOT skip product discovery.**

**Step 1: Discover products**

**Operation:** `discover_products`
```http
POST /api/v2/buyer/discovery/discover-products
{
  "advertiserId": "12345",
  "channels": ["ctv", "display"],
  "countries": ["US"],
  "brief": "<<< ALWAYS include the ENTIRE brief from the client here — never summarize >>>",
  "storefrontIds": [42, 57]
}
```
**As operation:**
```json
{ "operation": "discover_products", "body": { "advertiserId": "12345", "channels": ["ctv", "display"], "countries": ["US"], "brief": "<<< ALWAYS include the ENTIRE brief from the client here — never summarize >>>", "storefrontIds": [42, 57] } }
```
→ Returns `{ "discoveryId": "...", "productGroups": [...], "totalGroups": 25, "hasMoreGroups": true, "summary": { ... } }`

Save the `discoveryId` for all subsequent steps.

**MCP progress notifications:** `discover_products` supports the MCP protocol's `_meta.progressToken`. When the client supplies one on the `api_call` tool call, the server emits `notifications/progress` once per seller as it answers during the fan-out — `progress` = sellers settled, `total` = sellers queried, message like "Magnite answered — 11 products (3 of 14)" — before the tool result returns. Works on both progressive and non-progressive calls; clients that send no progress token see no change.

**Step 2: Present results and END YOUR TURN**

Present the discovered products and END YOUR TURN. If `hasMoreGroups: true`, tell the user more are available. If `incompleteAgents` is present with `retryWithLongerWaitAvailable: true`, explain that some storefronts timed out or returned partial results under the quick wait, include any seller-provided `incompleteScopes` detail, and ask whether the user wants to retry with a longer wait window before setting `waitMode: "long"` or `waitSeconds` above 30.

To browse more products or apply filters, use:

**Operation:** `browse_discovery`
```http
GET /api/v2/buyer/discovery/{discoveryId}/discover-products?groupLimit=10&groupOffset=0&productsPerGroup=15
```
**As operation:**
```json
{ "operation": "browse_discovery", "pathParams": { "discoveryId": "<id>" }, "params": { "groupLimit": "10", "groupOffset": "0", "productsPerGroup": "15" } }
```

**When discovery returns no products:**

A discovery response with 0 products is **not an error** — it means no products matched. Do NOT say "discovery failed." Do NOT speculate about why (e.g. "likely because X wasn't set up correctly" — you have no idea why, and guessing will be wrong and confusing). Just state the fact and offer these specific next steps:

1. **Add specificity** — Include budget, flight dates, specific channels (e.g. CTV, display), or audience targeting. Richer briefs give agents more to match against.
2. **Try different channels or geos** — The available inventory may not cover the requested combination.
3. **Reduce the ask** — If the brief is very narrow (e.g. a niche audience + specific publisher + tight budget), broadening one or more constraints often unlocks results.
4. **Try specific filters** — Filter by `storefrontIds` (preferred), `storefrontNames`, or `publisherDomain` to target sellers known to have relevant inventory.

Example response when no products are returned:
> No products were returned for this brief. A few things that might help: adding a budget or flight dates, specifying channels (CTV, display, etc.), broadening the audience, or filtering by a specific seller. Want to try refining the brief?

**Explaining product relevance (IMPORTANT):**

Each product includes a `briefRelevance` field that explains WHY the product is a strategic fit for the campaign. When presenting products, you MUST:

1. **Lead with the "why"** — Don't just list product names and CPMs. For each product or product group, explain why it matters for THIS specific campaign. Use the `briefRelevance` text as your starting point but make it conversational.
2. **Connect products to the brief** — Reference the user's campaign goals, target audience, or brand context. Example: "These products from Acme target family audiences with parenting and food-related segments — a direct match for your Fanta Meals campaign aimed at families."
3. **Highlight strategic differentiators** — Call out what makes a product stand out: guaranteed delivery, best-value CPM, audience segment alignment, or estimated reach. Don't bury these in a list.
4. **Group-level insight** — When presenting a sales agent's products, summarize what that agent brings to the table for this campaign. Don't just say "Products from Acme Sales (5 products)" — say WHY Acme Sales matters here.
5. **Skip empty relevance** — If a product has no `briefRelevance`, present it normally with its attributes. Don't fabricate relevance that doesn't exist.

**Bad example** (too basic):
> Acme Sales has 5 products available: Americas Test Kitchen ($12 CPM, CTV), Rakuten TV ($8.50 CPM, CTV)...

**Good example** (explains why):
> Acme Sales is a strong fit here — they offer family-targeting segments like "Parenting Babies and Toddlers" and "With Children" that align directly with your Fanta Meals audience. Their Americas Test Kitchen inventory puts your ads in a food/cooking context, and at $8.50–$12.00 CPM you're getting competitive rates with guaranteed delivery on several products.

### Filtering Product Discovery Results

When discovering products, these filters narrow results before grouping and pagination:

- `publisherDomain`: Filter by publisher website. Use when a user mentions a specific publisher or website.
  - Example: "hulu" matches "hulu.com", "www.hulu.com" but "hul" does not
- `storefrontIds` (array of integers): **Highly encouraged.** Filter to these storefronts (IDs from `list_storefronts`).
- `storefrontNames` (array of strings): Filter to storefronts whose name matches (case-insensitive substring). Use when the user names a seller. Accepts multiple values.
- `pricingModel`: Filter by pricing model. Use when a user asks about specific pricing types.
  - Valid values: `cpm`, `vcpm`, `cpc`, `cpcv`, `cpv`, `cpp`, `flat_rate`
- `proposalCode`: Not a filter — a **redemption handle**. Set this to a `PRP-XXXXXXXX` code returned on a planning-brief `QUOTE` to pull the seller's saved offer into discovery. When present, all other filters are ignored. See Planning Briefs → "Redeeming a QUOTE via `discover_products`" for the full flow, rules, and 14-day expiry.

Filters can be combined. Multiple values within a filter use OR logic (match any); different filters use AND logic. Empty arrays for `storefrontIds` / `storefrontNames` are treated the same as omitting the field — both mean "no request-level filter," and the campaign-level pin is the fallback if a `campaignId` is also provided.

**How to communicate filtering to users:**

Do NOT mention filter parameter names. Respond naturally:
- User: "What do you have from [storefront name]?" → filter by `storefrontNames`
- User: "Show me [publisher] inventory" → filter by `publisherDomain`
- User: "What does [storefront] have on [publisher]?" → filter by both `storefrontNames` and `publisherDomain`
- User: "Show me inventory from [storefront 1] and [storefront 2]" → filter by `storefrontNames` (or `storefrontIds` if you have them)
- User: "Show me CPM inventory" → filter by `pricingModel=cpm`
- User: "What flat rate options are there?" → filter by `pricingModel=flat_rate`
- User: "Who sells CTV inventory?" → show unfiltered results, then offer to narrow by storefront

Each product group corresponds to a storefront's underlying sales agent. To focus on a specific storefront's inventory on subsequent requests, use the storefront filter.

**Step 2.5: Refine results (optional)**

If the user wants to iterate on the results, call `POST /discovery/discover-products` again with the same `discoveryId` plus a `refine` array. Each item in `refine` MUST be an object (not a string):

```http
POST /api/v2/buyer/discovery/discover-products
{
  "advertiserId": "12345",
  "discoveryId": "<<discoveryId from step 1>>",
  "refine": [
    { "scope": "request", "ask": "show me more video options" }
  ]
}
```

To omit a product:
```json
{ "refine": [{ "scope": "product", "id": "product_123", "action": "omit" }] }
```

To get more like a product:
```json
{ "refine": [{ "scope": "product", "id": "product_123", "action": "more_like_this" }] }
```

The response is the same shape as step 1 plus an optional `refinementApplied` array. Present updated results and let the user iterate or proceed to select.

**Step 3: Select products interactively**

Users can select products in two ways:
1. **Via the interactive card UI** — Users select product cards and click "Select". When this happens, you'll receive a message containing the discoveryId, productIds, and the exact `add_discovery_products` API call to execute. The message will also ask you to prompt the user about per-product budgets. **Ask the user if they'd like to set individual budgets before executing the API call, AND how they'd like to split the budget — by publisher then product, or just by product.** If they provide budgets, add a `budget` field to each product in the request body using their chosen split strategy. If they decline, execute the call as-is without budgets. Do not re-discover products or look up IDs.
2. **Via conversation** — Users describe which products they want in natural language. The productId, salesAgentId, groupId, and groupName are included in the product listing from the discovery response — extract them from there to build the API call. Do not re-discover products to obtain IDs.

**Per-product budget:** Each product supports an optional `budget` field (number, in dollars). Ask about budgets before adding products — don't assume a budget if the user hasn't specified one. When asking, ALWAYS ask how they want to split the budget: **by publisher then product** (allocate per publisher first, then split within each publisher's products — use when the user cares about publisher mix or has publisher-level commitments) or **just by product** (flat allocation across products regardless of publisher — use when the best products should win regardless of publisher). Apply their choice to the per-product `budget` values.

**Bid price (REQUIRED for non-fixed pricing):** When a product's selected pricing option has `isFixed: false`, you MUST include `bidPrice` in the request body. Use the `rate` or `floorPrice` from the product's `pricingOptions` (from the discovery response) as the `bidPrice` value — do NOT ask the user for this. If `isFixed: true`, omit `bidPrice`.

**Operation:** `add_discovery_products`
```http
POST /api/v2/buyer/discovery/{discoveryId}/products
{
  "products": [
    {
      "productId": "product_123",
      "salesAgentId": "agent_456",
      "groupId": "ctx_123-group-0",
      "groupName": "Publisher Name",
      "bidPrice": 12.50,
      "budget": 5000
    }
  ]
}
```
**As operation:**
```json
{ "operation": "add_discovery_products", "pathParams": { "discoveryId": "<id>" }, "body": { "products": [{ "productId": "product_123", "salesAgentId": "agent_456", "groupId": "ctx_123-group-0", "groupName": "Publisher Name", "bidPrice": 12.50, "budget": 5000 }] } }
```

Show the updated selection with selected products and budget allocation.

**Step 4: Confirm readiness**

"You have selected X products with $Y allocated. Ready to create the campaign?"

**Step 5: Create the campaign**

> ⚠️ **Every campaign is created in the advertiser's currency; an advertiser is single-currency.** Pass the advertiser's currency in `body.budget.currency` or omit it — never a different currency, the server rejects mismatches (400, field `budget.currency`). Always confirm the currency with the user before calling `create_campaign`. Never assume USD — always use the currency the user specified.

**Operation:** `create_campaign`
```http
POST /api/v2/buyer/campaigns
{
  "advertiserId": "12345",
  "name": "Q1 2025 EU CTV Campaign",
  "brief": "CTV campaign with a 50,000 euro budget.",
  "discoveryId": "abc123-def456-ghi789",
  "flightDates": {
    "startDate": "2025-02-01T00:00:00Z",
    "endDate": "2025-03-31T23:59:59Z"
  },
  "budget": {
    "total": 50000,
    "currency": "EUR"
  }
}
```
**As operation:**
```json
{ "operation": "create_campaign", "body": { "advertiserId": "12345", "name": "Q1 2025 EU CTV Campaign", "brief": "CTV campaign with a 50,000 euro budget.", "discoveryId": "abc123-def456-ghi789", "flightDates": { "startDate": "2025-02-01T00:00:00Z", "endDate": "2025-03-31T23:59:59Z" }, "budget": { "total": 50000, "currency": "EUR" } } }
```

**Step 6: Launch**

**Operation:** `execute_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/execute
```
**As operation:**
```json
{ "operation": "execute_campaign", "pathParams": { "campaignId": "<id>" } }
```

### Discovery Management

- `list_discovery_products` operation — List selected products
- `add_discovery_products` operation — Add products
- `remove_discovery_products` operation — Remove products (body: `{ "productIds": ["..."] }`)

**Summary:** Discover products → Select products → Create campaign → Execute

**IMPORTANT:** Do NOT expose API details to the user. Communicate conversationally about campaigns, inventory, products, and budgets — not about endpoints or HTTP methods.

---

## Performance Optimization Workflow

**When to use:** User wants the system to optimize for business outcomes automatically.

**Prerequisites:** Advertiser exists (with brand set during creation) + Event source configured.

**Step 1: Verify advertiser**

**Operation:** `list_advertisers`
```http
GET /api/v2/buyer/advertisers?status=ACTIVE&name={advertiserName}
```
**As operation:**
```json
{ "operation": "list_advertisers", "params": { "status": "ACTIVE", "name": "<advertiserName>" } }
```

**Step 2: Check/create event source**

**Operation:** `list_event_sources`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/event-sources
```
**As operation:**
```json
{ "operation": "list_event_sources", "pathParams": { "advertiserId": "<id>" } }
```

If empty, register one or more via the upsert/sync endpoint:
```http
POST /api/v2/buyer/advertisers/{advertiserId}/event-sources/sync
{
  "account": { "account_id": "{advertiserId}" },
  "event_sources": [
    { "event_source_id": "website_pixel", "name": "Website Pixel", "event_types": ["purchase", "add_to_cart"] }
  ]
}
```

Save the `event_source_id` — it's required for optimization goals. Note the snake_case field names: this endpoint is ADCP-aligned (the only event-sources mutation route is `/sync`; there is no per-source POST/PUT/DELETE).

**Step 3: Gather required fields from the user**

Before calling the create endpoint, confirm you have:
- Campaign name (ask the user or confirm a suggested name)
- Flight dates (start and end — ask the user)
- Budget total and currency (ask the user — the campaign is created in the advertiser's single currency; pass that currency or omit it, never a different one)
- Optimization goals: at least one goal with `kind` ("event" or "metric"). For event goals: event source ID (from Step 2), event type (e.g. `purchase`, `lead`), and optionally a `target` (e.g. `kind: "per_ad_spend"` for ROAS or `kind: "cost_per"` for CPA). For metric goals: `metric` string and optional `target`.
- Optional: constraints, attribution window, priority

**Step 4: Create the campaign with performanceConfig**

> ⚠️ **Every campaign is created in the advertiser's currency; an advertiser is single-currency.** Pass the advertiser's currency in `body.budget.currency` or omit it — never a different currency, the server rejects mismatches (400, field `budget.currency`). Always confirm the currency with the user before calling `create_campaign`. Never assume USD — always use the currency the user specified.

**Operation:** `create_campaign`
```http
POST /api/v2/buyer/campaigns
{
  "advertiserId": "12345",
  "name": "Q1 EU ROAS Optimization",
  "brief": "Performance campaign with a 100,000 euro budget.",
  "flightDates": {
    "startDate": "2025-02-01T00:00:00Z",
    "endDate": "2025-03-31T23:59:59Z"
  },
  "budget": {
    "total": 100000,
    "currency": "EUR"
  },
  "performanceConfig": {
    "optimizationGoals": [{
      "kind": "event",
      "eventSources": [
        { "eventSourceId": "es_abc123", "eventType": "purchase", "valueField": "value" }
      ],
      "target": { "kind": "per_ad_spend", "value": 4.0 },
      "attributionWindow": { "clickThrough": "7d" },
      "priority": 1
    }]
  },
  "constraints": {
    "channels": ["ctv", "display"],
    "countries": ["US"]
  }
}
```
**As operation:**
```json
{ "operation": "create_campaign", "body": { "advertiserId": "12345", "name": "Q1 EU ROAS Optimization", "brief": "Performance campaign with a 100,000 euro budget.", "flightDates": { "startDate": "2025-02-01T00:00:00Z", "endDate": "2025-03-31T23:59:59Z" }, "budget": { "total": 100000, "currency": "EUR" }, "performanceConfig": { "optimizationGoals": [{ "kind": "event", "eventSources": [{ "eventSourceId": "es_abc123", "eventType": "purchase", "valueField": "value" }], "target": { "kind": "per_ad_spend", "value": 4.0 }, "attributionWindow": { "clickThrough": "7d" }, "priority": 1 }] }, "constraints": { "channels": ["ctv", "display"], "countries": ["US"] } } }
```

`performanceConfig` must include `optimizationGoals` array with at least one goal. Each goal has a `kind` discriminator: `"event"` goals require `eventSources` array (each with `eventSourceId` and `eventType`); `"metric"` goals require `metric` string. Both kinds support an optional `target` object (`kind: "per_ad_spend"` for ROAS targets, `kind: "cost_per"` for CPA targets, each with a `value`), `attributionWindow`, and `priority`.

**Step 5: Auto-select products (optional)**

If the campaign has discovered products (from auto-discovery at creation), let the system pick the best ones:

**Operation:** `auto_select_products`
```http
POST /api/v2/buyer/campaigns/{campaignId}/auto-select-products
```
**As operation:**
```json
{ "operation": "auto_select_products", "pathParams": { "campaignId": "<id>" } }
```
Returns selected products with budget allocations and AI-generated rationale. Present results and let the user review before launching.

**Step 6: Launch**

**Operation:** `execute_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/execute
```
**As operation:**
```json
{ "operation": "execute_campaign", "pathParams": { "campaignId": "<id>" } }
```

---

## Account Management

Some account management tasks are handled in the web UI at [interchange.io](https://interchange.io). Direct users to these pages for:

| Task | URL | Capabilities |
|------|-----|--------------|
| **API Access** | [interchange.io/user-api-keys](https://interchange.io/user-api-keys) | Organization admins manage WorkOS API keys; secrets appear only once at creation |
| **Team Members** | [interchange.io/admin](https://interchange.io/admin) | Invite members, manage roles, manage advertiser access |
| **Billing** | Available from user menu in the UI | Manage payment methods, view invoices (via Stripe portal) |
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

### Current Account

Get the customer the authenticated user is currently operating in:

**Operation:** `get_current_account`
```http
GET /api/v2/accounts/current
```
**As operation:**
```json
{ "operation": "get_current_account" }
```

Returns `{ "id", "company", "name", "role" }` where `role` is the user's normalized role (`ADMIN`, `MEMBER`, or `SUPER_ADMIN`).

### List Accounts

List all customers the authenticated user has active membership on:

**Operation:** `list_accounts`
```http
GET /api/v2/accounts
```
**As operation:**
```json
{ "operation": "list_accounts" }
```

Returns `{ "accounts": [{ "id", "company", "name", "role" }] }` where `role` is the user's normalized role (`ADMIN`, `MEMBER`, or `SUPER_ADMIN`).

### Switch Account

Switching the active customer context is a session-mutating operation and is not available through `api_call`. Use the dedicated `customer_switch` MCP tool instead:

```json
{
  "tool": "customer_switch",
  "arguments": { "customerId": 123 }
}
```

Any user can switch to a customer they have an active membership on. SuperAdmin users can additionally switch to any customer. Use the `list_accounts` operation above to discover available customer IDs before switching. After the switch, all subsequent operations in the session are performed on behalf of the selected customer.

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

Returns `{ "data": { ... } }` — one consolidated commercial-account document: plan and contract status, ToS acceptance, effective pricing, intelligence usage, credit/prepay standing and balance, agreements, child accounts, and the single next action (if any) needed to become or remain paid. Org-admin only; a child-account request resolves to the parent organization. This is the same read model as the `get_billing_account` hand tool (which additionally accepts a `section` argument to return just one section); the operation exists so the Plan & Billing MCP-app widget can hydrate through `api_call` on external MCP-UI hosts.

### Organization IU Rate Card

When an administrator asks about IU prices or wants to choose a plan, call the
dedicated `get_iu_rate_card_offer` tool. It owns the shared portable Task; do
not imitate acceptance with `api_call`. Its server-owned response contains the
one organization plan, corporate discount, rollover policy, activity terms,
governing agreement, current binding, and history shared by buyer and
storefront workloads. The activated social account term is read from the
current offer—never quote its IU amount from memory. Only the bound MCP App may
invoke the app-only `accept_iu_rate_card_offer` after a direct human
organization administrator confirms. Connecting accounts remains available at
no cost. The activated-social-account rate is displayed for planning, but its
meter remains non-billing until separate charging ratification; never claim it
is currently charged. Existing accepted media pricing is separate; this IU
Rate Card does not amend those terms.

---

## Entity Hierarchy & Prerequisites

Before creating campaigns, you MUST understand the entity hierarchy:

```
Customer (your account)
  └── Advertiser (brand account - REQUIRED first)
        ├── Catalogs (sync product/offering data to partners)
        ├── Campaigns (advertising campaigns)
        │     └── Creatives
        ├── Event Sources (event data pipelines for optimization)
        └── Test Cohorts (for A/B testing)
```

### ⚠️ CRITICAL: Brand Domain Required for All Advertiser Actions

**Before performing ANY action on an advertiser** (creating campaigns, managing accounts, syncing catalogs, etc.), check that the advertiser has a brand domain configured (the `brand` field is not null/missing in the advertiser response).

If the advertiser does NOT have a brand domain:
1. **Do not proceed** with the requested action.
2. Inform the user: "This advertiser does not have a brand domain configured, which is required. Please provide a brand domain (e.g., `nike.com`) so I can update the advertiser."
3. Once the user provides one, update the advertiser via `update_advertiser` operation with `{ "brand": "nike.com" }`.
4. Only then continue with the original request.

### Setup Checklist

**Before you can run a campaign, you need:**

1. **Advertiser with brand domain** (REQUIRED)
   - First, check if one exists: `list_advertisers` operation
   - If not, create one: `create_advertiser` operation (requires `brand`)
   - An advertiser represents a brand/company you're advertising for
   - The brand is resolved automatically from the domain during creation
   - If the brand is not yet registered but enrichment succeeds, the API can create the advertiser with a `brandWarning`; show that warning/details and only use `saveBrand: true` when the user wants to persist the enriched brand to the registry
   - Set `sandbox: true` to create a sandbox advertiser — all ADCP operations will use sandbox-flagged accounts with no real spend. See **Sandbox Mode** below. Sandbox mode cannot be changed after creation.

   **Sandbox Mode**

   Sandbox mode lets you test the full media buying lifecycle — discovery, campaign creation, creatives, and delivery — without real platform calls or spending real money.

   - Sandbox is **account-level, not per-request**. The seller provisions a dedicated sandbox account, and every request using that `account_id` is automatically treated as sandbox. This eliminates the risk of accidentally mixing real and test traffic in a multi-step flow.
   - All discovered accounts for a sandbox advertiser are sandbox accounts — `list_accounts` is called with `sandbox: true`.
   - The correct sandbox `account_id` is automatically injected into `create_media_buy`, `get_media_buy_delivery`, and `get_products` — delivery and reporting data are fully scoped to the sandbox environment.
   - Responses contain simulated but realistic data.
   - Reference: https://docs.adcontextprotocol.org/docs/media-buy/advanced-topics/sandbox#sandbox-mode

2. **Event Sources** (REQUIRED for performance optimization)
   - Register event data pipelines: `POST /api/v2/buyer/advertisers/{advertiserId}/event-sources/sync` (ADCP-aligned snake_case body — see the Event Sources section below)
   - Referenced by `event_source_id` in optimization goals

3. **Creative Manifests** (REQUIRED)
   - Every campaign needs creative assets
   - **Asset creation and upload is UI-only** — call `GET /api/v2/buyer/creative-dashboard-url?advertiserId={advertiserId}&campaignId={campaignId}` to get the full URL, then direct the buyer to the returned `url`
   - Via MCP you can only **list**, **get**, **update metadata**, and **delete** existing manifests — see the **Creative Manifests** section below
   - Delivery to sales agents is automatic at campaign execute and re-syncs on later creative updates. There is no separate manual `sync_creatives` step for the buyer to call. How each seller receives creatives (`sync_creatives` vs inline) is chosen by the platform from the seller's capabilities; see the [creative object guide](/v2/object-guides/creative#creative-transport-modes) for transport details.

---

## Core Concepts

| Concept | Description | Required For |
|---------|-------------|--------------|
| **Advertiser** | Top-level account representing a brand/company | Everything |
| **Catalog** | Product/offering data synced to partner platforms via ADCP | Catalog sync |
| **Campaign** | Advertising campaign with budget, dates, targeting | Running ads |
| **Creative Manifest** | Campaign-scoped container for creative assets (images, videos, URLs). Created/uploaded via UI only; list/get/update/delete via MCP | Ad delivery |
| **Event Source** | Event data pipeline (pixel, SDK, etc.) | Performance optimization |
| **Syndication** | Push audiences/events/catalogs to ADCP agents | Audience distribution |
| **Test Cohort** | A/B test configuration | Experimentation |
| **Media Buy** | Executed purchase record — reduce budgets, cancel, and archive via `update_campaign` | Campaigns, Reporting |

---

## First-Time Setup

If you're starting fresh with a new advertiser, follow these steps.

```
Step 1: Check if an advertiser already exists
Operation: list_advertisers
→ If advertisers exist, you can use one. If not, create one (see Step 1b).

Step 1b: Create an advertiser (if needed)
Operation: create_advertiser
Body: { "name": "Acme Corp", "brand": "acme.com" }
→ If brand is registered: advertiser is created with linked brand.
→ If brand is not registered but enrichment succeeds: advertiser is created with a
  brandWarning. Show the warning/enriched details to the user. Do not retry create.
  Use saveBrand: true only when the user wants the enriched brand persisted to the
  registry.

Step 2: Create an event source (for performance optimization)
POST /api/v2/buyer/advertisers/{advertiserId}/event-sources
{
  "eventSourceId": "website_pixel",
  "name": "Website Pixel"
}

Step 3: Now you can discover products and create campaigns!
```

---

## API Endpoints Reference

### Advertisers

#### List Advertisers

**Operation:** `list_advertisers`
```http
GET /api/v2/buyer/advertisers?status=ACTIVE&name=Acme&linkedAccountPartnerId=snap&limit=10&offset=0&includeBrand=true
```
**As operation:**
```json
{ "operation": "list_advertisers", "params": { "status": "ACTIVE", "name": "Acme", "linkedAccountPartnerId": "snap", "limit": "10", "offset": "0", "includeBrand": "true" } }
```

**Query Parameters (Filters):**
- `status` (optional): Filter by status - `ACTIVE` or `ARCHIVED`
- `name` (optional): Filter by name (case-insensitive, partial match). Example: `name=Acme` matches "Acme Corp", "acme inc", etc.
- `linkedAccountPartnerId` (optional): Server-side filter for advertisers linked to at least one account from the given partner / sales agent ID (for example, `snap`).
- `limit` (optional): Maximum number of advertisers per page (default: 100, max: 100)
- `offset` (optional): Pagination offset (default: 0)
- `includeBrand` (optional): When `true`, include the linked brand summary/manifest details that Murph can use for creative assets before asking the buyer to upload logos or other brand files.

**Response:** each row is the advertiser **summary** shape — `id`, `name`, `status`, `brand`, `sandbox`, `linkedAccountCount`, `createdAt`, `updatedAt`. With `includeBrand=true`, summary rows may also include `linkedBrand` with resolved brand identity details such as logos, colors, tone, tagline, and attachable catalog/brand assets. `description`, `optimizationApplyMode`, linked partner accounts, UTM config, and frequency caps are NOT on the summary — call `get_advertiser` for the full resource.

**Response format:**
```json
{
  "items": [ ... ],
  "total": 42,
  "hasMore": true,
  "nextOffset": 10
}
```
Use `nextOffset` as the `offset` parameter for the next page. When `hasMore` is `false`, `nextOffset` is `null`.

**Display Requirements — ALWAYS include when listing advertisers:**

Present each advertiser as a structured entry (not prose). For every advertiser, show:
- **Name** and **ID**
- **Status** (ACTIVE, ARCHIVED)
- **Brand** — brand name or domain (show "No brand" if missing)
- **Sandbox** — Yes/No
- **Linked Accounts** — show `linkedAccountCount`. If `0`, say "No linked accounts" and offer to discover/link. For sandbox advertisers, do not mention linking accounts — sandbox accounts are provisioned automatically when the user has credentials for a sales agent. To enumerate the accounts (storefront/source the account is reachable through, account ID, status), call `get_advertiser`.

Never summarize into a sentence like "You have 13 advertisers." Always show the per-item details above for every advertiser in the response.

**Note:** `get_advertiser` returns the full advertiser — including resolved brand details, linked partner accounts, UTM config, and frequency caps.

#### Get Advertiser

**Operation:** `get_advertiser`
```http
GET /api/v2/buyer/advertisers/{id}
```
**As operation:**
```json
{ "operation": "get_advertiser", "pathParams": { "advertiserId": "<id>" } }
```

Returns the advertiser with full brand details, linked partner accounts, UTM config, and frequency caps.

**Response:**
```json
{
  "id": "34",
  "name": "Acme Corp",
  "description": null,
  "status": "ACTIVE",
  "createdAt": "2026-02-15T10:00:00Z",
  "updatedAt": "2026-02-15T10:00:00Z",
  "brand": "acme.com",
  "brandWarning": null,
  "linkedBrand": {
    "id": "brand_123",
    "name": "Acme Corp",
    "domain": "acme.com",
    "manifest": {
      "name": "Acme Corp",
      "logos": [{ "url": "https://acme.com/logo.png", "tags": ["primary"] }],
      "colors": { "primary": "#FF5733" },
      "industry": "Technology",
      "tagline": "Innovation for Everyone",
      "tone": "professional"
    },
    "logoUrl": "https://acme.com/logo.png",
    "industry": "Technology",
    "colors": { "primary": "#FF5733" },
    "tagline": "Innovation for Everyone",
    "tone": "professional"
  }
}
```

**linkedBrand fields:**
- `id`: Brand agent ID (prefixed with `brand_`)
- `name`: Resolved brand name
- `domain`: Brand domain
- `manifest`: Resolved brand identity data — includes logos, colors, fonts, tone, tagline, assets, product catalog, disclaimers, and more.
- `logoUrl`: Primary logo URL (convenience field extracted from manifest)
- `industry`: Brand industry (convenience field)
- `colors`: Brand colors (convenience field)
- `tagline`: Brand tagline (convenience field)
- `tone`: Brand tone (convenience field)

#### Create Advertiser

**⚠️ IMPORTANT: `brand` is required when creating an advertiser.**

When a user asks to create an advertiser:

1. **Research first when a domain is present** — if the user provides a brand domain or website URL, use available context and the API's brand enrichment before asking follow-up questions. Do not ask for the name or brand again when the user already provided them.
2. **Ask only for genuinely missing required choices** — `name` and `brand` are required. `sandbox` is optional but immutable, so ask real spend vs sandbox only if the user did not already say it.
3. **Confirm the advertiser's primary currency** — `primaryCurrency` is **required** (no default); it is the currency of every campaign under this advertiser, which is single-currency. Ask for the right ISO 4217 currency during setup; never assume USD. It can be changed via `update_advertiser` only until the advertiser's first campaign exists — after that it is locked.
4. **Do not ask about `optimizationApplyMode` during advertiser setup** — sub-campaign optimization is handled later in the campaign/media-buy flow. Leave it unset unless the user explicitly asks to configure advertiser/campaign approval mode or volunteers a clear `"AUTO"`/`"MANUAL"` preference.
5. **Create the advertiser** with the researched brand details and confirmed required fields.

The system resolves the brand via Addie (AdCP registry + Brandfetch enrichment). There are three possible outcomes:

**Outcome 1: Brand exists in the registry** — Advertiser is created successfully with the linked brand.

**Outcome 2: Brand found via enrichment only (not yet registered)** — The advertiser is created successfully, but the response includes `brandWarning` explaining that the brand was resolved by enrichment rather than an official registry entry. The response can include enriched brand data such as name, domain, manifest with logos, colors, industry, tagline, tone, etc.

**When this occurs, you MUST do BOTH of the following:**

1. **ALWAYS show the warning/enriched brand details.** Present `brandWarning` and any enriched brand fields returned — show the brand name, domain, industry, colors, logo URL, tagline, tone, and any other fields present. This lets the user review and confirm the right brand was found.
2. **Do not retry create.** The advertiser already exists. If the user wants the enriched brand persisted to the AdCP registry, tell them you can save it by updating the advertiser with `saveBrand: true` after they review the enrichment.

**Optional save to registry after review:**

**Operation:** `update_advertiser`
```http
PUT /api/v2/buyer/advertisers/{advertiserId}
{
  "brand": "acme.com",
  "saveBrand": true
}
```
**As operation:**
```json
{ "operation": "update_advertiser", "pathParams": { "advertiserId": "adv_123" }, "body": { "brand": "acme.com", "saveBrand": true } }
```
This saves the enriched brand to the AdCP registry and keeps the existing advertiser linked to the brand.

**Outcome 3: No registry/enrichment brand data found** — If the user confirms the advertiser name and brand domain are correct, retry with `saveBrand: true` instead of sending them away. This registers the brand identity from the confirmed domain/name so advertiser creation can continue. Do not invent logos, colors, industries, or other brand facts.

**Retry with confirmed brand identity:**

**Operation:** `create_advertiser`
```http
POST /api/v2/buyer/advertisers
{
  "name": "Acme Corp",
  "brand": "acme.com",
  "saveBrand": true
}
```
**As operation:**
```json
{ "operation": "create_advertiser", "body": { "name": "Acme Corp", "brand": "acme.com", "saveBrand": true } }
```
Only direct the user to external registration if there is no confirmed name/domain to save or the registry save fails.

**Initial request (without `saveBrand`):**

**Operation:** `create_advertiser`
```http
POST /api/v2/buyer/advertisers
{
  "name": "Acme Corp",
  "brand": "acme.com",
  "description": "Global advertising account"
}
```
**As operation:**
```json
{ "operation": "create_advertiser", "body": { "name": "Acme Corp", "brand": "acme.com", "description": "Global advertising account" } }
```

**Request fields:**
- `name` (required): Advertiser name
- `brand` (required): Brand domain (e.g., `"nike.com"`)
- `description` (optional): Description
- `primaryCurrency` (**required**, string, ISO 4217, no default): The advertiser's primary currency. An advertiser is single-currency: every campaign under it is created in this currency, and a campaign `budget.currency` must match it (the server rejects a mismatch). Contract rate cards do NOT affect campaign currency. Ask for this during setup; never assume USD. It can be changed via `update_advertiser` only until the advertiser's first campaign exists — after that it is locked.
- `saveBrand` (optional, boolean, default `false`): When `true`, saves the resolved or confirmed brand identity to the AdCP registry if the brand is not yet registered. Usually not needed for advertiser creation when enrichment succeeds; set this only after reviewing enrichment or when the user confirms registry persistence is desired.
- `linkedAccounts` (optional, array): Accounts to link at creation time. Each item: `{ storefrontId, sourceId, accountId, billingType? }`. Get `storefrontId` from `list_storefronts` and `sourceId` from `get_storefront_capabilities`. Use `list_available_accounts` operation to discover valid accountIds — never ask the user to provide one manually.
- `optimizationApplyMode` (optional, string): `"AUTO"` or `"MANUAL"` (default `"MANUAL"`). Controls whether Scope3 AI model optimizations to media buys are applied automatically or require manual approval for campaigns under this advertiser. Do not ask for this during advertiser setup; only set it when the user explicitly asks to configure approval mode or volunteers a clear preference.
- `utmConfig` (optional, array, max 20): Default UTM parameters for this advertiser. These are appended to landing page URLs during clickthrough redirection. Each item: `{ paramKey, paramValue }`. `paramKey` is the query parameter name (e.g. `"utm_source"`, `"bg_campaign"`). `paramValue` is a macro (e.g. `"{CAMPAIGN_ID}"`) resolved dynamically at click time, or a static string (e.g. `"scope3"`). Available macros follow the ADCP universal macros spec: https://docs.adcontextprotocol.org/docs/creative/universal-macros#universal-macros. If omitted, defaults are applied: `utm_source=scope3`, `utm_medium=agentic`, `utm_campaign={CAMPAIGN_ID}`, `utm_content={CREATIVE_ID}`, `utm_media_buy={MEDIA_BUY_ID}`, `utm_package={PACKAGE_ID}`. Campaign-level UTM config can override these per param key.
- `frequencyCaps` (optional, array): Buyer-side frequency caps that apply across all campaigns/creatives for this advertiser. Each item: `{ maxExposure, window: { interval, unit } }`. `maxExposure` is a positive integer (max exposures per user). `window.interval` is a positive integer; `window.unit` is `"minutes"`, `"hours"`, `"days"`, `"weeks"`, or `"months"`. Multiple caps are combined as AND (all must hold). Example: `[{ "maxExposure": 3, "window": { "interval": 1, "unit": "days" } }, { "maxExposure": 10, "window": { "interval": 1, "unit": "weeks" } }]` = max 3/day AND max 10/week. See the Frequency Caps section below for replace semantics.

**Response** includes `brand`, `linkedBrand`, `primaryCurrency`, `optimizationApplyMode`, optional `utmConfig` (advertiser-level UTM params, only present when configured), `frequencyCaps`, and optional `brandWarning` (e.g., if data came from Brandfetch enrichment rather than a well-known manifest).

#### Update Advertiser

**Operation:** `update_advertiser`
```http
PUT /api/v2/buyer/advertisers/{id}
{
  "name": "Acme Corporation",
  "description": "Updated description",
  "brand": "newbrand.com"
}
```
**As operation:**
```json
{ "operation": "update_advertiser", "pathParams": { "advertiserId": "<id>" }, "body": { "name": "Acme Corporation", "description": "Updated description", "brand": "newbrand.com" } }
```

**Optional fields:** `name`, `description`, `brand`, `primaryCurrency` (ISO 4217 — the advertiser's single currency; can be changed ONLY until the advertiser's first campaign exists, after which it is locked and the server rejects the change), `linkedAccounts` (array of `{ storefrontId, sourceId, accountId, billingType? }` to add — does not remove existing links), `optimizationApplyMode` (`"AUTO"` or `"MANUAL"` — controls whether Scope3 AI model optimizations to media buys are applied automatically or require manual approval for campaigns under this advertiser), `utmConfig` (array of `{ paramKey, paramValue }`, max 20 — replaces all existing advertiser-level UTM params; pass `[]` to clear), `frequencyCaps` (array of `{ maxExposure, window: { interval, unit } }` — **full replace semantics**: omit the field to leave caps unchanged; pass `[]` to clear all caps; pass the full array to replace the set). Discover valid accountIds via `list_available_accounts` operation — never ask the user to provide an account ID.

If `brand` is provided, the system resolves the new brand and updates the linked brand agent.

#### Link Agent Account to Advertiser

Use this workflow to discover and link an inventory-source agent's account to a
specific advertiser.

**Official adapter storefronts are not this workflow.** Built-in adapters such
as Amazon, Google, Meta, Pinterest, Reddit, Snap, Spotify, TikTok, OpenAI,
Gemini, fal.ai, ElevenLabs, Veo, and AudioStack are storefront dispatchers. They
are not source rows under another storefront, and the buyer agent should
not search public storefronts for a "Snap source", "OpenAI source", or "Google
source" to connect. Use `list_storefronts` with `limit: 100` to discover adapter
storefronts. Rows expose `adapterSourceKind`; creative generators have
`adapterSourceKind: "creative"` plus `creativeCapabilities` describing
modalities (`image`, `audio`, `video`), transformer ids, representative format
ids when known, and whether exact formats require `list_creative_formats`.
For fal.ai, treat fal.ai as the connected provider/gateway and the transformer
ID (for example `fal_flux`) as the specific model family/rendering path.
For final production, ask or infer where the creative will run and which
placements/formats matter. For early concept exploration, a
`start_creative_session` call can omit `target_format_id` and let the
server-side creative planner choose a draft fallback target format and
synthesize required text assets from the plain-language brief. Do not make the
buyer provide `target_format_id`, `transformer_id`, or adapter-specific manifest
JSON unless they explicitly request a specific technical format.
For AudioStack creative sessions, preserve requested duration and language from
the buyer's plain-language prompt. For example, "15-second ad in Dutch" should
select or pass the 15-second AudioStack format and carry Dutch as the spoken
language; do not treat those details as optional copy suggestions.
Use ElevenLabs for voiceover/text-to-speech drafts. If the buyer asks for a
finished audio ad with background music, mastering, or sound design, prefer
AudioStack or clearly explain that ElevenLabs will only produce the spoken
voiceover in the current creative workflow.
Rows also expose `adapterConnection`, so you can say which providers are
already connected and which need auth. If the user asks to connect to an
official adapter, call `connect_storefront` with the adapter storefront ID and
give the returned `connectionUrl` to the human. For OAuth providers that link
goes straight to the provider's consent screen; bearer-only providers land on
a first-party token-entry page. Do not ask the user to paste provider secrets
into chat. Use `list_storefront_connections` to check status
or `select_storefront_connection_account` if the provider returns multiple
accounts. `register_source_credentials`, `list_available_accounts`, and
advertiser `linkedAccounts` are for legacy or third-party external AdCP
inventory sources that appear in `get_storefront_capabilities`.

**Two-step process overview:**
1. **Register agent credentials** (customer-level) — done once per source via `register_source_credentials` operation
2. **Link account to advertiser** (advertiser-level) — discover available accounts for a specific agent and link one to an advertiser

**Prerequisites:** Agent credentials must already be registered for the relevant
inventory source via `register_source_credentials` operation (see Register
Agent Credentials in the Storefronts section). **Only external AdCP sources
where `get_storefront_capabilities` returns `requiresCredentials: true` support
account linking.** To find eligible sources, use `list_storefronts` for the
storefront ID, call `get_storefront_capabilities` for active source rows, and
inspect `requiresCredentials`. Do not use this scan to find official adapters
like Snap or Google; those are adapter storefronts, not nested sources.

**⚠️ CRITICAL: Multiple Credentials for the Same Agent**

A customer may register **multiple sets of credentials** for the same sales agent (e.g., two different Snap ad accounts with different API keys). This is fully supported. When this happens:
- Each set of credentials discovers its own set of ad accounts via `list_accounts`
- The `list_available_accounts` operation **requires a `credentialId` parameter** so the system knows which credential to use for discovery
- If the customer has multiple credentials and `credentialId` is omitted, the API returns a **validation error listing the available credential IDs** — present these to the user and ask them to pick
- Once the user picks a credential, re-call the discovery endpoint with `credentialId` to get that credential's accounts
- When the user links an account, all future operations for that advertiser+agent pair automatically use the credential associated with that account

**Workflow when multiple credentials exist:**
1. Use `list_agent_credentials` operation to list the customer's registered credentials
2. Present the credentials to the user (show `id` and `accountIdentifier` for each)
3. Ask the user which credential to use
4. Pass the chosen `credentialId` to the discovery endpoint

**Step 1 — Create advertiser with brand**

**Operation:** `create_advertiser`
```http
POST /api/v2/buyer/advertisers
{ "name": "Acme Corp", "brand": "acme.com" }
```
**As operation:**
```json
{ "operation": "create_advertiser", "body": { "name": "Acme Corp", "brand": "acme.com" } }
```

**Step 2 — Discover available accounts for the agent**

**⚠️ CRITICAL: Account IDs MUST come from the discovery endpoint — NEVER from user input.**
- You MUST call the discovery endpoint below and use ONLY the `accountId` values returned in the response.
- If the user provides an account ID or account name verbally (e.g., "the account ID is 06cd7033..."), do NOT use that value. Instead, call the discovery endpoint and match against the returned results.
- If no accounts are returned from discovery, tell the user no matching accounts were found. Do NOT pretend to link an account that was not returned by the API.
- **NEVER fabricate, guess, or use a user-provided account ID directly.** The only valid account IDs are those returned by this endpoint.

**Operation:** `list_available_accounts`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/accounts/available?storefrontId={storefrontId}&sourceId={sourceId}
GET /api/v2/buyer/advertisers/{advertiserId}/accounts/available?storefrontId={storefrontId}&sourceId={sourceId}&credentialId={credentialId}
```
**As operation:**
```json
{ "operation": "list_available_accounts", "pathParams": { "advertiserId": "<id>" }, "params": { "storefrontId": "1", "sourceId": "<sourceId>" } }
```

- **Identify the source via `storefrontId` + `sourceId`** (both required). Get `storefrontId` from `list_storefronts` and `sourceId` from `get_storefront_capabilities`.
- `credentialId` is **required when the customer has multiple credentials** registered for this agent. If omitted with multiple credentials, the API returns a validation error listing available credential IDs — present them to the user and ask which to use, then retry with `credentialId`.
- `credentialId` is optional when the customer has only one credential for the agent.
- Returns accounts filtered by the advertiser's brand domain for the specified agent and credential.
- Response includes `accounts` array with `accountId`, `name`, `house`, `advertiser`, `sources[]` (the `(storefrontId, sourceId, sourceName)` pairs that surface this account), and `billingOptions.supported`.
- **Show ALL discovered accounts to the user and let them pick.** Present each account's `name` (and `advertiser` if different from the brand).
- If `accounts` is empty, tell the user no matching accounts were found for that agent. Do NOT proceed with linking.
- **Do NOT ask about billing type.** Use `billingOptions.default` from the response if available. Only include `billingType` in the link request if the response shows multiple `billingOptions.supported` values AND no default — in that case, present the options and ask the user to choose.

**Step 3 — Link the selected account to the advertiser**

**Operation:** `update_advertiser`
```http
PUT /api/v2/buyer/advertisers/{advertiserId}
{
  "linkedAccounts": [
    { "storefrontId": 1, "sourceId": "src_main", "accountId": "acc_123" }
  ]
}
```
**As operation:**
```json
{ "operation": "update_advertiser", "pathParams": { "advertiserId": "<id>" }, "body": { "linkedAccounts": [{ "storefrontId": 1, "sourceId": "src_main", "accountId": "acc_123" }] } }
```
- Identify the source via `storefrontId` + `sourceId` on each entry (both required).
- The `accountId` here MUST be one returned from Step 2. Never use a value from any other source.
- `linkedAccounts` adds accounts — it does not remove existing links. Include `billingType` if the agent requires a specific billing arrangement.

---

### Campaigns

Campaigns use a single endpoint for creation and update. Configuration is done through action endpoints.

#### List Campaigns

**Operation:** `list_campaigns`
```http
GET /api/v2/buyer/campaigns?advertiserId=12345&status=ACTIVE
```
**As operation:**
```json
{ "operation": "list_campaigns", "params": { "advertiserId": "12345", "status": "ACTIVE" } }
```

**Query Parameters:**
- `advertiserId` (optional): Filter by advertiser
- `status` (optional): `DRAFT`, `ACTIVE`, `PAUSED`, `COMPLETED`, `ARCHIVED`

**Response:** each row is the campaign **summary** shape — `campaignId`, `advertiserId`, `name`, `status`, `flightDates`, `budget` (compact: `total` + `currency` only), `productCount`, `createdAt`, `updatedAt`. Brief, media buys, audiences, creative coverage, fees, pacing, performance config, frequency caps, and the full `budget` (with `dailyCap` / `pacing`) are NOT on the summary — call `get_campaign` for the full resource.

#### Get Campaign

**Operation:** `get_campaign`
```http
GET /api/v2/buyer/campaigns/{campaignId}
```
**As operation:**
```json
{ "operation": "get_campaign", "pathParams": { "campaignId": "<id>" } }
```

`get_campaign` returns the full campaign — including `brief`, `mediaBuys`, `audiences` (currently active audiences with `audienceId`, `name`, `status`, `type` (`"TARGET"` or `"SUPPRESS"`), `enabledAt`), `creativeFormats`, `frequencyCaps`, `pacingPeriods`, `performanceConfig`, `constraints`, `allocatedBudget`, `unallocatedBudget`, `products`, `storefronts` (hydrated storefront pin, see below), and the original `budget` object. There is no `fees` array and no `mediaBudget` object in the response — see below.

Each `mediaBuys[].products[]` entry includes an optional `formatOptions` field with the publisher's declared format requirements. Filter to `format_kind: "video_vast"` or `"video_hosted"` entries for video constraints; each `params` object may carry `duration_ms_exact`, `duration_ms_range`, `width`, `height`, and `sizes`. When present, use it to tell the buyer what rendition their creative must include: "This product requires a 30s VAST with a 640x480 rendition." We cannot inspect VAST renditions - the buyer must verify their creative meets the spec.

**Storefront pin (`storefronts`):** When the campaign was created or updated with `storefrontIds`, the response surfaces a `storefronts` array — each entry is `{ id, platformId, name }`, hydrated against `storefront_config` so the response is renderable without a follow-up lookup. Absent or empty means the campaign is not pinned to any specific storefronts. The input field on `create_campaign` / `update_campaign` is `storefrontIds` (just the IDs); the **response** uses the richer `storefronts` shape.

**Budget fields (returned by `get_campaign` whenever `budget.total` is set):**
- There is no `campaignType` field — routing (`DECISIONED` vs `ROUTED`) belongs to each media buy, derived server-side from the storefront it executes against, and is not returned on campaign or media buy responses. To tell whether a buy is routed, look at its storefront: routed-only (adapter) storefronts report `supportedRoutingTypes: ["ROUTED"]` on `list_storefronts` / `get_storefront`.
- The campaign has no `feeType` field and no `fees` array — every campaign budget is GROSS, and the media/fee split is itemized per media buy in its `budget_breakdown` (below).
- `allocatedBudget` — Sum of active media buy budgets plus performance spend on archived media buys, expressed in `budget.currency`. All gross — the same denomination as `budget.total`.
- `unallocatedBudget` — Budget remaining for new media buys, computed server-side as `budget.total` minus `allocatedBudget` (both gross). Use it directly — **do not sum media buy budgets yourself**. May be negative if archived performance spend exceeds `budget.total`.
- `mediaBuys[].budget_denomination` — `"gross"`: the buy's budgets (per-product and package) are fee-inclusive amounts in the buyer currency. Present only on buys with fee terms locked at creation — legacy buys carry neither this field nor `budget_breakdown`.
- `mediaBuys[].budget_breakdown` — read-only `{ media_budget, fee_amount, fee_rate_percent, effective_gross_cpm }`, derived at the fee terms locked when the buy was created. `media_budget` is the portion that buys media; `fee_amount` is the Scope3 fee inside the gross budget; `fee_rate_percent` is the locked fee rate; `effective_gross_cpm` is the buy's gross budget ÷ impression goal × 1000 — the all-in price per thousand impressions, so "budget ÷ CPM = impressions" holds on gross numbers (for a buy priced at a single fixed seller CPM, it works out to `seller CPM ÷ (1 − fee rate)`) — `null` when the buy has no positive impression goal or no gross budget. Omitted (along with `budget_denomination`) on legacy buys without locked fee terms. Never an input — budgets are set and updated in gross only, and updates re-split at the locked rate. When presenting the fee to a user, say "fee rate" or "fee terms" — never "margin".

The list summary exposes only the compact `budget: { total, currency }` (a 2-field nested pick of the full budget) — call `get_campaign` for `allocatedBudget`, `unallocatedBudget`, and the full budget object (with `dailyCap` / `pacing`).

**DRAFT campaigns only:** The response includes `discoveryId`, `products`, and `productCount` fields representing the product selection from the discovery workflow. These fields are **only present while the campaign is in DRAFT status**. After execution, product data is represented through `mediaBuys` — do not look for `discoveryId` or `products` on executed campaigns.

**Understanding duplicate media buys in the response:** The `mediaBuys` array may contain **two entries with the same media buy ID but different statuses** — e.g., one `ACTIVE` and one `PENDING_APPROVAL`. This is **normal and expected**. The `PENDING_APPROVAL` entry is a pending version of the media buy that has been submitted to the sales agent/publisher for approval. The original `ACTIVE` version remains unchanged until the publisher approves the update — once approved, the pending version becomes ACTIVE and replaces the old one. Do NOT flag this as unusual or ask the user about it — simply explain that the update is pending publisher approval.

**Surviving response truncation (`mediaBuyRefs` + `mediaBuyId` filter):** Campaigns with many media buys produce large `get_campaign` responses where the nested `mediaBuys[]` tail can be truncated by your context window. To handle this:

- `Campaign.mediaBuyRefs` is a small array placed near the **top** of the `get_campaign` response (`[{ mediaBuyId, status }, ...]`) covering every media buy on the campaign. Use it to enumerate IDs reliably even when the heavier `mediaBuys[]` array later in the response is truncated. **If the user asks for a specific media buy that you cannot find in `mediaBuys[]`, check `mediaBuyRefs` before claiming the buy doesn't exist.**
- `get_campaign` accepts a `mediaBuyId` query param (single value or repeated) that filters the embedded `mediaBuys[]` array to just the requested buys. The campaign object and `mediaBuyRefs` are unchanged: only the heavy nested array is narrowed. Use this to drill into specific buys without loading the full tree.
- When the question is specifically "what products are on this campaign / this media buy?", prefer `get_campaign_products` over `get_campaign`. It's a much smaller response (no budget tree, no creative coverage, no packages) and is the only surface that lists products with their human-readable `productName` / `publisherName` even when no discovery session is attached. Pass `mediaBuyId` to narrow it to a single buy.

```json
{ "operation": "get_campaign", "pathParams": { "campaignId": "<id>" }, "params": { "mediaBuyId": "mb_ETBn4gJ9Wu" } }
{ "operation": "get_campaign_products", "pathParams": { "campaignId": "<id>" }, "params": { "mediaBuyId": "mb_ETBn4gJ9Wu" } }
```

**Pacing periods (`pacingPeriods`):** When a campaign has `pacingPeriods` on the GET response, it defines one or more time-based segments — each with its own `label`, `start`, and `end` date — that together cover the campaign flight. Each period is typically executed as its own media buy (or set of packages). Media buy `start_time` and `end_time` MAY correspond to a specific pacing period's `start` / `end`, but pacing periods do not strictly govern media buy dates: a media buy can have any dates within the campaign flight, with or without pacing periods. A media buy's `start_time` can never be earlier than the campaign's `flightDates.startDate`, and `end_time` must fall within the campaign flight. When the user refers to a specific period or package name (e.g. "Package 1" or "the first flight"), match it to the `label` in `pacingPeriods` and use that period's `start` / `end` as the media buy dates.

#### Get Campaign Products

**Operation:** `get_campaign_products`
```http
GET /api/v2/buyer/campaigns/{campaignId}/products
GET /api/v2/buyer/campaigns/{campaignId}/products?mediaBuyId=mb_X
GET /api/v2/buyer/campaigns/{campaignId}/products?mediaBuyId=mb_X&mediaBuyId=mb_Y
```
**As operation:**
```json
{ "operation": "get_campaign_products", "pathParams": { "campaignId": "<id>" } }
{ "operation": "get_campaign_products", "pathParams": { "campaignId": "<id>" }, "params": { "mediaBuyId": "mb_ETBn4gJ9Wu" } }
```

Returns every ad product attached to the campaign with its human-readable `productName`, `publisherName`, `salesAgentName`, and the media buys it lives on. The source of truth is `media_buy_products` (the actual products attached to each media buy), with discovery context layered on when a discovery session is attached. Products staged via discovery but not yet executed onto a media buy are folded in as `pending` entries (skipped when `mediaBuyId` narrows the query to specific buys).

**Use this instead of paging through `get_campaign`'s nested `mediaBuys[]` tree** when a campaign has many products and the full payload exceeds your context window. With `mediaBuyId`, you can pull the products for a single buy without loading the rest of the campaign.

**Response shape:**
- `products[]`: one entry per product with `productId`, `productName`, `salesAgentId`, `salesAgentName`, `publisherDomain`, `publisherName`, `bidPrice`, `budget`, `pricingOptionId`, `pricingModel`, `selectedAt`, `searchContext` (`{ id, brief }` of the discovery run that surfaced it, when available), and `mediaBuys[]` (each `{ mediaBuyId, name, status }`; empty array means the product is staged but not yet executed). Multi-strategy campaigns may have the same product on multiple media buys, listed together here.
- `searchContexts[]`: one entry per distinct discovery run on this campaign with `id`, `brief`, `channels`, `countries`, `createdAt`, `productCount`. Empty when the campaign has no discovery session attached.
- `summary`: `{ totalProducts, productsOnMediaBuys, productsPending }`

#### Create Campaign

**⚠️ BEFORE creating a campaign: verify the advertiser has a brand domain.**
Use `get_advertiser` operation and check the `brand` field is not null/missing.
If it is missing, do NOT proceed — tell the user: "This advertiser doesn't have a brand domain configured. Please provide one (e.g. `nike.com`) so I can update it first." Then use `update_advertiser` operation with `{ "brand": "..." }` before continuing.

**⚠️ Every campaign is created in the advertiser's currency; an advertiser is single-currency.** Pass the advertiser's currency in `body.budget.currency` or omit it — never a different currency, the server rejects mismatches (400, field `budget.currency`). Always confirm the currency with the user before calling `create_campaign`. Never assume USD — always use the currency the user specified.

**Operation:** `create_campaign`
```http
POST /api/v2/buyer/campaigns
{
  "advertiserId": "12345",
  "name": "Q1 Campaign",
  "flightDates": {
    "startDate": "2025-02-01T00:00:00Z",
    "endDate": "2025-03-31T23:59:59Z"
  },
  "budget": {
    "total": 50000,
    "currency": "EUR"
  },
  "brief": "<<< ALWAYS include the ENTIRE brief from the client — never summarize or truncate >>>",
  "constraints": {
    "channels": ["ctv", "display"],
    "countries": ["US"]
  },
  "discoveryId": "optional-existing-discovery-id",
  "productIds": ["prod_123", "prod_456"],
  "audienceConfig": {
    "targetAudienceIds": ["aud_001", "aud_002"],
    "suppressAudienceIds": ["aud_003"]
  },
  "performanceConfig": {
    "optimizationGoals": [{
      "kind": "event",
      "eventSources": [
        { "eventSourceId": "es_abc123", "eventType": "purchase", "valueField": "value" }
      ],
      "target": { "kind": "per_ad_spend", "value": 4.0 },
      "priority": 1
    }]
  }
}
```

**Minimum required fields:**
- `advertiserId`: Advertiser ID
- `name`: Campaign name (1-255 chars)
- `flightDates`: Start and end dates
- `budget`: Total and currency

**Creation choices and optional fields:**
- Routing (`DECISIONED` vs `ROUTED`) is **not a creation input** — there is no `campaignType` field. Each media buy derives its routing server-side from the storefront it executes against: buys on adapter storefronts are ROUTED (direct buyer↔sales-agent execution on the customer's own pre-authorized credentials — Scope3 does not decision, optimize, or pace), and buys on every other storefront are DECISIONED (Scope3 runs decisioning/optimization). A single campaign can mix routed and decisioned media buys. Do not ask the user to choose DECISIONED vs ROUTED.
- Budget model: every campaign is GROSS. `budget.total` is the all-in amount the customer pays and the Scope3 fee is carved out of it server-side — there is no `mediaBudget.total` or `fees` field in the response to read the carve-out from directly; use `unallocatedBudget` (see **Get Campaign**) to know what's left for new media buys. There is no `feeType` field on the campaign — there is no fee-model choice to send or ask the buyer about.
- `brief`: Campaign brief. **MUST be the ENTIRE brief from the client — never summarize or truncate.**
- `constraints.channels`: Target channels (display, olv, ctv, social)
- `constraints.geo_countries` / `geo_countries_exclude`: ISO 3166-1 alpha-2 country codes (`countries` is accepted as a deprecated alias and normalized to `geo_countries`)
- `constraints.geo_regions` / `geo_regions_exclude`: ISO 3166-2 subdivisions (e.g. `"US-CA"`)
- `constraints.geo_metros` / `geo_metros_exclude`: metro targeting, `{ system, values }` objects (e.g. Nielsen DMAs)
- `constraints.geo_postal_areas` / `geo_postal_areas_exclude`: postal-area targeting, preferably `{ country, system, values }` objects (e.g. `{ country: "NL", system: "postal_code", values: ["1011"] }`). Use legacy `{ system, values }` only for deprecated country-fused systems.
- City names are not a supported targeting field. When a buyer names cities/towns, preserve the exact city list in `brief` and use a machine-readable overlay only when you can resolve the intent safely: `geo_postal_areas` for known postal areas, `geo_proximity` for radius/travel-time/store-area intent, or `geo_metros` only for declared metro systems. Do not invent `geo_cities`, do not pass raw city names as targeting, and call out ambiguous places (for example, Bergen or Laren) for confirmation.
- `constraints.language`, `constraints.device_type`, `constraints.device_platform`: AdCP language/device overlays
- `discoveryId`: Attach an existing discovery session
- `productIds`: Product IDs to pre-select from the discovery session (requires discoveryId)
- `storefrontIds`: Array of storefront IDs (from `list_storefronts`) the campaign is **pinned** to. When set, every `discover_products` run on this campaign auto-applies this filter — buyers don't need to resend it. **Highly encouraged** so the campaign only sources inventory from sellers the buyer has chosen. Empty array or omitted = no pin.
- `audienceConfig`: Audience targeting and suppression. `targetAudienceIds` (string array) — audiences to include. `suppressAudienceIds` (string array) — audiences to exclude. Audience IDs come from `list_audiences` operation.
- `performanceConfig`: Contains `optimizationGoals` array. Each goal has `kind` (`"event"` or `"metric"`). Event goals have `eventSources` array (each with `eventSourceId`, `eventType`, optional `valueField`), optional `target` (`kind: "per_ad_spend"` or `kind: "cost_per"` with `value`), optional `attributionWindow`, optional `priority`. Metric goals have `metric` string, optional `target`, optional `priority`.
- `optimizationApplyMode`: `"AUTO"` or `"MANUAL"` (default). Controls whether Scope3 AI model optimizations to media buys are applied automatically or require manual approval. Overrides the advertiser-level default.
- `utmConfig`: Campaign-level UTM parameter overrides. Object with `params` (array of `{ paramKey, paramValue }`, max 20) and optional `deleteMissing` (boolean — if `true`, removes campaign-level UTM params not in this request; if `false`/omitted, additive mode). Campaign UTM params override advertiser-level defaults per matching `paramKey`.
- `pacingPeriods`: Time-based pacing schedule. Divides the flight into labelled periods, each with its own `start` / `end` and either a `weight` (weight mode) or `budget` (budget mode). On execution, each selected product is split into one package per period. Media buys tied to a specific pacing period typically use that period's `start` / `end` as the media buy's `start_time` / `end_time`, but pacing periods do not strictly govern media buy dates — media buys can have any dates within the campaign flight, with or without pacing periods. Can only be set on DRAFT campaigns. Example: `{ "mode": "weight", "periods": [ { "label": "Package 1", "start": "2026-08-01", "end": "2026-08-09", "weight": 1 }, { "label": "Package 2", "start": "2026-08-10", "end": "2026-09-30", "weight": 2 } ] }`.
- `frequencyCaps`: Buyer-side frequency caps for this campaign. Array of `{ max_impressions, window: { interval, unit } }`. These are **campaign-scoped** caps (they override nothing at the advertiser level — enforcement evaluates all applicable caps). See the Frequency Caps section below for details and replace semantics. Example: `[{ "max_impressions": 3, "window": { "interval": 1, "unit": "days" } }]`.

**After creating a campaign, suggest ONLY these next steps (never mention strategies, tactics, or media plans):**
1. **Discover products** — find and attach inventory via `discover_products` operation
2. **Attach audiences** — link synced audiences for targeting/suppression via `update_campaign` with `audienceConfig`
3. **Set performance configuration** — configure optimization goals via `update_campaign` with `performanceConfig`

#### Update Campaign

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "name": "Updated Campaign Name",
  "budget": { "total": 75000 },
  "audienceConfig": {
    "targetAudienceIds": ["aud_004"],
    "suppressAudienceIds": ["aud_005"]
  },
  "performanceConfig": {
    "optimizationGoals": [{
      "kind": "event",
      "eventSources": [
        { "eventSourceId": "es_abc123", "eventType": "purchase", "valueField": "value" }
      ],
      "target": { "kind": "per_ad_spend", "value": 5.0 },
      "priority": 1
    }]
  }
}
```
**HARD RULE — `budget.total` is the client's committed number. NEVER raise it on your own initiative.** A request to add media buys, an `INSUFFICIENT_MEDIA_BUDGET` rejection, or a `minimumAllowedBudget` value in an error is NOT authorization to increase the budget. When a new or resized buy doesn't fit inside `unallocatedBudget`, free up allocation instead: lower, cancel, or delete other media buys — or present the shortfall to the buyer and ask how they want to proceed. Raise `budget.total` ONLY when the user explicitly instructs a budget increase in this conversation (e.g. "increase the campaign budget to $X"). "Add these buys" or "move budget to publisher X" always means rework the allocation WITHIN the existing total.

All fields are optional. `audienceConfig` is **additive** by default — it adds audiences without removing existing ones. Set `deleteMissing: true` inside `audienceConfig` to replace the full audience set (audiences not in the list are soft-disabled). To remove all audiences, send `{ "audienceConfig": { "deleteMissing": true } }`.

`frequencyCaps` uses **full-replace semantics**: omit the field to leave existing caps unchanged; pass `[]` to clear all caps; pass the full desired array to replace the set.

`storefrontIds` uses **full-replace semantics** for the campaign-level storefront pin: omit the field to leave the existing pin unchanged; pass `[]` to clear the pin (campaign is no longer scoped to specific storefronts); pass a non-empty array to replace the pinned set. Subsequent `discover_products` runs against this campaign auto-apply the new pin.

When `pacingPeriods` is included in the update, the response includes a `pacingCascadeResult` block with the per-media-buy outcome of pushing appended periods to live media buys via ADCP `add_packages`. Always inspect this when adding heavy-up periods to a running campaign — agents reported with `outcome: "unsupported"` need a manual new-media-buy workaround. See the Pacing Periods guide for the full result shape and append-only rules.

**Heavy-up on one media buy (per-buy pacing).** Campaign-level `pacingPeriods` shapes the *whole* portfolio: every paced media buy under the campaign follows that shape. When the buyer wants to heavy-up *one specific* media buy (e.g. a publisher with seasonal or event-driven inventory) without affecting the others, set `pacingPeriods` on the `mediaBuys[]` entry instead. The per-buy schedule replaces the campaign-level shape for that buy.

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "mediaBuys": [
    {
      "mediaBuyId": "mb_vox_concerts",
      "pacingPeriods": {
        "mode": "weight",
        "periods": [
          { "label": "Pre-tour",       "start": "2026-06-01", "end": "2026-06-30", "weight": 1.0 },
          { "label": "Tour heavy-up",  "start": "2026-07-01", "end": "2026-07-15", "weight": 3.0 },
          { "label": "Wind-down",      "start": "2026-07-16", "end": "2026-08-31", "weight": 0.5 }
        ]
      },
      "updated_reason": "Heavy-up on Vox concert inventory during the tour window"
    }
  ]
}
```

Bootstrapping pacing onto an unpaced **ACTIVE** buy is rejected (introducing period packages on top of live flat packages would double-spend). Allowed states: DRAFT / PENDING_APPROVAL (persisted, used at execute time) and ACTIVE / PAUSED already-paced buys with a strict append to the existing schedule (sent to the seller via `update_media_buy.new_packages`, requires `add_packages` capability).

When you append periods to a live paced media buy, the resulting per-period packages inherit the media buy's resolved `creative_ids` from the same `update_campaign` call. If the buyer expects specific creatives on the new windows that differ from the buy's existing assignments, set `creative_ids` on the same `mediaBuys[]` entry. Otherwise the auto-sync from the campaign's manifest applies.

##### Updating Media Buys via Campaign Update

To perform any media buy operation, include the `mediaBuys` array in the campaign update body. Each entry targets a specific media buy by ID and uses the `action` field to specify the operation. Media buy changes must fit within the campaign's existing `budget.total` — never raise the campaign budget to make room for a buy (see the HARD RULE under **Update Campaign**). For executed media-buy budget reductions, include the `packages[].packageId` + `packages[].budget` reductions AND the lower `budget.total` in ONE atomic request — lowering `budget.total` alone below current live allocations is rejected with `INSUFFICIENT_MEDIA_BUDGET` (package budgets are never auto-cut); the combined request is validated against the projected post-update allocation and applied atomically.

**Products vs packages — what you can update by status:**
- When a media buy is **DRAFT** (not yet executed), update its **`products`** — you can add, remove, or update existing products (budget, pacing, bidPrice).
- Once a media buy has executed and has deployed packages (**ACTIVE**, **PAUSED**, **COMPLETED**, **INPUT_REQUIRED**, or a **PENDING_APPROVAL** version staged on top of one of these), `products[].budget`/`pacing`/`bidPrice` is rejected: these fields are informational only once packages exist and never reach the seller. Use `packages[].packageId` + `budget`/`pacing`/`bidPrice` instead. Fetch `packageId` via `get_campaign` with `includeProductDetails=false` (avoids response truncation on buys with many creatives). You still cannot add or remove products in these statuses.
- **PENDING_APPROVAL** buys that have NOT yet executed (no deployed packages) still accept product field updates the same as DRAFT; adding or removing products is blocked either way.

**Example — Update budget and pacing for an ACTIVE media buy's package:**

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "mediaBuys": [
    {
      "mediaBuyId": "mb_abc123",
      "packages": [
        {
          "packageId": "pkg_xyz",
          "budget": 5000,
          "pacing": "even"
        }
      ],
      "updated_reason": "Increase budget for Q2 push"
    }
  ]
}
```

**Example — Update a DRAFT media buy's products:**

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "mediaBuys": [
    {
      "mediaBuyId": "mb_draft456",
      "products": [
        {
          "productId": "prod_001",
          "budget": 3000,
          "pacing": "even"
        }
      ]
    }
  ]
}
```

**Reducing campaign budget when an ACTIVE media buy is allocated (atomic reduce):**

If the campaign has an executed (ACTIVE) media buy, reducing the campaign budget alone will fail with `INSUFFICIENT_MEDIA_BUDGET` because the current media buy allocation exceeds the new campaign budget. Do NOT attempt to update the media buy first and then the campaign - include both changes in a single request. The server validates the campaign budget against the post-update media buy allocations when both are in the same call.

To find the `packageId`, call `GET /api/v2/buyer/campaigns/{campaignId}` and look in `mediaBuys[].packages[].packageId`. If the packages array is truncated, use the `mediaBuyId` query param to fetch just that buy.

**Example — Reduce campaign budget from $100 to $50 while reducing the active media buy package from $96 to $48:**

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "budget": {
    "total": 100
  },
  "mediaBuys": [
    {
      "mediaBuyId": "mb_abc123",
      "packages": [
        {
          "packageId": "pkg_xyz",
          "budget": 48
        }
      ],
      "updated_reason": "Reducing campaign budget"
    }
  ]
}
```

**Media buy update fields:**
- `mediaBuyId` (required): ID of the media buy to update (from campaign GET response `mediaBuys` array)
- `name` (optional): Updated media buy name
- `packages` (optional, for **ACTIVE** media buys): Array of package updates. Each: `packageId` (required), `budget`, `pacing` (`"even"` or `"asap"`), `bidPrice`, `creative_ids`
- `pacingPeriods` (optional): Per-media-buy pacing schedule with the same shape as `campaign.pacingPeriods` (`mode` + `periods[]`). When set, replaces the campaign-level pacing shape for this specific buy: use it to heavy-up one media buy without affecting the others. On **DRAFT** / **PENDING_APPROVAL** buys, the schedule is persisted and used at execute time to split packages. On **ACTIVE** / **PAUSED** buys, only strict appends to an already-paced buy are supported (sent to the seller via `update_media_buy.new_packages`, requires `add_packages` capability). Bootstrapping pacing onto an unpaced live buy is rejected, create a new media buy instead. Pass `null` to clear an existing per-buy schedule (deployed period packages keep running).
- `products` (optional): Array of product updates. Adding or removing products requires **DRAFT** status. Updating existing product fields (`budget`, `pacing`, `bidPrice`) is allowed on ACTIVE/PAUSED buys without pausing delivery. Each: `productId` (required), `pricingOptionId`, `budget`, `pacing` (`"asap"`, `"even"`, `"front_loaded"`), `bidPrice`, `remove` (boolean, optional)
- `start_time` (optional): `"asap"` or ISO 8601 date-time. Start of **this media buy**. **Cannot be earlier than the campaign's `flightDates.startDate`.** Media buy dates MAY correspond to a `pacingPeriods[].start` when the media buy represents a specific period, but pacing periods do not govern media buy dates — a media buy can have any start within the campaign flight, with or without pacing periods.
- `end_time` (optional): ISO 8601 date-time. End of **this media buy**. Must fall within the campaign's flight dates. Media buy dates MAY correspond to a `pacingPeriods[].end` when the media buy represents a specific period, but pacing periods do not govern media buy dates — a media buy can have any end within the campaign flight, with or without pacing periods.
- `creative_ids` (optional): Updated creative assignments
- `optimization_goals` (optional): Media-buy-level optimization goals applied to every package at execution time. See **Media Buy Optimization Goals** below. Pass an empty array to clear all goals. **For decisioned media buys only — ALWAYS ask the buyer what they want to optimize for before creating or updating one; do not fall back silently. For a routed media buy (one on an adapter storefront), optimization goals do not apply — do not prompt for them.**
- `updated_reason` (optional): Reason for update (stored with version history)

##### Media Buy Optimization Goals

**Optimization goals tell downstream optimizers what to tune for.** A media buy without optimization goals cannot be auto-optimized by the Scope3 AI model or any sales-agent-side optimizer. **You MUST ask the buyer what they want to optimize for** when creating or executing a decisioned media buy — never fall back to a default silently. For a routed media buy (one on an adapter storefront), optimization goals do not apply; skip this prompt entirely.

Optimization goals are stored on the media buy and fanned out to every package at execution time (submitted per-package in the ADCP `create_media_buy` request).

**Goal schema.** Each goal is one of two shapes:

1. **Event goal** (`kind: "event"`) — tied to registered event sources (e.g., a conversion pixel, purchase feed, lead form). `event_sources` is an **array of `{ event_source_id, event_type }` objects**, not top-level fields:
   ```json
   {
     "kind": "event",
     "event_sources": [
       { "event_source_id": "es_website_pixel", "event_type": "purchase" }
     ],
     "target": { "kind": "per_ad_spend", "value": 4.0 }
   }
   ```

2. **Metric goal** (`kind: "metric"`) — tied to a built-in delivery metric (e.g., clicks, views, video completions):
   ```json
   {
     "kind": "metric",
     "metric": "clicks",
     "target": { "kind": "cost_per", "value": 0.50 }
   }
   ```

**Target kinds** (differ by goal kind):

For **event goals** (`kind: "event"`):
- `cost_per` — CPA target (target cost per conversion event in the buy currency)
- `per_ad_spend` — ROAS target (return on ad spend multiple, e.g., `4.0` for 4× return). Requires at least one event source entry to include `value_field`.
- `maximize_value` — maximize total conversion value within budget (no `value` field; requires `value_field` on at least one event source entry)

For **metric goals** (`kind: "metric"`):
- `cost_per` — target cost per metric unit (e.g., CPC, CPM)
- `threshold_rate` — minimum per-impression rate (e.g., minimum click-through or completion rate)

**Example — Update a media buy with a ROAS goal tied to purchase conversions:**

**Operation:** `update_campaign`
```http
PUT /api/v2/buyer/campaigns/{campaignId}
{
  "mediaBuys": [
    {
      "mediaBuyId": "mb_abc123",
      "optimization_goals": [
        {
          "kind": "event",
          "event_sources": [
            {
              "event_source_id": "es_website_pixel",
              "event_type": "purchase",
              "value_field": "order_total"
            }
          ],
          "target": { "kind": "per_ad_spend", "value": 4.0 }
        }
      ],
      "updated_reason": "Shift to ROAS target for Q2"
    }
  ]
}
```

**Rules:**
- Event goals require the `event_source_id` to be registered under the advertiser first (see `list_event_sources` / create event source).
- Pass an empty array (`"optimization_goals": []`) to clear all goals.
- Optimization goals are applied to **every** package in the media buy at execution. Per-package goal overrides are not supported at this level.

**Media buy update versioning:** When you update an ACTIVE media buy, the system creates a **new pending version** with status `PENDING_APPROVAL`. This is expected:
- The original ACTIVE version remains unchanged until the sales agent (publisher) approves the update.
- The campaign GET response will temporarily show **two entries** for the same media buy: the original ACTIVE version and the new PENDING_APPROVAL version.
- The pending version's `packages` may not immediately reflect the requested changes — the updated values are submitted to the sales agent for approval. Product changes are not allowed on PENDING_APPROVAL media buys.
- Once the sales agent approves, the pending version becomes ACTIVE and replaces the old one.
- **Do NOT treat this as an error or try alternative approaches.** Simply inform the user that the update has been submitted and is pending approval from the sales agent/publisher.

#### Media Buy Cancel and Archive via Campaign Update

All media buy operations go through the campaign update endpoint. Use the `action` field in the `mediaBuys` array:

##### Cancel a Media Buy
```json
{
  "mediaBuys": [{
    "action": "cancel",
    "mediaBuyId": "mb_123",
    "reason": "No longer needed",
    "packageIds": ["pkg_1", "pkg_2"]
  }]
}
```

- Works for ACTIVE, PAUSED, PENDING_APPROVAL, and INPUT_REQUIRED statuses
- For DRAFT media buys, use `action: "delete"` instead (DRAFT is internal-only, not sent to any sales agent)
- For active media buys, a cancellation request is sent to the sales agent via ADCP
- `reason` (optional): Cancellation reason sent to the sales agent
- `packageIds` (optional): Cancel specific packages instead of the entire media buy

##### Archive a Media Buy
```json
{
  "mediaBuys": [{
    "action": "delete",
    "mediaBuyId": "mb_123"
  }]
}
```

Archives a media buy (soft delete). Sets status to ARCHIVED and removes it from active listings. Data is preserved for historical reporting.

##### Mix Actions in One Update
You can combine update, cancel, and delete actions in a single campaign update:
```json
{
  "mediaBuys": [
    { "action": "cancel", "mediaBuyId": "mb_1", "reason": "Budget cut" },
    { "action": "delete", "mediaBuyId": "mb_2" },
    { "mediaBuyId": "mb_3", "packages": [{ "packageId": "pkg_1", "budget": 5000 }] }
  ]
}
```

---

#### Delete Campaign

**Operation:** `delete_campaign`
```http
DELETE /api/v2/buyer/campaigns/{campaignId}
```
**As operation:**
```json
{ "operation": "delete_campaign", "pathParams": { "campaignId": "<id>" } }
```

---

#### Campaign Action Endpoints

#### Execute Campaign (Launch)

**BEFORE executing:** Confirm that every DRAFT **decisioned** media buy has `optimization_goals` set. A buy's routing is derived from its storefront and is not returned on campaign or media buy responses, so key this check off the storefront each buy targets: a buy on an adapter storefront (e.g. Snap, Google, Meta — its storefront reports `supportedRoutingTypes: ["ROUTED"]` on `list_storefronts` / `get_storefront`) is routed, and every other buy is decisioned. Decisioned media buys without goals cannot be auto-tuned — if any is missing goals, ALWAYS ask the buyer what they want to optimize for (ROAS target with a purchase event? CPC target? completion rate?) and set them via `update_campaign` → `mediaBuys` → `optimization_goals` first. Do not execute silently with missing goals. Skip this check for routed media buys — they pass through to the external platform, which handles its own optimization; `optimization_goals` do not apply. A campaign can mix both kinds of buy, so apply the check per media buy, not per campaign.

**Operation:** `execute_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/execute
```
**As operation:**
```json
{ "operation": "execute_campaign", "pathParams": { "campaignId": "<id>" } }
```

**Optional request body:**
```json
{
  "debug": true
}
```

**Response:**
```json
{
  "campaignId": "campaign_abc123",
  "previousStatus": "DRAFT",
  "newStatus": "ACTIVE",
  "success": true
}
```

**On partial failure** (some media buys failed to execute):
```json
{
  "campaignId": "campaign_abc123",
  "previousStatus": "DRAFT",
  "newStatus": "ACTIVE",
  "success": false,
  "errors": [
    {
      "mediaBuyId": "mb_xyz",
      "salesAgentId": "snap_abc",
      "message": "Failed to submit media buy to publisher: ...",
      "debug": {
        "request": { "...full ADCP create_media_buy request..." },
        "response": { "...full ADCP response from sales agent..." },
        "debugLogs": [ { "...A2A request/response logs..." } ],
        "error": "error message"
      }
    }
  ]
}
```

- `success` is `false` when any media buy execution failed
- `errors` array contains structured error objects per failed media buy
- `debug` field contains the same debug info as v1 `execute_media_buy` (full ADCP request, response, and A2A debug logs) — only present when `debug: true` was sent in the request body
- Campaign is still set to ACTIVE even with partial failures — re-execute to retry failed media buys

**How execute works internally:**
1. **Reconcile** — reads the attached discovery session's selected products and creates DRAFT media buys per sales agent. Products already on an existing non-terminal media buy are skipped. Each sales agent gets its own media buy. If the agent already has a DRAFT, new products are added to it. If the agent has no media buy or only terminal ones (CANCELLED, ARCHIVED, COMPLETED, REJECTED), a new DRAFT is created.
2. **Execute DRAFTs** — all DRAFT media buys on the campaign are submitted to their respective sales agents.
3. Existing non-DRAFT media buys (ACTIVE, PAUSED, etc.) are **never touched** — execute only operates on DRAFTs.

This means you can safely re-execute a campaign after attaching a new discovery session. New products for new sales agents get their own media buys without affecting existing ones.

**Note on Media Buys:** Media buys are child resources of campaigns — there are **no standalone media buy endpoints**. DRAFT media buys are created automatically when a discovery session is attached (via `update_campaign` with `discoveryId`) or when `execute_campaign` is called. They are included in campaign GET responses (`mediaBuys` array) and modified through campaign updates (`update_campaign` operation with `mediaBuys` array). For ACTIVE media buys, update `packages` for deployed line items or `products` to adjust existing product budgets/pacing without pausing delivery (adding/removing products still requires DRAFT). See "Updating Media Buys via Campaign Update" above for full schema and examples.

#### Pause Campaign

Pause an active campaign. Cascades to all active media buys and returns the per-media-buy outcome (`mediaBuyResults`, `successCount`, `failureCount`). `failureCount > 0` means the campaign transitioned to `PAUSED` but one or more media buys failed to pause at the ADCP level — inspect `mediaBuyResults` entries where `success: false`.

**Operation:** `pause_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/pause
```
**As operation:**
```json
{ "operation": "pause_campaign", "pathParams": { "campaignId": "<id>" } }
```

#### Reactivate Campaign

Reactivate a paused campaign. The campaign must be `PAUSED`. Cascades to all paused media buys and returns the per-media-buy outcome (`mediaBuyResults`, `successCount`, `failureCount`).

**Operation:** `reactivate_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/reactivate
```
**As operation:**
```json
{ "operation": "reactivate_campaign", "pathParams": { "campaignId": "<id>" } }
```

#### Auto-Select Products

**Operation:** `auto_select_products`
```http
POST /api/v2/buyer/campaigns/{campaignId}/auto-select-products
```
**As operation:**
```json
{ "operation": "auto_select_products", "pathParams": { "campaignId": "<id>" } }
```
No request body. Automatically selects products from the campaign's discovery session and allocates budget based on measurability. Replaces any previous selections. Requires a performance campaign (`performanceConfig` set) with discovered products.

**Response:**
- `selectedProducts` (array): Products with budget allocations (`productId`, `salesAgentId`, `budget`, `cpm`, `pricingOptionId`)
- `budgetContext` (object): `campaignBudget`, `totalAllocated`, `remainingBudget`, `currency`
- `selectionRationale` (string): Explanation of the selection strategy
- `selectionMethod` (string): `"scoring"`, `"measurability"`, or `"cpm_heuristic"`
- `testBudgetPerProduct` (number, optional): Test budget allocated per product
- `productCount` (number): Total products selected

#### Get Media Buy ADCP Status

Refresh or read the current ADCP status of all media buys in a campaign. Direct sales-agent and delegated-adapter buys are polled during the request; storefront-routed buys are checked from persisted upstream-leg state. Updates local status when changes are detected (e.g., pending → active, active → paused). Use this to check whether pending media buys have been activated or to detect status transitions that webhooks may have missed.

**Operation:** `get_media_buy_status`
```http
GET /api/v2/buyer/campaigns/{campaignId}/media-buy-status
```
**As operation:**
```json
{ "operation": "get_media_buy_status", "pathParams": { "campaignId": "<id>" } }
```
No request body. Returns current ADCP status for each media buy in the campaign. Direct sales-agent and delegated-adapter buys are polled during the request; storefront-routed buys are checked from persisted upstream-leg state because one buyer-facing storefront buy can fan out to multiple upstream sources.

**Response:**
- `campaign_id` (string): The campaign ID queried
- `campaign_status` (string): Stored campaign lifecycle status
- `operational_status` (string): Rolled-up delivery status to present to users (`draft`, `pending_creatives`, `pending_start`, `active`, `paused`, `completed`, `attention_required`, or `no_media_buys`)
- `media_buys` (array): Status for each media buy
  - `media_buy_id` (string): Internal media buy ID
  - `adcp_media_buy_id` (string): ADCP-assigned media buy ID
  - `internal_status` (string): Current internal status (ACTIVE, PENDING_APPROVAL, PAUSED, COMPLETED, etc.)
  - `adcp_status` (string|null): Current ADCP status (active, pending_creatives, pending_start, paused, completed, rejected, canceled)
  - `operational_status` (string): Per-buy delivery status behind the campaign rollup
  - `previous_internal_status` (string): Status before this check
  - `previous_adcp_status` (string|null): ADCP status before this check
  - `updated` (boolean): Whether this call changed Interchange's stored status. `false` can still be current.
  - `status_refresh_source` (string): One of `direct_agent`, `delegated_adapter`, `storefront_route_rollup`, `not_submitted`, or `refresh_failed`
  - `blockers` (array): Per-buy reasons delivery is not ready
- `agents_queried` (number): Number of direct sales-agent or delegated-adapter status calls made during this request. It can be `0` for a current storefront-routed buy when the per-buy `status_refresh_source` is `storefront_route_rollup`.
- `errors` (array): Any errors encountered per media buy
- `blockers` (array): Campaign-level reasons delivery is not ready
- `pending_creative_reviews` (number): Count of creative reviews still waiting
- `has_upstream_media_buy` (boolean): Whether any media buy has an upstream ADCP media-buy ID
- `has_delivery` (boolean): Whether the campaign currently has an active delivery signal

Interpretation rule: use `has_delivery`, `has_upstream_media_buy`, `operational_status`, and each buy's `status_refresh_source` as the delivery signal. Do not treat `agents_queried: 0` or per-buy `updated: false` alone as stale status or broken dispatch; a storefront route rollup can confirm `ACTIVE` / `has_delivery: true` without a direct agent query in that same response.

Storefront-routed media buys additionally carry why-visibility fields when forwarding state exists:
  - `pending_reason` (string, optional): Why the buy is not delivering yet, rolled up to the most-blocking wait (`awaiting_storefront_approval`, `awaiting_source_moderation`, `creative_processing_at_source`, `awaiting_creative_approval`, `forward_failed_retrying`, `forward_failed_needs_correction`, `accepted_awaiting_trafficking`, `scheduled_not_started`). An annotation — never a status.
  - `pending_since` (string, optional): When the current wait began (ISO 8601)
  - `error_code` (string, optional): Buyer-safe error code (`product_no_longer_available`, `source_rejected`, `storefront_rejected`, `source_unavailable`, `invalid_request`, `quote_expired`, `platform_error`)
  - `error_owner` (string, optional): Who owns the fix (`buyer_input`, `platform`, `seller`)
  - `source_message` (string, optional): The source's sanitized rejection/moderation message
  - `forwarded_at` (string, optional): When the buy was forwarded to its source(s) (ISO 8601)
  - `buyer_reference` (string, optional): Support reference (`sf:<storefrontId>:<mediaBuyId>`) to quote to the seller or Scope3 support

#### Get Media Buy (why is my buy stuck)

Fetch a single media buy with its why-visibility annotation. Use this FIRST when a buyer asks why a buy is stuck or not delivering — it answers in one call instead of campaign archaeology. Use `get_campaign` for packages, products, and delivery.

**Operation:** `get_media_buy`
```http
GET /api/v2/buyer/media-buys/{mediaBuyId}
```
**As operation:**
```json
{ "operation": "get_media_buy", "pathParams": { "mediaBuyId": "<id>" } }
```
No request body.

**Response** (`mediaBuy` object):
- `mediaBuyId`, `name`, `status`, `startTime`, `endTime`, `createdAt`, `updatedAt`
- `pendingAt` (optional): Layer the buy is parked at while `PENDING_APPROVAL` (`storefront` / `salesagent` / `unknown`)
- `pendingReason` (optional): Why it is not delivering yet (same vocabulary as above, camelCase surface)
- `pendingSince` (optional): When the current wait began
- `errorCode` / `errorOwner` (optional): Buyer-safe failure code and who owns the fix
- `sourceMessage` (optional): The source's sanitized rejection/moderation text
- `forwardedAt` (optional): When the buy was forwarded to its source(s)
- `buyerReference` (optional): Support reference to quote (`sf:<storefrontId>:<mediaBuyId>`)



---

### Discovery

#### Discover Products

Discovers products based on advertiser context and returns a discoveryId for managing selections.

**Operation:** `discover_products`
```http
POST /api/v2/buyer/discovery/discover-products
{
  "advertiserId": "12345",
  "channels": ["ctv", "display"],
  "countries": ["US"],
  "brief": "<<< ALWAYS include the ENTIRE brief from the client here — never summarize >>>",
  "publisherDomain": "example",
  "storefrontIds": [42, 57],
  "debug": true
}
```
**As operation:**
```json
{ "operation": "discover_products", "body": { "advertiserId": "12345", "channels": ["ctv", "display"], "countries": ["US"], "brief": "...", "publisherDomain": "example", "storefrontIds": [42, 57], "debug": true } }
```

**Filtering Parameters:**
- `publisherDomain` (optional): Filter by publisher domain (exact domain component match)
- `pricingModel` (optional): Filter by pricing model (`cpm`, `vcpm`, `cpc`, `cpcv`, `cpv`, `cpp`, `flat_rate`)
- `storefrontIds` (optional, array of integers): Filter to these storefronts. Highly encouraged when scoping to specific sellers. IDs come from `list_storefronts`.
- `storefrontNames` (optional, array of strings): Filter to storefronts whose name matches (case-insensitive substring).

**Debug Parameter:**
- `debug` (optional, boolean): When `true`, includes detailed ADCP agent request/response debug logs in the response. Returns an `agentResults` array with per-agent success/failure status, raw response data, and full HTTP request/response logs (authorization headers redacted). Same structure as v1 `media_product_discover` debug output.

#### Discover Products for Existing Session

**Operation:** `browse_discovery`
```http
GET /api/v2/buyer/discovery/{discoveryId}/discover-products?groupLimit=10&groupOffset=0&productsPerGroup=15
```
**As operation:**
```json
{ "operation": "browse_discovery", "pathParams": { "discoveryId": "<id>" }, "params": { "groupLimit": "10", "groupOffset": "0", "productsPerGroup": "15" } }
```

**Query Parameters (Pagination):**
- `groupLimit` (optional): Max product groups (default: 10, max: 10)
- `groupOffset` (optional): Groups to skip (default: 0)
- `productsPerGroup` (optional): Max products per group (default: 10, max: 15)
- `productOffset` (optional): Products to skip within each group (default: 0)

**Query Parameters (Filtering):**
- `publisherDomain` (optional): Filter by publisher domain (exact component match). "hulu" matches "hulu.com" but "hul" does not
- `pricingModel` (optional): Filter by pricing model (`cpm`, `vcpm`, `cpc`, `cpcv`, `cpv`, `cpp`, `flat_rate`)
- `storefrontIds` (optional, comma-separated integers): Filter to storefront ID(s) from `list_storefronts`
- `storefrontNames` (optional, comma-separated strings): Filter to storefront name(s) (case-insensitive substring match)
- `debug` (optional): When `true`, includes ADCP agent request/response debug logs in the response (see debug section below)

Filters can be combined. Example: `?publisherDomain=example&pricingModel=cpm&storefrontIds=42,57`

**Response:**
```json
{
  "discoveryId": "abc123-def456-ghi789",
  "productGroups": [
    {
      "groupId": "group-0",
      "groupName": "Publisher Name",
      "products": [
        {
          "productId": "product_123",
          "name": "Premium CTV Inventory",
          "channel": "ctv",
          "bidPrice": 12.50,
          "salesAgentId": "agent_456",
          "publisherProperties": [
            { "publisherDomain": "hulu.com", "selectionType": "all" },
            { "publisherDomain": "espn.com", "selectionType": "by_id" }
          ]
        }
      ],
      "productCount": 5,
      "totalProducts": 20,
      "hasMoreProducts": true
    }
  ],
  "totalGroups": 25,
  "hasMoreGroups": true,
  "summary": {
    "totalProducts": 150,
    "publishersCount": 25,
    "priceRange": { "min": 5.0, "max": 25.0, "avg": 12.5 }
  },
  "budgetContext": {
    "sessionBudget": 50000,
    "allocatedBudget": 0,
    "remainingBudget": 50000
  }
}
```

**Product fields:**
- `publisherProperties` (array, optional): Publisher domains and targeting details for this product. Each product can have multiple publishers. Each entry contains `publisherDomain` (string) and `selectionType` (`"all"` or `"by_id"`). Use this to understand which publishers a product targets.

**Debug response** (when `debug: true`):

The response includes an `agentResults` array containing only failed agents with full ADCP request/response logs for troubleshooting:
```json
{
  "agentResults": [
    {
      "agentId": "agent_789",
      "agentName": "Failed Agent",
      "success": false,
      "productCount": 0,
      "error": "Connection timeout",
      "rawResponseData": { "..." },
      "debugLogs": [
        {
          "timestamp": "2026-03-18T10:00:00Z",
          "type": "request",
          "request": { "method": "POST", "url": "...", "headers": { "authorization": "[REDACTED]" }, "body": { "..." } },
          "response": { "status": 500, "body": { "..." } }
        }
      ]
    }
  ]
}
```

**Pagination:**
- `hasMoreGroups`: Use `groupOffset` to fetch more groups
- `hasMoreProducts`: Use `productOffset` to fetch more products within a group
- To paginate products for a single group, combine `productOffset` with a filter (`storefrontIds` or `storefrontNames`) to isolate that group

#### Add Products to Selection

**Operation:** `add_discovery_products`
```http
POST /api/v2/buyer/discovery/{discoveryId}/products
{
  "products": [
    {
      "productId": "product_123",
      "salesAgentId": "agent_456",
      "groupId": "ctx_123-group-0",
      "groupName": "Publisher Name",
      "bidPrice": 12.50,
      "budget": 5000
    }
  ]
}
```
**As operation:**
```json
{ "operation": "add_discovery_products", "pathParams": { "discoveryId": "<id>" }, "body": { "products": [{ "productId": "product_123", "salesAgentId": "agent_456", "groupId": "ctx_123-group-0", "groupName": "Publisher Name", "bidPrice": 12.50, "budget": 5000 }] } }
```

**Required per product:** `productId`, `salesAgentId`, `groupId`, `groupName`
**Optional per product:** `bidPrice` (required when `isFixed: false`), `budget`

**Response:**
```json
{
  "discoveryId": "abc123-def456-ghi789",
  "products": [
    {
      "productId": "product_123",
      "salesAgentId": "agent_456",
      "bidPrice": 12.50,
      "budget": 5000,
      "selectedAt": "2025-02-01T10:00:00Z",
      "groupId": "ctx_123-group-0",
      "groupName": "Publisher Name"
    }
  ],
  "totalProducts": 1,
  "budgetContext": {
    "sessionBudget": 50000,
    "allocatedBudget": 5000,
    "remainingBudget": 45000
  }
}
```

#### Get Selected Products

**Operation:** `list_discovery_products`
```http
GET /api/v2/buyer/discovery/{discoveryId}/products
```
**As operation:**
```json
{ "operation": "list_discovery_products", "pathParams": { "discoveryId": "<id>" } }
```

Response format same as Add Products.

#### Remove Products from Selection

**Operation:** `remove_discovery_products`
```http
DELETE /api/v2/buyer/discovery/{discoveryId}/products
{
  "productIds": ["product_123", "product_456"]
}
```
**As operation:**
```json
{ "operation": "remove_discovery_products", "pathParams": { "discoveryId": "<id>" }, "body": { "productIds": ["product_123", "product_456"] } }
```

Response format same as Add Products (with updated list).

#### Refine Discovery Results

Iterate on previous discovery results by calling `POST /discovery/discover-products` with the `discoveryId` and a `refine` array. Supports three refinement scopes:
- **request**: Direction for the overall discovery (e.g., "more video options", "focus on sports content")
- **product**: Target a specific product — `include`, `omit`, or `more_like_this`
- **proposal**: Target a specific proposal — `include`, `omit`, or `finalize`

Requires a previous `POST /discovery/discover-products` call (results must be cached). The `discoveryId` field is required when `refine` is provided.

```http
POST /api/v2/buyer/discovery/discover-products
{
  "advertiserId": "12345",
  "discoveryId": "abc123-def456-ghi789",
  "refine": [
    { "scope": "request", "ask": "show me more video options" },
    { "scope": "product", "id": "product_123", "action": "more_like_this" },
    { "scope": "product", "id": "product_456", "action": "omit" },
    { "scope": "proposal", "id": "proposal_789", "action": "include", "ask": "increase budget allocation" }
  ],
  "groupLimit": 10,
  "productsPerGroup": 10,
  "debug": false
}
```

**Refine item fields:**
- `scope` (required): `"request"`, `"product"`, or `"proposal"`
- `ask` (required for request-scope, optional for product/proposal): Natural language direction
- `id` (required for product/proposal-scope): The product or proposal ID from the previous discovery response
- `action` (required for product-scope): `"include"`, `"omit"`, or `"more_like_this"`
- `action` (required for proposal-scope): `"include"`, `"omit"`, or `"finalize"`

**Pagination parameters:** Same as browse (`groupLimit`, `productsPerGroup`, `groupOffset`, `productOffset`).

**Response:** Same structure as discover-products, plus an optional `refinementApplied` array showing what the sales agents did with each refinement instruction:
```json
{
  "discoveryId": "abc123-def456-ghi789",
  "productGroups": [ ... ],
  "totalGroups": 10,
  "hasMoreGroups": false,
  "summary": { ... },
  "refinementApplied": [
    { "scope": "request", "status": "applied", "notes": "Added video content filter" },
    { "scope": "product", "id": "product_456", "status": "applied", "notes": "Removed from results" }
  ]
}
```

**Workflow:** Call discover-products with refine, present updated results, let the user iterate or proceed to select products. Each refine call replaces the cached results, so subsequent browse/pagination operates on the refined set.

---

### Event Sources

Event data pipelines (website pixels, mobile SDKs, warehouse syncs, CRM feeds, etc.) registered at the advertiser level. Referenced by `eventSourceId` in campaign optimization goals.

Event sources are managed through sync (bulk upsert following the ADCP spec). There is no per-source REST mutation.

#### Sync Event Sources

Sync is the preferred way to manage event sources — it uses upsert semantics (creates or updates as needed).

**Operation:** `sync_event_sources`
```http
POST /api/v2/buyer/advertisers/26/event-sources/sync
{
  "account": { "account_id": "26" },
  "event_sources": [
    {
      "event_source_id": "website_pixel",
      "name": "Website Pixel",
      "event_types": ["purchase", "add_to_cart"],
      "allowed_domains": ["shop.example.com"]
    },
    {
      "event_source_id": "mobile_sdk",
      "name": "Mobile App SDK",
      "event_types": ["app_install", "purchase"]
    }
  ],
  "delete_missing": false
}
```

**URL path:** `/advertisers/{advertiserId}/event-sources/sync` — the `{advertiserId}` is the numeric advertiser ID (e.g. `26`). Also include it in the request body as `account.account_id`.

**`account` (required in body):**
- `account_id` (string): The advertiser ID — same value as the path `{advertiserId}` (e.g. `"26"`).

**`event_sources` array (required, 1–50 items). Each object:**
- `event_source_id` (string, required): Buyer-assigned identifier, referenced by optimization goals
- `name` (string, optional): Human-readable label
- `event_types` (array, optional): ADCP/IAB ECAPI event types this source handles. When omitted, accepts all types. Values: `page_view`, `view_content`, `select_content`, `select_item`, `search`, `share`, `add_to_cart`, `remove_from_cart`, `viewed_cart`, `add_to_wishlist`, `initiate_checkout`, `add_payment_info`, `purchase`, `refund`, `lead`, `qualify_lead`, `close_convert_lead`, `disqualify_lead`, `complete_registration`, `subscribe`, `follow`, `content_view`, `watch_milestone`, `start_trial`, `app_install`, `app_launch`, `contact`, `schedule`, `donate`, `submit_application`, `custom`
- `allowed_domains` (array, optional): Domains authorized to send events
- `integration_platform` (string, optional): Scope3 extension for the source system, such as Hightouch, CRM, MMP, or server API
- `mapping` (object, optional): Scope3 extension that records source fields used to produce `log_event` payloads. Useful keys include `eventIdField`, `eventTypeField`, `eventTimeField`, `userMatchFields`, `valueField`, `currencyField`, `orderIdField`, `contentIdsField`, `consentField`, `dedupeStrategy`, and `notes`
- `test_event_code` (string, optional): Default test code to use while validating the integration

**Other optional fields:**
- `delete_missing` (boolean): Archive event sources not included in this request (default: false)

**Response (200):**
```json
{
  "data": {
    "event_sources": [
      { "event_source_id": "website_pixel", "action": "created" },
      { "event_source_id": "mobile_sdk", "action": "updated" }
    ]
  }
}
```

Actions: `created`, `updated`, `unchanged`, `failed`, `deleted`

#### List Event Sources

**Operation:** `list_event_sources`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/event-sources
```
**As operation:**
```json
{ "operation": "list_event_sources", "pathParams": { "advertiserId": "<id>" } }
```

**Query Parameters:**
- `take` / `skip` (optional): Pagination

**Response fields to use:**
- `eventSources[].eventSourceId`, `name`, `eventTypes`, `allowedDomains`
- `eventSources[].integrationPlatform` — Scope3 metadata such as `Hightouch`, `CRM`, `MMP`, or `server API` when it has been synced/backfilled
- `eventSources[].mapping` — source-to-`log_event` field contract, including event ID/type/time, user match fields, value/currency/order/content fields, consent field, dedupe strategy, and notes
- `eventSources[].testEventCode` — default test code when present
- `eventSources[].health` — `status`, latest accepted event timestamps/type, and latest source error when known

#### Create / Update / Delete Event Sources

There is no per-source REST mutation. Use **`POST .../event-sources/sync`** (documented above) — it accepts an array of event sources and treats each as upsert-by-`event_source_id`. Pass `delete_missing: true` to archive sources omitted from the request.

---

### Event Summary

Get hourly-aggregated event counts for an advertiser. Use this to verify that events (impressions, clicks, conversions, etc.) are being ingested before setting up optimization goals.

**Important:** Event data is aggregated hourly. Newly reported events may take up to 1 hour to appear in this summary. If the user has just started reporting events, let them know to wait before checking.

#### Get Event Summary

**Operation:** `get_event_summary`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/events/summary
```
**As operation:**
```json
{ "operation": "get_event_summary", "pathParams": { "advertiserId": "<id>" } }
```

**Query Parameters:**
- `eventType` (string, optional): Filter by event type — one of `impression`, `click`, `conversion`, `measurement`, `mmp`. When omitted, returns all types.
- `startHour` (string, optional): Start of query range (inclusive), hour-aligned ISO 8601 (e.g. `2026-03-27T14:00:00Z`). Defaults to start of last completed UTC hour.
- `endHour` (string, optional): End of query range (exclusive), hour-aligned ISO 8601. Defaults to end of last completed UTC hour.

**Response (200):**
```json
{
  "data": {
    "periodStart": "2026-03-27T14:00:00.000Z",
    "periodEnd": "2026-03-27T15:00:00.000Z",
    "entries": [
      {
        "eventHour": "2026-03-27T14:00:00.000Z",
        "eventType": "impression",
        "eventCount": 1500
      },
      {
        "eventHour": "2026-03-27T14:00:00.000Z",
        "eventType": "conversion",
        "eventCount": 25
      }
    ],
    "totalEventCount": 1525
  }
}
```

The response includes both advertiser-specific events and customer-level shared events (events not tied to a specific advertiser but shared across all advertisers under the same customer).

---

### Log Event

Log conversion and marketing events for attribution. Events are forwarded to the tracking endpoint (CAPI). Requires an event source registered via sync_event_sources.

#### Log Events

**Operation:** `log_event`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/log-event
{
  "event_source_id": "website_pixel",
  "events": [
    {
      "event_id": "txn_abc123",
      "event_type": "purchase",
      "event_time": "2026-03-15T14:30:00-05:00",
      "action_source": "website",
      "event_source_url": "https://example.com/checkout",
      "user_match": {
        "hashed_email": "a1b2c3d4e5f6...",
        "click_id": "abc123",
        "click_id_type": "gclid"
      },
      "custom_data": {
        "value": 99.99,
        "currency": "USD",
        "order_id": "order_456",
        "content_ids": ["prod_789"],
        "num_items": 2
      }
    }
  ],
  "test_event_code": "TEST123"
}
```

**Request body:**
- `event_source_id` (string, required): Event source registered via sync_event_sources
- `events` (array, required, 1–10,000 items): Events to log
- `test_event_code` (string, optional): Test code for validation without affecting production data

**Each event object:**
- `event_id` (string, required): Unique identifier for deduplication
- `event_type` (enum, required): ADCP/IAB ECAPI event type. Values: `page_view`, `view_content`, `select_content`, `select_item`, `search`, `share`, `add_to_cart`, `remove_from_cart`, `viewed_cart`, `add_to_wishlist`, `initiate_checkout`, `add_payment_info`, `purchase`, `refund`, `lead`, `qualify_lead`, `close_convert_lead`, `disqualify_lead`, `complete_registration`, `subscribe`, `follow`, `content_view`, `watch_milestone`, `start_trial`, `app_install`, `app_launch`, `contact`, `schedule`, `donate`, `submit_application`, `custom`
- `event_time` (string, required): When the event occurred (ISO 8601 with timezone)
- `action_source` (enum, optional): `website`, `app`, `in_store`, `phone_call`, `system_generated`, `other`
- `event_source_url` (string, optional): URL where the event occurred
- `custom_event_name` (string, optional): Name for custom events (when event_type is `custom`)
- `user_match` (object, optional): User identity for attribution matching
  - `uids` (array): Universal ID values (`{type, value}` — rampid, id5, uid2, euid, pairid, maid)
  - `hashed_email`: SHA-256 of lowercase trimmed email
  - `hashed_phone`: SHA-256 of E.164 phone number
  - `click_id` / `click_id_type`: Platform click identifier
  - `client_ip` / `client_user_agent`: For probabilistic matching
- `custom_data` (object, optional): Event-specific data
  - `value`: Monetary value
  - `currency`: ISO 4217 code (e.g. `USD`)
  - `order_id`: Transaction identifier
  - `content_ids`: Product identifiers
  - `content_type`: Category (product, service, etc.)
  - `num_items`: Item count
  - `contents`: Array of `{id, quantity, price, brand}`

**Response (200):**
```json
{
  "data": {
    "events_received": 1,
    "events_processed": 1,
    "partial_failures": [],
    "warnings": [],
    "match_quality": 0.85
  }
}
```

---

### Measurement Data

Sync advertiser performance measurement data as an alternative to CAPI. Accepts time-series metric data over date ranges keyed by campaign, media buy, package, and/or creative. Uses upsert semantics — re-submitting the same data is safe and idempotent.

#### Sync Measurement Data

**Operation:** `sync_measurement_data`
```http
POST /api/v2/buyer/advertisers/26/measurement-data/sync
{
  "measurements": [
    {
      "start_time": "2026-03-01T00:00:00-05:00",
      "end_time": "2026-03-07T23:59:59-05:00",
      "metric_id": "incremental_revenue",
      "metric_value": 8450.75,
      "unit": "currency",
      "currency": "USD",
      "campaign_id": "camp_456"
    }
  ]
}
```

**URL path:** `/advertisers/{advertiserId}/measurement-data/sync` — the `{advertiserId}` is the numeric advertiser ID (e.g. `26`).

**`measurements` array (required, 1–1000 items). Each object:**
- `start_time` (string, required): Start of the measurement period (ISO 8601 with timezone)
- `end_time` (string, required): End of the measurement period (ISO 8601 with timezone, must be after start_time)
- `metric_id` (enum, required): `revenue`, `incremental_revenue`, `conversions`, `incremental_conversions`, `page_view_count`, `add_to_cart_count`, `purchase_count`, `ltv_1d`, `ltv_7d`, `ltv_30d`
- `metric_value` (number, required): Measured value for this metric
- `unit` (enum, required): `currency`, `count`, `ratio`, `percentage`
- `currency` (string, conditional): 3-letter uppercase ISO 4217 code (e.g. `"USD"`) — required when `unit` is `"currency"`
- `advertiser_id` (string, optional): Advertiser identifier
- `campaign_id` (string, optional): Campaign identifier
- `media_buy_id` (string, optional): Media buy identifier
- `package_id` (string, optional): Package identifier
- `creative_id` (string, optional): Creative identifier
- `source` (string, optional): Source of the measurement data
- `source_platform` (string, optional): Platform the data originates from
- `external_row_id` (string, optional): External row identifier for idempotency

**Constraint:** At least one of `advertiser_id`, `campaign_id`, `media_buy_id`, `package_id`, or `creative_id` must be provided.

**Response (200):**
```json
{
  "data": {
    "measurements": [
      { "index": 0, "action": "created" }
    ]
  }
}
```

Actions: `created`, `updated`, `unchanged`, `failed`

---

### Measurement Engine

Configure measurement sources, upload measurement and context records, and check freshness of incoming outcome data.

#### List Measurement Sources

**Operation:** `list_measurement_sources`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/measurement-sources
```
**As operation:**
```json
{ "operation": "list_measurement_sources", "pathParams": { "advertiserId": "<id>" } }
```

Optional query params: `outcomeType`, `status`, `take`, `skip`

#### Create Measurement Source

**Operation:** `create_measurement_source`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/measurement-sources
{
  "sourceKey": "mmm_sales",
  "name": "MMM Sales Lift",
  "outcomeType": "sales_volume",
  "granularity": "dma_week",
  "cadence": "weekly",
  "provider": "analytic-partner"
}
```
**As operation:**
```json
{ "operation": "create_measurement_source", "pathParams": { "advertiserId": "<id>" }, "body": { "sourceKey": "mmm_sales", "name": "MMM Sales Lift", "outcomeType": "sales_volume", "granularity": "dma_week", "cadence": "weekly", "provider": "analytic-partner" } }
```

Required: `sourceKey`, `name`, `outcomeType`, `granularity`, `cadence` (continuous, daily, weekly, biweekly, monthly, quarterly), `provider`

Optional: `lagWeeks` (default 1), `ingestionMethod`, `attributionConfig`, `signalWeight` (0-1, default 1.0), `status` (pending, active, paused), `notes`

#### Upload Measurement Records

**Operation:** `upload_measurement_records`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/measurement-records
{
  "records": [
    {
      "outcomeType": "sales_volume",
      "geo": "Atlanta",
      "timeWindowStart": "2026-01-06",
      "timeWindowEnd": "2026-01-12",
      "value": 1250,
      "source": "mmm_sales"
    }
  ]
}
```
**As operation:**
```json
{ "operation": "upload_measurement_records", "pathParams": { "advertiserId": "<id>" }, "body": { "records": [{ "outcomeType": "sales_volume", "geo": "Atlanta", "timeWindowStart": "2026-01-06", "timeWindowEnd": "2026-01-12", "value": 1250, "source": "mmm_sales" }] } }
```

Each record requires: `outcomeType`, `geo`, `timeWindowStart` (YYYY-MM-DD), `timeWindowEnd` (YYYY-MM-DD), `value`, `source`

Optional per record: `baselineValue`, `confidenceInterval`, `lagDays`

Accepts 1-5000 records per request. Uses upsert semantics (safe to re-submit).

#### List Measurement Records

**Operation:** `list_measurement_records`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/measurement-records
```

Optional query params: `outcomeType`, `geo`

#### Upload Context Records

**Operation:** `upload_context_records`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/context-records
{
  "records": [
    {
      "geo": "Atlanta",
      "timeWindowStart": "2026-01-06",
      "timeWindowEnd": "2026-01-12",
      "promoActive": true,
      "promoType": "BOGO"
    }
  ]
}
```

Each record requires: `geo`, `timeWindowStart`, `timeWindowEnd`

Optional: `promoActive` (default false), `promoType`, `temperatureAvg`, `competitorActivity`, `seasonalityIndex`, `flightStatus` (active, dark, pre_flight, post_flight)

#### Get Measurement Freshness

**Operation:** `get_measurement_freshness`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/measurement-freshness?flightStart=2026-01-06&geos=Atlanta,Richmond
```

Required query params: `flightStart` (YYYY-MM-DD), `geos` (comma-separated)

Optional: `flightEnd` (YYYY-MM-DD)

Returns per-source, per-geo freshness windows showing which time periods have data and which are missing.

---

### Test Cohorts

For A/B testing campaign variations.

#### List Test Cohorts

**Operation:** `list_test_cohorts`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/test-cohorts
```
**As operation:**
```json
{ "operation": "list_test_cohorts", "pathParams": { "advertiserId": "<id>" } }
```

#### Create Test Cohort

**Operation:** `create_test_cohort`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/test-cohorts
{
  "name": "Q1 Creative Test",
  "description": "Testing new vs old creatives",
  "splitPercentage": 50
}
```
**As operation:**
```json
{ "operation": "create_test_cohort", "pathParams": { "advertiserId": "<id>" }, "body": { "name": "Q1 Creative Test", "description": "Testing new vs old creatives", "splitPercentage": 50 } }
```

---

### Creative Manifests (Campaign-Scoped)

Creative manifests live under campaigns — they are always scoped to a specific campaign.

**Dashboard URL**: Call `GET /api/v2/buyer/creative-dashboard-url?advertiserId={advertiserId}&campaignId={campaignId}` to get a fully-resolved creative dashboard URL. The response contains a single `url` field — use it directly. Never construct URLs manually. The URL varies by environment and includes the customer ID automatically.

---

#### Bring Your Own Creative (in-chat creative-intent prompts)

When the buyer **brings or uploads a finished creative they already have** — they attach an image/video/tag, or say things like "add this as a creative", "I have creative for {advertiser}", "add this creative to a campaign", or "attach these" — your FIRST action is the creative-intent card. Do **NOT** call `get_my_campaigns`, `get_my_advertisers`, or `list_campaigns` to enumerate campaigns and then ask in prose — that is the wrong move; the `get_creative_intent` card IS the campaign picker and lists the campaigns for you. Do NOT jump to `start_creative_session` either (that flow is for *generating* new creative from a brief, not for placing creative the buyer already has). Route to the buyer-creative-v2 prompts:

- **No campaign chosen yet** (the buyer named an advertiser but not a campaign): call `buyer_api_call` with `operation: "get_creative_intent"` and the `advertiserId`. It returns the in-chat "which campaign are these for?" card — recent campaigns to pick from, a new-campaign option, and a "save to {advertiser} for now" option — plus a referenceable saved-as name. Let the card collect the choice; do not list the campaigns yourself in text.
- **Campaign chosen (existing or just-created) AND the buyer brought creative**: call `buyer_api_call` with `operation: "attach_creatives_to_campaign"`, the `campaignId`, and a `body.assets` array — one entry per brought attachment, each `{ "data_url": "murph-attachment://N" }` using the current turn's placeholder(s) (add `"name"` with the filename when you know it). This **actually attaches** the creatives to the campaign and returns the "here's what I added to {campaign}" card (creatives + format coverage). This is the operation that makes the attach real — do not just show the card.
- **Re-showing the card without attaching** (e.g. the buyer asks "show me {campaign}'s creatives again"): call `operation: "get_creative_confirmation"` with the `campaignId`. This is **read-only** — it lists what's already on the campaign and attaches nothing. Never use it as the way to attach brought creative; use `attach_creatives_to_campaign` for that.
- **Finding a creative brought or uploaded on an earlier turn, or on the web platform** (e.g. "attach the creative I uploaded earlier", "use the one from the library"): call `operation: "list_advertiser_creatives"` with the `advertiserId`. It returns every creative saved on that advertiser — evergreen/reference creatives promoted to the library ("the shelf") AND flight-specific creatives not yet attached to a campaign, so a web-uploaded creative always shows up here even before it's promoted. Pass `queryParams.promoted=true` only when the buyer explicitly wants just the reusable library, not a specific brought-but-unattached upload.
- Only tell the buyer a creative was attached if the card actually lists it. If the card is empty, the attach did not happen — say so and retry `attach_creatives_to_campaign` rather than claiming success.
- If any of these operations returns `NOT_FOUND`, the buyer-creative-v2 surface is off for this account — fall back to asking which campaign in plain chat.

Bringing an existing/finished creative → `get_creative_intent` (no campaign yet) / `attach_creatives_to_campaign` (campaign chosen) / `get_creative_confirmation` (read-only re-show) / `list_advertiser_creatives` (find a creative brought earlier or uploaded on the web). Generating new creative from a brief → creative session (below).

---

#### Agent-Native Creative Build Flow

The MCP interface can create and manage campaign-scoped creative manifests. Prefer creative sessions when the buyer asks to generate, iterate, upload, attach, preview, or refine creatives. Creative sessions are designed for the real human workflow: draft gallery → pick a direction → natural-language refinement → evaluator checks → final campaign creative.

1. Discover the campaign creative formats/templates with `list_creative_templates`.
2. Handle references before the session. If the buyer's request already signals references — explicitly ("use references"), or by naming a catalog / past creative / brand asset / "like the {campaign} campaign" — open the reference picker directly. If the request gives NO reference signal, ask once, in plain text, whether to ground on references ("want to ground this on references from your library, or jump straight to concepts?"): on yes, open the picker; on no, skip straight to the concept count. Open the picker with `get_creative_references` (`pathParams.campaignId` — its advertiser is resolved server-side) before calling `start_creative_session`. The picker renders as the `reference-picker` view of the `creativeMcpui` artifact (`kind: 'creative-references'`) and shows "sourced materials" (role=reference library items, any media type) and "past creatives" (every creative the advertiser has, filterable by campaign). An advertiser with no references yet is NOT a text fallback: the picker still opens in an empty state with an "Attach assets" affordance and a generate-without-references action — surface it and let the card carry the choice; do not tell the buyer the library is empty or offer attach/skip as a text option yourself. If it returns `FEATURE_NOT_ENABLED`, the buyer-generative-creative surface is off for this account — say nothing about it and fall back to `start_creative_session` directly, same as when it's on.
3. Start a reviewable draft gallery with `start_creative_session`.
4. Use `refine_creative_session_variant` when the buyer says "make the book more prominent", "more like this one", "add people in the background", or similar.
5. Use `evaluate_creative_session` for draft advisory checks or final blocking checks.
6. Use `finalize_creative_session` after the buyer approves a direction; it saves the selected/refined leaf as a normal campaign creative manifest.
7. Do not use `preview_creative` for draft session variants. Creative-session responses already include `variants[].preview.url`, `format_renders[]`, and the MCPUI gallery; use `get_creative_session` if you need to refresh them.
8. Use lower-level `build_creative_variants`, `create_creative_manifest`, `preview_creative`, and `duplicate_creative` only when the user asks for raw adapter/debug behavior or when a creative session is not appropriate.

For real product/package/logo/book-cover assets supplied by the user, preserve the exact asset:
- Before asking for logos or brand assets, inspect the advertiser's linked brand data (`list_advertisers` with `includeBrand=true` or `get_advertiser`). The resolved brand identity may already contain logos, colors, tone, taglines, and catalog assets.
- If the linked advertiser brand conflicts with the user's explicit product, attached asset, catalog item, or stated website/domain, the user-provided product/site wins. Surface the mismatch plainly and do not use unrelated brand-library logos, industries, colors, or descriptions in the creative brief.
- Put the primary product/package/book image in `source_asset`. Put logos and other additional brand assets in `assets[]` with `role`, `source`, `locked_asset`, `can_transform`, rights, and preservation notes.
- In generation briefs: put the primary product image in `request.creative_brief.product.product_shot.url` and set `locked_asset: true`, `asset_role: "product"`, `can_transform: false`, plus specific `preservation_notes`.
- If an image generator cannot ingest exact logos, keep the logo in `assets[]`; the creative-session renderer/finalizer can composite the exact locked asset instead of asking the buyer to use an external design tool.
- In saved manifests: put the asset in `linked_assets[]` with `locked_asset: true`, `asset_role: "product"`, `can_transform: false`, and the same preservation notes.
- Do not redraw, reinterpret, or replace locked product assets. Only scene, lighting, composition, copy, and background may vary.

Creative asset lifecycle:
- **Source assets** are buyer inputs: uploaded files, DAM URLs, product-catalog images, brand-library assets, logos, audio, video, copy, or references. Track them in `source_asset` / `asset_store.assets[]` with `source`, `role`, `locked_asset`, `can_transform`, `rights.status`, dimensions/checksum when known, and optional crop/mask metadata. A source asset is not automatically the final deliverable.
- **Draft variants** are generated or composited directions returned in `variants[]`. They have preview URLs, evaluator checks, quality/status, and lineage (`parent_build_variant_id` after refinement). They are for human exploration and should remain editable until the buyer approves one.
- **Final products** are campaign creative manifests. `finalize_creative_session` turns the selected/refined leaf into a normal campaign creative with `creative_id`, `format_id`, linked assets, text assets, preview, and sync status.
- Keep provenance separate from rights. `source: "upload"` or `source: "dam"` explains where an asset came from; `rights.status` explains whether it is cleared. `unknown` rights are a non-blocking final review warning. `restricted` or `expired` rights block finalization.
- If a product photo includes studio background/whitespace, prefer a DAM/catalog transparent cutout, mask, or subject bounds. `render_crop` is an acceptable preservation-safe fallback, but it is not as clean as a real cutout.
- To decide whether a finished creative can launch, attach or create the manifest and deliver it through the launch path. The launch path runs local manifest/format checks and then sends creatives through `sync_creatives` or inline creative payloads based on the seller's declared capabilities. `sync_creatives` with `dry_run: true` is the appropriate non-mutating check for the submission path when the seller supports it.

Seller-specific creative coverage:
- Required seller formats come from the campaign's selected products. Use `list_creative_templates` and the campaign's `creativeFormats.required/covered/missing` to know what still needs coverage.
- A single final direction can need multiple seller-specific executions. Example: one final creative direction may require square image, portrait story, audio, video, and/or HTML manifests depending on the products selected.
- `format_renders[]` previews the approved leaf in canonical publisher/format renderers. It is a review surface, not a substitute for saving the right campaign creative manifest for each required format.
- After finalizing, re-fetch the campaign or creative coverage. If `creativeFormats.missing` is not empty, generate/refine additional manifests for those missing formats while preserving the shared source-asset lineage and direction.

Performance loop:
- Creative iteration stops at a saved manifest unless the campaign is launched, measured, and compared.
- For performance campaigns, configure `performanceConfig`, execute once required creative formats are covered, collect reporting and measurement data, compare performance by campaign/media buy/package/creative where available, then start a new creative session or duplicate/refine a winner.
- Do not claim a creative "won" unless that conclusion comes from reporting or measurement data. The draft/final evaluators check readiness and brand/format risk; they do not prove market performance.

Evaluator strategy:
- Treat evaluators as selected checks, not one universal score.
- Core readiness evaluators answer whether previews/manifests are renderable and structurally valid.
- Asset evaluators answer whether locked source assets, provenance, rights, crop/mask/cutout, and fidelity are acceptable.
- Brand/brief evaluators answer whether the concept fits campaign and brand guidance.
- Format/seller evaluators answer whether the final execution satisfies the selected seller or publisher format.
- Policy/safety evaluators answer whether there are legal, claims, regulated-category, or platform-policy risks.
- Performance evaluators answer whether the creative worked after launch; they require reporting or measurement data.
- Draft evaluator warnings should guide refinement. Final hard/blocking failures prevent finalization. Non-blocking final warnings should be summarized as review notes, not hidden and not called clean passes.

**Note:** Template detection and format matching happen automatically when assets are uploaded via the UI. The response includes `auto_detected_template` with the detected `template_id` and detection `method`. There is no need to call a separate templates endpoint.

##### 1. List Creative Manifests

**Operation:** `list_creatives`
```
GET /api/v2/buyer/campaigns/{campaignId}/creatives
```
**Query parameters:**
- `quality` (optional): Filter by quality level
- `search` (optional): Case-insensitive name search
- `take` (optional): Page size, default 50
- `skip` (optional): Pagination offset

**Response:** `{ "manifests": [...], "total": <number> }`. Each manifest is the **summary** shape — `creative_id`, `campaign_id`, `name`, `template_id`, `format_id`, `brand_domain`, `preview_url`, `asset_count`, `sync_status` (`{ synced, agent_count }`), `created_at`, `updated_at`. `assets[]`, `html_processing`, `creative_manifest`, `tracking`, `frequencyCaps[]`, `message`, `target_format_ids[]`, `format_previews[]`, and `auto_detected_template` are NOT on the summary — call `get_creative` for the full manifest.

##### 2. Get Creative Manifest

**Operation:** `get_creative`
```
GET /api/v2/buyer/campaigns/{campaignId}/creatives/{creativeId}
```
Optional query: `?preview=true`

**Response:** Single manifest with all fields including `preview_url`, `format_previews[]`, `auto_detected_template`, `html_processing` (macros injected, unresolved refs), and assets array.

##### 3. Update Creative Manifest (metadata only — NO file uploads)

**Operation:** `update_creative`
```
PUT /api/v2/buyer/campaigns/{campaignId}/creatives/{creativeId}
```
**Body (all fields optional):**
- `name` (string): Manifest name
- `message` (string): Creative brief
- `tag` (string): Tag
- `quality` (string): Quality level
- `format_id` (object): `{ agent_url: string, id: string }` — the **legacy** (v1 named-format) format reference. The canonical 3.1 alternative is `format_kind` (a bare enum like `image`, `video_vast`, `video_hosted` — **no `agent_url`**). They are mutually exclusive; never put one inside the other. For time-based formats, video/audio duration is a **canonical param** (`params.duration_ms_exact` / `duration_ms_range`), not a field on `format_id`. Note `video_standard`/`display_300x250_html` are legacy ids, not `format_kind` values. See `docs/creatives/formats-canonical-vs-legacy.md`.
- `template_id` (string): ADCP format template ID (e.g. `"display_300x250_html"`, `"video_standard"`, `"vendor_dcm_tag"`)
- `click_url` (string): Canonical click-through URL. Use this to update the landing page that powers `click_tracker_url`.
- `url_asset` (object): `{ url: string, url_type: string }` — add a URL-based asset
- `url_assets` (array): Slot-bound URL assets. Use only when you need a specific format slot; prefer `click_url` for landing page/click-through changes.
- `delete_asset_ids` (string[]): Asset IDs to soft-delete
- `reclassify_assets` (array): `[{ asset_id: string, asset_type: string }]` — change asset type
- `frequencyCaps` (array): Buyer-side frequency caps scoped to this creative. Array of `{ max_impressions, window: { interval, unit } }`. **Full-replace semantics**: omit to leave existing caps unchanged, pass `[]` to clear, pass the full array to replace. See the Frequency Caps section below.

**Note:** File uploads are NOT possible via MCP. The MCP update only handles metadata changes, URL assets, and asset deletion/reclassification. For file uploads, direct the buyer to the UI.

##### 4. Bulk Update Creative Manifests

**Operation:** `bulk_update_creatives`
```
POST /api/v2/buyer/campaigns/{campaignId}/creatives/bulk-update
```

Use this for requests like "bulk update the click-through URL for all April creatives" or "change all image creatives matching this folder/name/date range." Do **not** loop `update_creative` unless the buyer explicitly selected one creative.

**Body:**
```json
{
  "selection": {
    "creative_ids": ["creative_1"],
    "name_contains": "2026april",
    "created_after": "2026-04-01T00:00:00.000Z",
    "created_before": "2026-05-01T00:00:00.000Z",
    "updated_after": "2026-04-01T00:00:00.000Z",
    "updated_before": "2026-05-01T00:00:00.000Z",
    "template_id": "display_300x250_html",
    "format_kind": "image",
    "asset_type": "IMAGE"
  },
  "update": {
    "click_url": "https://www.chime.com/join/campaigns/debit/?ad=scope3_debit",
    "format_id": {
      "agent_url": "https://api.openads.ai/adcp/creative",
      "id": "display_300x250_nongenerative"
    },
    "target_format_ids": [
      {
        "agent_url": "https://api.openads.ai/adcp/creative",
        "id": "display_300x250_nongenerative"
      }
    ]
  },
  "dry_run": true,
  "exclude_creative_ids": [],
  "limit": 100
}
```

At least one selector and at least one `update` field are required. The `update` fields are: `click_url` (landing page), `format_id` (set the primary format on every matched creative, e.g. to retype creatives to a sales agent's required format like `display_300x250_nongenerative`), and `target_format_ids` (set the additional formats each matched creative covers). Set these per size: only match creatives whose dimensions fit the target format. `dry_run` defaults to `true`; first call it in dry-run mode and show the matched set (`creative_id`, name, asset count, status) for buyer confirmation. After the buyer confirms, call the same operation with `dry_run: false`, preserving any excludes they requested.

**Response:** `{ dry_run, matched_count, updated_count, updates, creatives: [...] }`. In dry-run, each creative has `status: "matched"`. In execution, each creative has `status: "updated"` or `status: "failed"` with an `error`.

##### 5. Sync Creatives to Storefronts

**Operation:** `sync_creatives`
```
POST /api/v2/buyer/campaigns/{campaignId}/creatives/sync
```

**As operation:**
```json
{
  "operation": "sync_creatives",
  "pathParams": { "campaignId": "campaign_abc123" }
}
```

Pushes all campaign creatives (with full name and asset payloads) to every connected storefront's review queue. Use when the buyer asks to re-send or re-sync creatives after an update, or when a storefront reports it never received the creative assets. No request body required.

**Response:** `{ synced: [{ storefront_id, adcp_media_buy_id, status: "synced" | "error", message? }] }`

##### 6. Delete Creative Manifest

**Operation:** `delete_creative`
```
DELETE /api/v2/buyer/campaigns/{campaignId}/creatives/{creativeId}
```
Soft-deletes the manifest and all its assets (sets `archived_at`).

**Response:** `204 No Content`

##### 6. List Creative Templates / Required Formats

**Operation:** `list_creative_templates`
```
GET /api/v2/buyer/campaigns/{campaignId}/creatives/templates
```
**As operation:**
```json
{
  "operation": "list_creative_templates",
  "pathParams": { "campaignId": "campaign_abc123" }
}
```

Use this before build/save when you need the target `format_id`, available `template_id`, or slot asset IDs.

##### 6. Start Creative Session

**Operation:** `start_creative_session`
```
POST /api/v2/buyer/campaigns/{campaignId}/creative-sessions
```

Starts a human-reviewable creative iteration session and returns an MCPUI gallery payload. The response includes stable `creative_session_id`, draft `variants[]`, evaluator badges, selected/refined variant IDs, and finalization state.

The session is the preferred contract for non-technical creative production. It carries:
- `asset_store`: source assets with provenance, role, lock, rights, and checksum metadata.
- `renderer_capabilities`: session review renderers available for the target creative format or modality.
- `evaluation[]`: staged checks. Draft checks are advisory; final hard/blocking failures prevent finalization.
- `format_renders[]`: publisher/format previews for the approved leaf.

**As operation:**
```json
{
  "operation": "start_creative_session",
  "pathParams": { "campaignId": "campaign_abc123" },
  "body": {
    "storefrontId": 99,
    "providerType": "openai",
    "brand": "The Shift",
    "objective": "<one-line campaign objective from the brief>",
    "prompt": "<what to generate, from the buyer's brief — never a stock scene>",
    "source_asset": {
      "label": "The Shift book cover",
      "url": "murph-attachment://0",
      "source": "upload",
      "role": "product",
      "locked_asset": true,
      "can_transform": false,
      "rights": { "status": "unknown" },
      "preservation_notes": "Preserve the book cover typography, portrait, proportions, and artwork exactly."
    },
    "assets": [
      {
        "label": "Primary brand logo",
        "url": "https://brand.example/logo.png",
        "source": "brand_library",
        "role": "logo",
        "locked_asset": true,
        "can_transform": false,
        "rights": { "status": "unknown" },
        "preservation_notes": "Preserve the logo exactly; do not redraw, recolor, re-letter, or restyle it."
      }
    ],
    "request": {
      "idempotency_key": "creative-run-123",
      "creative_brief": {
        "name": "<creative exploration name from the brief>",
        "prompt": "<creative concept summary derived from the buyer's brief>",
        "audience": "<target audience from the brief>",
        "objective": "Draft product-safe creative variants, then finalize one campaign-ready execution.",
        "product": {
          "name": "The Shift",
          "category": "coffee-table book",
          "product_shot": {
            "url": "murph-attachment://0",
            "locked_asset": true,
            "asset_role": "product",
            "can_transform": false,
            "preservation_notes": "Preserve the book cover typography, portrait, proportions, and artwork exactly."
          }
        }
      },
      "quality": "draft",
      "variant_axis": {
        "dimension": "theme",
        "values": [
          "<concept direction 1 from the brief>",
          "<concept direction 2 from the brief>",
          "<concept direction 3 from the brief>"
        ]
      }
    }
  }
}
```

##### 7. Refine Creative Session Variant

**Operation:** `refine_creative_session_variant`
```
POST /api/v2/buyer/campaigns/{campaignId}/creative-sessions/{sessionId}/variants/{variantId}/refine
```

**As operation:**
```json
{
  "operation": "refine_creative_session_variant",
  "pathParams": {
    "campaignId": "campaign_abc123",
    "sessionId": "cs_123",
    "variantId": "bv_123"
  },
  "body": {
    "feedback": "Make the book more prominent and add the uploaded logo exactly.",
    "assets": [
      {
        "label": "Uploaded logo",
        "url": "murph-attachment://0",
        "role": "logo",
        "locked_asset": true,
        "can_transform": false
      }
    ]
  }
}
```

##### 8. Evaluate Creative Session

**Operation:** `evaluate_creative_session`
```
POST /api/v2/buyer/campaigns/{campaignId}/creative-sessions/{sessionId}/evaluate
```

Use `stage: "draft"` for advisory checks while exploring. Use `stage: "final"` before final approval. Final-stage checks include `severity` and `blocking`; a hard/blocking failure means the creative needs another refinement before `finalize_creative_session`.

##### 9. Finalize Creative Session

**Operation:** `finalize_creative_session`
```
POST /api/v2/buyer/campaigns/{campaignId}/creative-sessions/{sessionId}/finalize
```

Saves the selected/refined leaf as a normal campaign creative manifest and returns the updated MCPUI session payload with `finalization.creative_id`. The service re-runs final checks first; hard/blocking failures return a validation error and do not create the campaign creative.

##### 10. Build Creative Variants

**Operation:** `build_creative_variants`
```
POST /api/v2/buyer/storefronts/{storefrontId}/creative/build
```
**Critical body shape:** the body has exactly two top-level fields: `providerType` and `request`. Do not send top-level `brief`, `assets`, `variants`, `quality`, `channel`, or `format`.

**As operation:**
```json
{
  "operation": "build_creative_variants",
  "pathParams": { "storefrontId": "99" },
  "body": {
    "providerType": "openai",
    "request": {
      "idempotency_key": "creative-run-123",
      "creative_brief": {
        "name": "<creative exploration name from the brief>",
        "prompt": "<creative concept summary derived from the buyer's brief>",
        "audience": "<target audience from the brief>",
        "objective": "Draft product-safe creative variants, then finalize one campaign-ready execution.",
        "product": {
          "name": "The Shift",
          "category": "coffee-table book",
          "product_shot": {
            "url": "murph-attachment://0",
            "locked_asset": true,
            "asset_role": "product",
            "can_transform": false,
            "preservation_notes": "Preserve the book cover typography, portrait, proportions, and artwork exactly."
          }
        }
      },
      "quality": "draft",
      "variant_axis": {
        "dimension": "theme",
        "values": [
          "<concept direction 1 from the brief>",
          "<concept direction 2 from the brief>",
          "<concept direction 3 from the brief>"
        ]
      }
    }
  }
}
```

If the user attached an image in Murph, use the `murph-attachment://N` placeholder shown in the current turn. The tool layer resolves it to the upload bytes.

##### 11. Create Creative Manifest

**Operation:** `create_creative_manifest`
```
POST /api/v2/buyer/campaigns/{campaignId}/creatives/create
```
**As operation:**
```json
{
  "operation": "create_creative_manifest",
  "pathParams": { "campaignId": "campaign_abc123" },
  "body": {
    "name": "<final creative name from the brief>",
    "message": "<final creative selection summary from the brief>",
    "format_id": {
      "agent_url": "http://localhost:4001/storefront/adapter-smoke-openai/mcp",
      "id": "universal_display"
    },
    "template_id": "image",
    "linked_assets": [
      {
        "data_url": "murph-attachment://0",
        "asset_type": "IMAGE",
        "label": "The Shift book cover locked product asset",
        "slot_asset_id": "image",
        "make_primary": true,
        "locked_asset": true,
        "asset_role": "product",
        "can_transform": false,
        "preservation_notes": "Preserve the book cover typography, portrait, proportions, and artwork exactly."
      }
    ],
    "text_assets": [
      { "asset_id": "headline", "content": "<headline copy derived from the brief>" },
      { "asset_id": "cta", "content": "Preview the book" },
      { "asset_id": "brand_name", "content": "The Shift" }
    ]
  }
}
```

If `build_creative_variants` returns a generated image data URL for the selected variant, save that generated image as the primary `linked_assets[].data_url`. Also retain locked product-asset controls when the final render contains a preserved product asset.

##### 12. Preview Creative

**Operation:** `preview_creative`
```json
{
  "operation": "preview_creative",
  "pathParams": {
    "campaignId": "campaign_abc123",
    "creativeId": "creative_123"
  }
}
```
Returns rendered preview payloads, including HTML/image/audio/video renderers when available.

##### 13. Duplicate Creative for Refinement

**Operation:** `duplicate_creative`
```json
{
  "operation": "duplicate_creative",
  "pathParams": {
    "campaignId": "campaign_abc123",
    "creativeId": "creative_123"
  }
}
```

No request body — the copy is automatically named "Copy of {original name}". To rename it afterward, call `update_creative` with the new `name`.

Use this when the buyer says "make a variation", "refine this", "try people in the background", or similar.

---

##### 14. Upgrade Creative Manifest (set format_kind)

Use this when a creative has `requires_upgrade: true` — meaning it was created with a legacy `format_id` and the canonical `format_kind` has not been set. Call this to pin `format_kind` so storefront agents can correctly classify the format.

```http
POST /api/v2/buyer/campaigns/:campaignId/creatives/:creativeId/upgrade
```

**Via `api_call`:**
```json
{
  "method": "POST",
  "path": "/api/v2/buyer/campaigns/campaign_abc123/creatives/creative_123/upgrade",
  "body": {
    "format_kind": "image"
  }
}
```

**Body fields:**
- `format_kind` (string, optional): AdCP 3.1 canonical format kind. Valid values: `image`, `html5`, `display_tag`, `image_carousel`, `video_hosted`, `video_vast`, `audio_hosted`, `audio_daast`, `sponsored_placement`, `native_in_feed`, `responsive_creative`, `agent_placement`, `custom`. When omitted, the server infers from `format_id` (e.g. `display_300x250_html` → `html5`). Must be supplied when the `format_id` is ambiguous (e.g. `display_html`).
- `brandAgentId` (string of digits, optional): Numeric brand agent ID to associate with the manifest. The server backfills from the campaign when omitted; supply only to override the campaign default.

**Response:** the updated creative manifest, with `requires_upgrade: false` once set.

---

#### Display Requirements — When listing manifests (`list_creatives`):
- **Name** and **Creative ID**
- **Template** and **Format ID** (if set) — these are ADCP format IDs (e.g. `display_300x250_html`, `video_standard`)
- **Asset count** — show `asset_count`
- **Sync status** — show `sync_status.synced` and `sync_status.agent_count`
- **Created/Updated timestamps**

For per-asset details (filename, type, `asset_source`, URL), `auto_detected_template`, `message`, `tracking`, `html_processing`, or `frequencyCaps`, call `get_creative`.

---

### Reporting

#### Get Reporting Metrics

**Operation:** `get_reporting_metrics`
```http
GET /api/v2/buyer/reporting/metrics?view=summary&days=7&advertiserId=12345&campaignId=campaign_abc
```
**As operation:**
```json
{ "operation": "get_reporting_metrics", "params": { "view": "summary", "days": "7", "advertiserId": "12345", "campaignId": "campaign_abc" } }
```

Returns reporting data in one of two views: **summary** (hierarchical breakdown) or **timeseries** (flat rows broken out by hierarchy × day).

All `spend` values (and the money metrics derived from them: `ecpm`, `cpc`) are GROSS — fee-inclusive, stated at the fee terms locked on each media buy — so delivered spend compares directly against campaign and media buy budgets with no conversion. The exception: legacy media buys created before fee terms were locked report spend net, exactly as the seller reported it (these buys also carry neither `budget_denomination` nor `budget_breakdown`) — present those numbers as-is. Never restate spend as "net" or subtract a fee from it when presenting numbers to the buyer.

**🛑 REQUIRED QUESTION — ASK BEFORE CALLING**

This endpoint is SERVER-ENFORCED. If you call `get_reporting_metrics` without a date range, the server returns a 400 error telling you to ask the user. Do NOT guess defaults.

**Date range** — always required. Ask the user which period they want (e.g. "last 7 days", "last 30 days", or specific `startDate`/`endDate` in YYYY-MM-DD). Pass as `params.startDate` + `params.endDate` OR `params.days`. Pick `view="timeseries"` if the user wants daily breakdown; otherwise leave it as `summary` (default).

Validation summary:
- date range missing → error
- Both `view="summary"` and `view="timeseries"` honor `days` up to 90, or any explicit `startDate`/`endDate` window. For wider ranges or a file the user can download, pass `download=true` to get a CSV export.

**Query Parameters:**
- `view` (optional): Response format — `summary` (default) or `timeseries`
  - `summary`: Hierarchical breakdown by advertiser → campaign → media buy → package
  - `timeseries`: Flat rows — one entry per (advertiser → campaign → media buy → package) × day. Same hierarchy fields as `summary` plus a `date` column.
- `days` (optional): Number of days to include (default: 7, max: 90). Pass `0` for "all time" — the server resolves the start date to the customer's first reporting date. Ignored if both startDate and endDate are provided.
- `startDate` (optional): Start date in ISO format (YYYY-MM-DD)
- `endDate` (optional): End date in ISO format (YYYY-MM-DD)
- `advertiserId` (optional): Filter by advertiser ID
- `campaignId` (optional): Filter by campaign ID
- `demo` (optional, boolean): When `true`, returns auto-generated demo data instead of querying real data sources. Default: `false`. Useful for testing and previewing the reporting UI without live campaign data.

**⚠️ CRITICAL: Disambiguating "demo" in user requests**

The word "demo" can mean two different things in reporting requests. You MUST distinguish between them:

| User intent | Example phrases | Action |
|-------------|----------------|--------|
| **Demo flag** (synthetic demo data) | "show reporting (demo)", "demo show reporting", "show reporting with demo flag", "show reporting demo mode" | Set `demo=true` query parameter |
| **Name filter** (advertiser/campaign containing "demo") | "show reporting for demo advertiser", "show reporting for % demo %", "show campaigns named demo", "reporting for 'demo brand'" | Use `advertiserId` or `campaignId` filters to match entities whose names contain "demo" — do NOT set `demo=true` |

**How to tell the difference:**
- If "demo" appears as a **modifier or flag on the reporting request itself** (in parentheses, as "demo mode", "demo flag", or as a standalone qualifier adjacent to "reporting"), the user wants `demo=true`.
- If "demo" appears as a **value describing an advertiser, campaign, or entity name** (preceded by "for", "named", "called", or wrapped in quotes/wildcards), the user is filtering by name — do NOT set `demo=true`.

**Summary Response** (`view=summary`, default):
```json
{
  "advertisers": [
    {
      "advertiserId": "12345",
      "advertiserName": "Acme Corp",
      "metrics": { "impressions": 15000, "spend": 750, "clicks": 300, "views": 1500, "completedViews": 1200, "conversions": 75, "leads": 30, "videoCompletions": 1125, "ecpm": 50, "cpc": 2.5, "ctr": 0.02, "completionRate": 0.8 },
      "campaigns": [
        {
          "campaignId": "campaign_abc",
          "campaignName": "Summer Campaign",
          "metrics": { "..." : "..." },
          "mediaBuys": [
            {
              "mediaBuyId": "mb_1",
              "name": "Media Buy One",
              "status": "ACTIVE",
              "metrics": { "..." : "..." },
              "packages": [
                { "packageId": "pkg_1", "metrics": { "..." : "..." } }
              ]
            }
          ]
        }
      ]
    }
  ],
  "totals": { "impressions": 35000, "spend": 1750 },
  "periodStart": "2025-01-01",
  "periodEnd": "2025-01-31"
}
```

**Timeseries Response** (`view=timeseries`):

Flat rows — one entry per (advertiser → campaign → media buy → package) × day. Same hierarchy fields as the summary view, plus a `date` column. When a media buy has no packages, `packageId`, `productId`, and `productName` are empty strings.

```json
{
  "timeseries": [
    {
      "date": "2025-01-01",
      "advertiserId": "12345",
      "advertiserName": "Acme Corp",
      "campaignId": "campaign_abc",
      "campaignName": "Summer Campaign",
      "mediaBuyId": "mb_1",
      "mediaBuyName": "Media Buy One",
      "mediaBuyStatus": "ACTIVE",
      "packageId": "pkg_1",
      "productId": "prod_a",
      "productName": "Product A",
      "metrics": { "impressions": 5000, "spend": 250, "clicks": 100, "views": 500, "completedViews": 400, "conversions": 25, "leads": 10, "videoCompletions": 375, "ecpm": 50, "cpc": 2.5, "ctr": 0.02, "completionRate": 0.8 }
    },
    {
      "date": "2025-01-01",
      "advertiserId": "12345",
      "advertiserName": "Acme Corp",
      "campaignId": "campaign_abc",
      "campaignName": "Summer Campaign",
      "mediaBuyId": "mb_1",
      "mediaBuyName": "Media Buy One",
      "mediaBuyStatus": "ACTIVE",
      "packageId": "pkg_2",
      "productId": "prod_b",
      "productName": "Product B",
      "metrics": { "..." : "..." }
    }
  ],
  "totals": { "impressions": 35000, "spend": 1750 },
  "periodStart": "2025-01-01",
  "periodEnd": "2025-01-07"
}
```

**Metrics included:** impressions, spend, clicks, views, completedViews, conversions, leads, videoCompletions, ecpm, cpc, ctr, completionRate

**FX conversion disclosure:** When a media buy settles in a currency that differs from the buyer's account currency (e.g. a USD-settling storefront for a ZAR buyer), the API converts spend automatically and includes a `deliveryFxConversion` field on the affected media buy:

```json
{
  "fromCurrency": "USD",
  "rate": 16.4336,
  "asOfDate": "2026-07-02",
  "source": "booked"
}
```

`source` is `"booked"` (rate locked at buy creation time, deterministic) or `"snapshot"` (historical daily rate, used for buys created before the conversion feature shipped).

**When `deliveryFxConversion` is non-null on any media buy, always mention it without being asked.** Buyers need this for reconciliation. Example phrasing:

> "Spend was converted from USD to ZAR at a rate of 16.43 (booked at buy time). The R 1,134 shown reflects approximately $69 USD."

If `source` is `"snapshot"`, include the date: "...using the historical rate from 2026-07-02."

#### Export / Download Reporting Metrics as CSV

To export or download reporting data as a CSV file, use the same reporting metrics endpoint with `?download=true`. This generates a CSV and returns a signed download URL (valid 7 days) instead of JSON data.

**IMPORTANT:** When a user asks to "export", "download", "save as CSV", or "get a spreadsheet" of their reporting data, use this endpoint with `download=true`.

**Operation:** `get_reporting_metrics`
```http
GET /api/v2/buyer/reporting/metrics?days=30&download=true
```
**As operation:**
```json
{ "operation": "get_reporting_metrics", "params": { "days": "30", "download": "true" } }
```

All the same query parameters apply (`view`, `days`, `startDate`, `endDate`, `advertiserId`, `campaignId`). The only addition is `download=true`. This means `view=timeseries` is fully supported with `download=true` — use it when the user wants a date-by-date CSV export.

**Response when `download=true`:**
```json
{
  "downloadUrl": "https://storage.googleapis.com/...",
  "expiresAt": "2025-02-20T12:00:00.000Z",
  "fileName": "reporting-metrics-2025-02-06-to-2025-02-13.csv",
  "rowCount": 42
}
```

**CSV columns — summary view (default, 20 columns):** Advertiser ID, Advertiser Name, Campaign ID, Campaign Name, Media Buy ID, Media Buy Name, Media Buy Status, Package ID, Impressions, Spend, Clicks, Views, Completed Views, Conversions, Leads, Video Completions, eCPM, CPC, CTR, Completion Rate. One row per package (media buys with no packages get one row with empty Package ID).

**CSV columns — timeseries view (`view=timeseries`, 21 columns):** Same 20 columns as summary, plus a **Date** column (YYYY-MM-DD). One row per (package × day); media buys with no packages get one row per day with empty Package ID.

**CRITICAL: NEVER generate your own CSV, Excel, or spreadsheet files.** Always use this `?download=true` endpoint to produce reporting exports. The endpoint handles proper formatting, escaping, and data integrity. Do not use artifacts, code execution, or any other mechanism to create files — use the API.

When the download response is received, present the `downloadUrl` to the user as a clickable download link. Include the `fileName` and note that the link expires after 7 days (`expiresAt`).

---

### Data Delivery

Data Delivery ships log-level data (LLD) — impressions, clicks, conversions, MMP postbacks, VAST events, media-buy delivery — from Scope3 into a buyer-owned destination (GCS bucket, S3 bucket, or Azure Blob container) on a recurring cadence. It is the **push counterpart to `get_reporting_metrics`**: reporting metrics are pulled aggregates; Data Delivery writes row-level files into the buyer's own storage.

When the user says "ship impressions to our S3 bucket", "send daily clicks to GCS", "set up MMP postback exports", "deliver log-level data to our warehouse landing zone", or anything similar, this is the capability they want.

#### Concept model

Two objects, both **advertiser-scoped**:

| Object | What it does | Where it lives |
|---|---|---|
| **Data Delivery Credential** | Named handle that tells Scope3 *where* to write and how to authenticate. Probed asynchronously. | `dataDelivery.credentials[]` on the advertiser body |
| **Data Delivery Output** | Standing subscription: "ship `<dataDeliveryType>` to `<credentialName>` every `<cadence>` as `<format>` under `<pathPrefix>`". | `dataDelivery.outputs[]` on the advertiser body (default) or on the campaign body (override) |

Limits: up to **20 credentials per advertiser**, and **20 Outputs per scope** (the advertiser default array, plus up to 20 per campaign override). One Output per `(dataDeliveryType, credentialName)` pair per scope — to fan a data type out to two destinations, list one Output per destination.

#### How to set it up — use `update_advertiser`

There is **no separate `create_data_delivery_credential` tool**. Credentials and Outputs are inline arrays on the advertiser body, set via `update_advertiser` (or `create_advertiser` on first create). Both arrays are **full-replace** — the request body is the new desired state.

**Always fetch the current state with `get_advertiser` first** so you don't accidentally archive existing credentials or clear existing Outputs. A credential cannot be archived while any live Output (advertiser- or campaign-scoped) still references it — remove or repoint the Output first or the request will fail validation.

```json
{
  "operation": "update_advertiser",
  "pathParams": { "advertiserId": "12345" },
  "body": {
    "dataDelivery": {
      "credentials": [
        {
          "name": "primary-gcs",
          "config": { "type": "GCS", "bucket": "acme-scope3-data" }
        }
      ],
      "outputs": [
        {
          "dataDeliveryType": "IMPRESSIONS",
          "cadence": "HOURLY",
          "credentialName": "primary-gcs",
          "deliveryConfig": { "type": "GCS", "pathPrefix": "scope3/impressions/", "format": "PARQUET" }
        }
      ]
    }
  }
}
```

Credentials are applied before Outputs in a single transaction, so a brand-new credential can be referenced by a brand-new Output in the same request.

#### Field reference

| Field | Type | Values |
|---|---|---|
| `dataDelivery.credentials[].name` | string | Unique per advertiser among live credentials. Outputs reference it by this name. |
| `dataDelivery.credentials[].config.type` | enum | `GCS` \| `S3` \| `AZURE_BLOB` |
| `dataDelivery.credentials[].config.bucket` | string | GCS or S3 bucket name |
| `dataDelivery.credentials[].config.region` | string | **S3 only** — e.g., `us-east-1`, `us-gov-east-1` |
| `dataDelivery.credentials[].config.storageAccountName` | string | **Azure only** — 3–24 lowercase alphanumerics |
| `dataDelivery.credentials[].config.containerName` | string | **Azure only** — 3–63 chars, lowercase + hyphens |
| `dataDelivery.credentials[].config.auth` | object | **Azure only** — `{ "mode": "SAS_TOKEN", "sasToken": "<sas>" }`. SAS is write-only — never echoed back on `get_advertiser`. |
| `dataDelivery.outputs[].dataDeliveryType` | enum | `MB_DELIVERY` \| `IMPRESSIONS` \| `CLICKS` \| `VAST_EVENTS` \| `CAPI_ATTRIBUTION` \| `MMP_POSTBACKS` |
| `dataDelivery.outputs[].cadence` | enum | `HOURLY` (minute 0) \| `DAILY` (00:00 UTC) \| `WEEKLY` (00:00 UTC on `syncWeeklyDay`) |
| `dataDelivery.outputs[].syncWeeklyDay` | int | **Required when cadence=WEEKLY.** `0`=Sunday … `6`=Saturday. |
| `dataDelivery.outputs[].enabled` | boolean | Default `true`. Set `false` to pause the schedule without removing the Output. |
| `dataDelivery.outputs[].credentialName` | string | Must reference a live credential on the same advertiser whose `destinationType` matches `deliveryConfig.type`. |
| `dataDelivery.outputs[].deliveryConfig.type` | enum | Must match the credential's `type` (`GCS` \| `S3` \| `AZURE_BLOB`). |
| `dataDelivery.outputs[].deliveryConfig.pathPrefix` | string | Used verbatim — leading slashes are not stripped, not templated. End with `/` for a directory-like layout. |
| `dataDelivery.outputs[].deliveryConfig.format` | enum | `JSONL` (default) \| `PARQUET` \| `CSV` |

#### Probe lifecycle

When a credential is created or its config changes, Scope3 asynchronously writes a sentinel object to the destination, deletes it, and updates the credential row:

| `status` | Meaning |
|---|---|
| `PENDING` | Just created or rotated. Probe hasn't completed yet — typically clears within seconds. |
| `VALIDATED` | Probe wrote and deleted a sentinel object successfully. `validatedAt` reflects the run. |
| `FAILED` | Probe could not write. `statusError` carries a human-readable reason — present it to the user verbatim. |

Re-fetch with `get_advertiser` to see the current `status`. Outputs can reference a credential in any status; the shipping workflow re-checks at run time.

To force a fresh Probe (e.g., the buyer fixed an IAM grant out-of-band and wants to revalidate without resubmitting the SAS token):

```json
{
  "operation": "revalidate_data_delivery_credential",
  "pathParams": { "advertiserId": "12345", "name": "primary-gcs" }
}
```

#### Common Probe failures to recognize

When `statusError` matches one of these patterns, surface the cause and the fix verbatim — the buyer's next step is on their cloud, not in the API.

| `statusError` contains | Destination | What to tell the buyer |
|---|---|---|
| `does not have storage.objects.create`, `403` | GCS | Grant `roles/storage.objectCreator` to `report-delivery@swift-catfish-337215.iam.gserviceaccount.com` on the bucket, then revalidate. |
| `AccessDenied`, `403` | S3 | Confirm the bucket policy grants `s3:PutObject` to `arn:aws:iam::948454267882:user/scope3-data-sync-service` on `arn:aws:s3:::<bucket>/*`. |
| `PermanentRedirect` | S3 | `config.region` doesn't match the bucket's actual region. |
| `Azure SAS token expired at ...` | Azure | The `se=` claim is in the past. Mint a new SAS and submit a fresh credential under the same `name` to rotate. |
| `AuthorizationPermissionMismatch` | Azure | SAS is valid but missing required permissions. Confirm `sp` includes `c`, `w`, `d` (`sp=cwd`). For Account SAS, also confirm `ss=b`. |
| `AuthorizationResourceTypeMismatch` | Azure | **Account SAS only** — `srt` is missing `o` (Object). The Probe performs blob-level operations (`PUT Blob`, `DELETE Blob`), so `srt` must include `o` (e.g., `srt=co`). `srt=s` or `srt=c` alone won't work. Ask the buyer to regenerate with `--resource-types co`. |
| `AuthorizationFailure` on `_scope3-probe/<uuid>.txt` | Azure | Service SAS was scoped to a single blob (`sr=b`) instead of the container (`sr=c`). The Probe writes to a randomized path, so a blob-scoped SAS will never match. Ask the buyer to regenerate with `az storage container generate-sas` so the SAS is bound to the container. |
| `ContainerNotFound` | Azure | `containerName` doesn't exist on the storage account, or the SAS was issued at a different scope. |

After the buyer applies the fix, call `revalidate_data_delivery_credential` to re-run the Probe — they don't have to resubmit the credential body for GCS/S3 cases (no secret to re-paste).

#### Campaign-scoped overrides — use `update_campaign`

For a single campaign that needs a different cadence, format, or destination than the advertiser default, set `dataDelivery.outputs` on the campaign. The override replaces the advertiser-scoped Output for the matching `dataDeliveryType` on this campaign only.

```json
{
  "operation": "update_campaign",
  "pathParams": { "campaignId": "99999" },
  "body": {
    "dataDelivery": {
      "outputs": [
        {
          "dataDeliveryType": "IMPRESSIONS",
          "cadence": "HOURLY",
          "credentialName": "primary-gcs",
          "deliveryConfig": { "type": "GCS", "pathPrefix": "scope3/campaign-99999/impressions/", "format": "PARQUET" }
        }
      ]
    }
  }
}
```

In `get_campaign` and `get_advertiser` responses, resolved Outputs are tagged with `source: "advertiser"` or `source: "campaign"` so the user can see which level produced each entry — surface that distinction when listing Outputs.

#### What to present

After setting up credentials or Outputs, **show the user the resolved state from the response**, not a summary:

- For each credential: `name`, `destinationType`, `status`, `validatedAt` (if VALIDATED), `statusError` (if FAILED), `expiresAt` (Azure only).
- For each Output: `dataDeliveryType`, `cadence`, `credentialName`, `deliveryConfig.pathPrefix`, `deliveryConfig.format`, `enabled`, `source` (advertiser vs campaign).

If a credential is `PENDING` after the call, tell the user the Probe is running and they can re-check via `get_advertiser`. If `FAILED`, surface the `statusError` exactly and offer to revalidate after they fix the underlying access grant.

#### Buyer-side setup not done via the API

Before a credential will validate, the buyer must grant Scope3 access on their cloud. **Do not attempt this for the user** — point them at the [Data Delivery guide](https://docs.interchange.io/v2/guides/data-delivery) for the exact GCP service account, AWS IAM principal, and Azure SAS permission list.

---

### Catalogs

Catalogs are managed entirely through sync — there is no separate create/update/delete.

**Before calling sync, you MUST collect from the user:**
1. Which advertiser — the advertiser ID goes in the URL path AND in `account.account_id` in the request body
2. The catalog(s) to sync — each needs a `catalog_id`, `type`, and either a feed `url` or inline `items`

#### Sync Catalogs

**Option A — Remote feed URL (multiple catalogs in one call):**

**Operation:** `sync_catalogs`
```http
POST /api/v2/buyer/advertisers/26/catalogs/sync
{
  "account": { "account_id": "26" },
  "catalogs": [
    {
      "catalog_id": "products-2026",
      "type": "product",
      "name": "2026 Product Catalog",
      "url": "https://example.com/products.xml",
      "feed_format": "google_merchant_center",
      "update_frequency": "daily"
    },
    {
      "catalog_id": "promotions-q1",
      "type": "promotion",
      "name": "Q1 Promotions",
      "url": "https://example.com/promotions.xml",
      "feed_format": "custom",
      "update_frequency": "hourly"
    }
  ]
}
```

**Option B — Inline items:**

**Operation:** `sync_catalogs`
```http
POST /api/v2/buyer/advertisers/26/catalogs/sync
{
  "account": { "account_id": "26" },
  "catalogs": [
    {
      "catalog_id": "my-catalog-1",
      "type": "product",
      "name": "Q1 Products",
      "items": [
        {
          "item_id": "sku-001",
          "title": "Blue Widget",
          "description": "A sturdy blue widget for everyday use",
          "price": "19.99 USD",
          "link": "https://example.com/products/blue-widget",
          "image_link": "https://example.com/images/blue-widget.jpg",
          "availability": "in stock",
          "brand": "Acme",
          "google_product_category": "Hardware > Tools"
        },
        {
          "item_id": "sku-002",
          "title": "Red Widget",
          "description": "A sturdy red widget for everyday use",
          "price": "24.99 USD",
          "link": "https://example.com/products/red-widget",
          "image_link": "https://example.com/images/red-widget.jpg",
          "availability": "in stock",
          "brand": "Acme",
          "google_product_category": "Hardware > Tools"
        }
      ]
    }
  ]
}
```

> Items are free-form key/value objects — the fields depend on the catalog `type`. The examples above use common product fields. For `job` catalogs use fields like `job_id`, `title`, `company`, `location`; for `hotel` use `hotel_id`, `name`, `address`, `star_rating`; etc.

**URL path:** `/advertisers/{advertiserId}/catalogs/sync` — the `{advertiserId}` is the numeric advertiser ID (e.g. `26`). Also include it in the request body as `account.account_id`.

**`account` (required in body):**
- `account_id` (string): The advertiser ID — same value as the path `{advertiserId}` (e.g. `"26"`).

**`catalogs` array (required, 1–50 items). Each object:**
- `catalog_id` (string, required): Buyer-assigned identifier
- `type` (string, required): `offering`, `product`, `inventory`, `store`, `promotion`, `hotel`, `flight`, `job`, `vehicle`, `real_estate`, `education`, `destination`
- `name` (string, optional): Display name
- `url` (string): Remote feed URL — provide this OR `items`, not both
- `items` (array): Inline catalog items — provide this OR `url`, not both
- `feed_format` (string, optional): `google_merchant_center`, `facebook_catalog`, `shopify`, `linkedin_jobs`, `custom`
- `update_frequency` (string, optional): `realtime`, `hourly`, `daily`, `weekly`
- `conversion_events` (array, optional): Conversion event IDs

**Other optional fields:**
- `catalog_ids` (array): Filter which catalog_ids from the `catalogs` array to process
- `delete_missing` (boolean): Archive catalogs not included in this request (default: false)
- `dry_run` (boolean): Preview changes without persisting (default: false)
- `validation_mode` (string): `strict` (default) or `lenient`

**Response:**
```json
{
  "data": {
    "results": [
      {
        "catalog_id": "my-catalog-1",
        "action": "created",
        "name": "Q1 Products",
        "type": "offering"
      }
    ]
  }
}
```

Actions: `created`, `updated`, `unchanged`, `failed`, `deleted`

#### List Catalogs

**Operation:** `list_catalogs`
```http
GET /api/v2/buyer/advertisers/26/catalogs
```
**As operation:**
```json
{ "operation": "list_catalogs", "pathParams": { "advertiserId": "26" } }
```

**URL path:** `/advertisers/{advertiserId}/catalogs` — the `{advertiserId}` is the numeric advertiser ID.

**Query Parameters:**
- `type` (optional): Filter by catalog type (`offering`, etc.)
- `take` / `skip` (optional): Pagination

**Response (200):**
```json
{
  "data": {
    "account": { "account_id": "26" },
    "catalogs": [
      {
        "catalogId": "my-catalog-1",
        "type": "offering",
        "name": "Q1 Products",
        "url": "https://example.com/feed.xml"
      }
    ]
  }
}
```

---

### Storefronts

Browse available storefronts, register credentials for inventory sources, and
link accounts to advertisers.

**Official adapter storefronts — CRITICAL:** Built-in adapters such as Amazon,
Google, Meta, Pinterest, Reddit, Snap, Spotify, TikTok, and AudioStack are
storefronts whose buyer-facing dispatcher calls the adapter directly. They are
not inventory sources inside another storefront. If a user asks to "connect to
Snap", "connect Google", or use any other official adapter, do not page through
public storefronts looking for a matching source row. Use the specific adapter
storefront MCP endpoint surfaced by discovery/configuration and let that
storefront return its OAuth/BYOK auth challenge. The per-source credential
registration flow below applies only to external AdCP source rows returned by
`get_storefront_capabilities`.

**CRITICAL — Account Status Awareness:**
For Chef/composed storefronts, the list returns each storefront's `sourceCount`
and `connectedSourceCount`. When `connectedSourceCount < sourceCount`, the
storefront has unconnected inventory sources. Call `get_storefront` for the
rolled-up state, `get_storefront_capabilities` for active source rows, and
`list_agent_credentials` for the credentials already registered. Specifically:
- If an external source has `requiresCredentials: true` and no active credential in `list_agent_credentials` for that `(storefrontId, sourceId)`: Tell the user they need to register credentials for this source before they can use it.
- If credentials exist for that source: Show the registered account identifiers and statuses from `list_agent_credentials`.
- If an external source has `requiresCredentials: false`: The platform handles credentials — no action needed from the user.
- If an external source has `requiresCredentials: null`: The credential requirement is unavailable from non-synthetic capabilities and stored source configuration; do not infer that registration is required from the capability diagnostic alone.
- If a source has `probeable: false`, do not call it unreachable or ask for AdCP credentials based on this diagnostic alone; managed ad-server-backed sources are not checked through this endpoint.

Never silently omit this information. The user needs to know which storefronts have unconnected sources and which are ready to use.

**CRITICAL — Proactive Registration Prompt for disconnected sources:**
When `connectedSourceCount < sourceCount` for any Chef/composed storefront in the list, you MUST:
1. **Drill in via `get_storefront_capabilities` and `list_agent_credentials`** — identify external sources with `requiresCredentials: true` and no active credential for that `(storefrontId, sourceId)`.
2. **Explicitly call it out in a separate section** — After showing the storefront list, add a clear callout like: "The following sources require you to register credentials: [source names]. Would you like to register credentials for any of them?"
3. **Offer to start the registration flow** — Ask the user if they want to register credentials now. If yes, use `register_source_credentials` operation.
4. **After registering credentials, offer to link an account to an advertiser** — Once credentials are registered, ask: "Now that credentials are set up for [source name], would you like to discover and link an account to a specific advertiser?" If yes, follow the Link Agent Account to Advertiser workflow (see Advertisers section):
   a. List the customer's advertisers via `list_advertisers` operation
   b. For the chosen advertiser, discover available accounts for the agent via `list_available_accounts` operation
   c. Present discovered accounts and let the user pick
   d. Link via `update_advertiser` operation with `linkedAccounts`

This end-to-end flow (list storefronts → get_storefront_capabilities →
list_agent_credentials → register credentials → link account to advertiser)
should feel seamless for inventory-source storefronts. Do NOT make the user
figure out the next step — always offer it.
Do not apply this flow to official adapter storefronts; their provider
authorization happens through the adapter MCP OAuth/BYOK flow.

#### List Storefronts

List all enabled storefronts visible to the buyer. Each storefront contains inventory sources backed by agents. Results are paginated.

**Operation:** `list_storefronts`
```http
GET /api/v2/buyer/storefronts?name=Roundel&visibility=public&limit=20&offset=0
```
**As operation:**
```json
{ "operation": "list_storefronts", "params": { "name": "Roundel", "visibility": "public", "limit": "20", "offset": "0" } }
```

**Query Parameters (all optional):**
- `name` (string): Filter by storefront name (partial match, case-insensitive)
- `status` (`"configuring"` | `"transacting"` | `"archived"`): Filter by display status. `transacting` is live and accepting transactions; `configuring` is listed but still setting up.
- `channel` (string): Filter to storefronts that carry this ADCP channel code (e.g. `display`, `olv`, `ctv`).
- `region` (string): Filter to storefronts that cover this region code (e.g. `EMEA`, `NORAM`, `APAC`).
- `visibility` (`"public"` | `"private"`): Which storefronts to list. Defaults to `"public"` — ACTIVE storefronts available to any buyer. Use `"private"` to return ALL storefronts (ACTIVE, PENDING, **and DISABLED**) owned by customers in the caller's parent org. The DISABLED case matters when a customer also operates sell-side under the same umbrella and needs to see paused storefronts to bring them back, run pre-launch tests, or prep credentials in advance.
- `publisherDomain` (string): Filter to storefronts that carry this publisher domain in their declared publisher list (e.g. `bbc.com`). Useful for vendor selection — "which storefronts represent this publisher?" Returns only non-adapter storefronts.
- `limit` (number): Maximum storefronts per page (default: 20, max: 50)
- `offset` (number): Number of storefronts to skip for pagination (default: 0)

**Response:**
```json
{
  "data": {
    "items": [
      {
        "id": 1,
        "platformId": "premium-video",
        "name": "Premium Video Exchange",
        "publisherDomain": "premiumvideo.com",
        "status": "ACTIVE",
        "channels": ["CTV", "display"],
        "sourceCount": 3,
        "connectedSourceCount": 2,
        "publishers": {
          "total": 120,
          "verified": 95,
          "sample": ["bbc.com", "guardian.com", "telegraph.co.uk"]
        }
      }
    ],
    "total": 25,
    "hasMore": true,
    "nextOffset": 20
  }
}
```

**Pagination:** When `hasMore` is true, use the `nextOffset` value as the `offset` parameter in your next request to fetch the next page. Continue until `hasMore` is false or `nextOffset` is null.

**Notes:**
- `visibility=public` (default) returns only ACTIVE storefronts; `visibility=private` can include `PENDING` and `DISABLED` storefronts owned by the caller's parent org, so always check `status` before assuming a row is sellable.
- Each storefront has one or more inventory sources, each backed by an agent
- `channels` is the list of ad channels the storefront supports (e.g. `["CTV", "display", "audio"]`). Empty array if not configured.
- `sourceCount` is the total number of inventory sources on the storefront; `connectedSourceCount` is how many of those the buyer is wired to use, either because no buyer credentials are required or because credentials are active
- `publishers` is a summary of the storefront's declared publisher coverage — `{ total, verified, sample[] }`. `null` for adapter storefronts (walled-garden platforms have no publisher list). Use `list_storefront_publishers` for the full paginated list.
- `supportedRoutingTypes: ["ROUTED"]` on a storefront means the storefront is routed-only. Adapter storefronts return this; a media buy through one is routed automatically (routing is derived per media buy from its storefront — there is nothing to set yourself, and other buys on the same campaign can still be decisioned).
- The list does NOT include per-source rows — call `get_storefront_capabilities` to enumerate active source IDs and external AdCP capability/auth state. Call `get_storefront` only for rolled-up storefront state such as `connected`, `requiresCredentials`, and `customerAccounts`.

**Credential rules — CRITICAL (apply to source rows on `get_storefront_capabilities`):**
- `requiresCredentials: true` on an external AdCP source → the buyer MUST provide their own credentials. Registration is done via `register_source_credentials` operation.
- `requiresCredentials: false` on an external AdCP source → Scope3 acts as the agent on behalf of the advertiser. **Individual credential registration is NOT possible.**
- `requiresCredentials: null` → the credential requirement is unavailable from non-synthetic capabilities and stored source configuration, or not applicable. Do not infer that registration is required.
- `probeable: false` → this diagnostic does not check an external AdCP agent for the source. Do not infer that the source is unreachable.
- `supportedRoutingTypes: ["ROUTED"]` → this source is routed-only. Adapter storefront sources return this; a media buy through one is routed automatically (routing is derived per media buy from its storefront — there is nothing to set yourself).
- **Account linking requires `requiresCredentials: true`.** Only sources where the buyer registers their own credentials can have accounts discovered and linked to advertisers.

**Display Requirements — ALWAYS include when listing storefronts:**

Present each storefront as a structured entry (not prose). For every storefront, show:
- **Name** and **ID**
- **Source coverage** — show `connectedSourceCount` / `sourceCount` (e.g. "2 of 3 sources connected")

Key rules:
- Never summarize into "You have N storefronts." Always show the per-item details above.
- **When `connectedSourceCount < sourceCount`, drill in via `get_storefront_capabilities` and `list_agent_credentials`** to identify external sources that still need credentials, then ask the user about registration. Do NOT just list them and move on.

#### Get Storefront AdCP Capabilities

Returns source-level capability diagnostics for each active source on a storefront. External AdCP sources include cached-or-refreshed capability details. Managed ad-server-backed sources are returned with `probeable: false` and `probeStatus: "not_applicable"`; do not treat those rows as unreachable agents. Use this to debug operation gating issues (e.g. why `update_media_buy` is blocked, whether sandbox mode is supported, what billing types are allowed) without needing to contact the agent team.

**Operation:** `get_storefront_capabilities`
```http
GET /api/v2/buyer/storefronts/:storefrontId/capabilities
```
**As operation:**
```json
{ "operation": "get_storefront_capabilities", "params": { "storefrontId": "53" } }
```

**Response (200):**
```json
{
  "data": {
    "storefrontId": 53,
    "agents": [
      {
        "sourceId": "selleragent-pubx-ai",
        "sourceName": "Pubx Direct",
        "executionType": "AGENT",
        "agentName": "Pubx Sales Agent",
        "requiresCredentials": false,
        "probeable": true,
        "probeStatus": "reachable",
        "message": null,
        "capabilities": {
          "version": "v3",
          "tools": ["get_products", "create_media_buy", "update_media_buy", "get_adcp_capabilities"],
          "protocols": ["media_buy", "creative"],
          "features": { "content_standards": false, "inline_creative_management": true },
          "accountResolution": "implicit_from_sync",
          "requireOperatorAuth": false,
          "defaultBilling": "agent",
          "supportedBillings": ["operator", "agent"],
          "reportingDeliveryMethods": null,
          "sandboxSupported": false,
          "synthetic": false,
          "raw": { ... }
        }
      },
      {
        "sourceId": "managed-ad-server",
        "sourceName": "Managed ad server",
        "executionType": "MANAGED_SALES_AGENT",
        "agentName": null,
        "requiresCredentials": null,
        "probeable": false,
        "probeStatus": "not_applicable",
        "message": "Managed ad-server-backed source; not probed via AdCP capabilities.",
        "capabilities": null
      }
    ]
  }
}
```

`probeStatus: "reachable"` means non-synthetic capabilities are currently available, possibly from cache; it is not a guarantee that this request opened a fresh network connection. `capabilities` is `null` for managed sources or if the external agent has no available capability payload. `synthetic: true` means the capabilities were synthesized from the tool list (no `get_adcp_capabilities` call succeeded). `raw` contains the full uninterpreted AdCP capability payload.

Use the `tools` array to check whether a specific operation (e.g. `update_media_buy`) is advertised. Use `supportedBillings` and `accountResolution` to debug account setup issues.

#### Get Storefront Publishers

Returns the full paginated list of publisher domains a storefront claims to represent. Use this when the buyer wants to verify a seller's publisher coverage before committing to a buy — "does this storefront actually carry BBC?", "what publishers does Ozone represent?".

Not available for adapter storefronts (Meta, Snap, TikTok, etc.) — returns 404. Use the `publishers` summary on the storefront detail (see `list_storefronts` / `get_storefront`) to know whether a publisher list exists before calling this endpoint.

**Operation:** `list_storefront_publishers`
```http
GET /api/v2/buyer/storefronts/:storefrontId/publishers?verification=verified&q=bbc
```
**As operation:**
```json
{ "operation": "list_storefront_publishers", "params": { "storefrontId": "42", "verification": "verified" } }
```

**Query Parameters (all optional):**
- `verification` (`"verified"` | `"declared"`): Filter by verification status. `verified` = cross-checked against the seller's adagents.json; `declared` = seller-asserted only.
- `q` (string): Prefix filter on domain name (e.g. `q=bbc` returns `bbc.com`, `bbc.co.uk`).
- `limit` (number): Max results per page (default: 20, max: 100).
- `offset` (number): Pagination offset.

**Response (200):**
```json
{
  "data": {
    "publishers": [
      { "domain": "bbc.com", "verification": "verified", "propertyCount": 142 },
      { "domain": "bbc.co.uk", "verification": "declared", "propertyCount": null }
    ],
    "total": 120,
    "verifiedCount": 95
  }
}
```

**Notes:**
- `domain` is the publisher_domain namespace from the seller's adagents.json authorization chain. It is the entity that controls seller access — not necessarily a property URL or property_id.
- `propertyCount` is the number of resolved properties under this publisher domain (from the AAO registry), or `null` if not yet resolved.
- `verifiedCount` is the count of `verified` entries out of `total`. Note: the inline summary (from `list_storefronts` / `get_storefront`) uses `verified` for this field; this full-list response uses `verifiedCount`.
- Adapter storefronts (walled-garden platforms) return 404 — they have no publisher list.
- The storefront's `publishers` summary in `list_storefronts` / `get_storefront` gives `{ total, verified, sample[] }` inline, so you only need this endpoint for full pagination or domain-level filtering.

#### List Registered Agent Credentials

List all agent credentials registered by this customer across all agents.

**Operation:** `list_agent_credentials`
```http
GET /api/v2/buyer/storefronts/credentials
```
**As operation:**
```json
{ "operation": "list_agent_credentials" }
```

**Response (200):**
```json
{
  "data": [
    {
      "id": "722",
      "accountIdentifier": "Retail Network Creds",
      "accountType": "CLIENT",
      "status": "ACTIVE",
      "registeredBy": "user@example.com",
      "createdAt": "2026-02-23T19:55:11.602Z",
      "updatedAt": "2026-02-23T19:56:56.272Z",
      "sources": [
        {
          "storefrontId": 1,
          "storefrontName": "Retail Media Network",
          "sourceId": "retail-network-agent",
          "sourceName": "Retail Network Agent"
        }
      ]
    }
  ]
}
```

**Notes:**
- Returns all credentials registered by this customer, scoped to the storefront sources each one gives access to
- `sources[]` lists every (storefrontId, sourceId) pair the credential covers — a single credential row can cover the same source across multiple storefronts
- Auth secrets are never returned (sensitive — stored in Google Secret Manager)
- Use this to check which sources are connected before linking accounts to advertisers

#### Register Agent Credentials

Register credentials for a specific inventory-source agent at the **customer
level**. This is the first step in connecting to a legacy or third-party source
listed by `get_storefront_capabilities` with
`requiresCredentials: true` — credentials belong to the whole customer, not a
specific advertiser. Once credentials are registered, accounts can be discovered
and linked to individual advertisers (see Link Agent Account to Advertiser in
the Advertisers section).

**Multiple credentials per agent:** A customer CAN register multiple sets of
credentials for the same inventory-source agent (e.g., two different accounts
with different API keys). Each set uses a different `accountIdentifier`. Each
credential discovers its own set of ad accounts. When discovering accounts for
linking, the `credentialId` parameter is required so the system knows which
credential to query — see "Link Agent Account to Advertiser" in the Advertisers
section for the full workflow.

**Do not use this endpoint for official adapters.** For built-in adapter
storefronts such as Snap, Google, Meta, Spotify, TikTok, Reddit, Pinterest,
Amazon, and AudioStack, provider authentication is handled by the adapter
storefront's OAuth/BYOK MCP flow. There is no `sourceId` to discover under a
different storefront and no `register_source_credentials` call to make.

After a registered AdCP source account is discovered and linked to an
advertiser, `connect_adcp_storefront` can project that existing relationship
into the shared connection plane for seller-managed campaigns. This alpha
operation does not register credentials and never accepts an endpoint URL or
raw secret:

```json
{
  "operation": "connect_adcp_storefront",
  "pathParams": { "storefrontId": "42", "sourceId": "src_abc123" },
  "body": {
    "advertiserId": "12345",
    "accountId": "account-returned-by-list-available-accounts",
    "credentialId": "credential-returned-by-list-available-accounts"
  }
}
```

It returns `connectionId` and `connectionAccountId`. Use those IDs with the
seller-managed campaign subscription operations. `credentialId` is optional
only when exactly one active credential exposes the selected account. The
connection records the source's reported tools as readiness evidence but does
not reject an otherwise active, mapped source for omitting an operation.
If a later subscribe, delivery, or write operation is unsupported, report that
operation's exact error and continue to offer the source's other capabilities.

**Operation:** `register_source_credentials`
```http
POST /api/v2/buyer/storefronts/{storefrontId}/sources/{sourceId}/credentials
{
  "accountIdentifier": "my-publisher-account",
  "auth": {
    "type": "bearer",
    "token": "my-api-key"
  }
}
```
**As operation:**
```json
{ "operation": "register_source_credentials", "pathParams": { "storefrontId": "1", "sourceId": "src_abc123" }, "body": { "accountIdentifier": "my-publisher-account", "auth": { "type": "bearer", "token": "my-api-key" } } }
```

**Path Parameters:**
- `storefrontId` (number): The storefront ID
- `sourceId` (string): The inventory source ID within the storefront

**Required Fields:**
- `accountIdentifier` (string): Unique account identifier for this agent

**Optional Fields:**
- `auth` (object): Authentication credentials. Required for API_KEY/JWT agents, not needed for OAUTH agents.
- `marketplaceAccount` (boolean): Admin-only flag for marketplace accounts

**OAUTH agents:** Do NOT ask the user for any OAuth credentials (client_id, client_secret, tokens, etc.). Just omit the `auth` field. The response will include an `oauth.authorizationUrl` — present this link to the user to complete authorization. The platform handles discovery, client registration, and token exchange automatically.

**Response (non-OAUTH, 201):**
```json
{
  "id": "123",
  "accountIdentifier": "my-publisher-account",
  "status": "ACTIVE",
  "registeredBy": "user@example.com",
  "createdAt": "2026-01-15T10:00:00Z"
}
```

**Response (OAUTH, 201):**
```json
{
  "id": "123",
  "accountIdentifier": "my-publisher-account",
  "status": "PENDING",
  "registeredBy": "user@example.com",
  "createdAt": "2026-01-15T10:00:00Z",
  "oauth": {
    "authorizationUrl": "https://agent.example.com/authorize?client_id=abc&...",
    "storefrontId": 1,
    "sourceId": "src_abc123",
    "sourceName": "Inventory Source Name"
  }
}
```

**Notes:**
- Agent must be ACTIVE before accounts can be registered
- For OAUTH agents, the account is created with PENDING status and includes an `authorizationUrl` for the user to click
- After the user authorizes, the account status changes to ACTIVE automatically

### Audiences

Sync first-party CRM audiences into Scope3 for later syndication to sales agents. Audiences contain hashed customer identifiers used for targeting. Processing is **asynchronous** — sync returns immediately with an `operationId`, and processing completes in the background.

**Important:** All member identifiers must be pre-hashed before sending:
- **Email:** SHA-256 of lowercase, trimmed email (64-char hex string)
- **Phone:** SHA-256 of E.164-formatted phone number (64-char hex string)
- **Universal IDs:** RampID, UID2, MAID, etc. passed as-is

**Limits:** Maximum 100,000 total members per sync call. For larger lists, chunk into sequential requests.

#### Sync Audiences

Sync audience data for an advertiser. The `accountId` in the URL is the **advertiser ID** (numeric, e.g. `25`) — the same `advertiserId` used when creating campaigns. Returns **202 Accepted** with an operation ID for tracking. Each member requires an `externalId` plus at least one hashed identifier.

**Operation:** `sync_audiences`
```http
POST /api/v2/buyer/advertisers/{accountId}/audiences/sync
{
  "audiences": [
    {
      "audienceId": "crm-high-value",
      "name": "High Value Customers",
      "add": [
        {
          "externalId": "user-001",
          "hashedEmail": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2"
        },
        {
          "externalId": "user-002",
          "uids": [{ "type": "uid2", "value": "uid2-token-value" }]
        }
      ],
      "consentBasis": "consent"
    }
  ],
  "deleteMissing": false
}
```

**Path Parameters:**
- `accountId` (string, required): Advertiser ID (numeric, e.g. `"25"`)

**Required Fields:**
- `audiences` (array): Audiences to sync
  - `audienceId` (string, required): Buyer's identifier for this audience

**Optional Fields per Audience:**
- `name` (string): Human-readable name
- `add` (array, max 10,000): Members to add (each needs `externalId` + at least one identifier)
- `remove` (array, max 10,000): Members to remove by `externalId`
- `delete` (boolean): When true, delete this audience entirely
- `consentBasis` (string): GDPR lawful basis — `consent`, `legitimate_interest`, `contract`, `legal_obligation`
- `deleteMissing` (boolean): When true, audiences not in this request are marked as deleted

**Response (202 Accepted):**
```json
{
  "success": true,
  "accountId": "25",
  "operationId": "550e8400-e29b-41d4-a716-446655440000",
  "taskId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Notes:**
- Processing is asynchronous — poll via `get_task` operation for progress (see [Tasks](#tasks))
- `status` values: `PROCESSING` (matching in progress), `READY` (available for targeting), `ERROR`, `TOO_SMALL` (below platform minimum)

#### List Audiences

List stored audiences for an account. Use this to check processing status after syncing.

**Operation:** `list_audiences`
```http
GET /api/v2/buyer/advertisers/{accountId}/audiences?take=50&skip=0
```
**As operation:**
```json
{ "operation": "list_audiences", "pathParams": { "advertiserId": "<id>" }, "params": { "take": "50", "skip": "0" } }
```

**Path Parameters:**
- `accountId` (string, required): Advertiser ID (numeric, e.g. `"25"`)

**Query Parameters (all optional):**
- `take` (number): Results per page (default: 50, max: 100)
- `skip` (number): Pagination offset (default: 0)

**Response:** each row is the audience **summary** shape — `audienceId`, `name`, `accountId`, `status`, `deleted`, `uploadedCount`, `matchedCount`, `createdAt`, `updatedAt`. `consentBasis` and `lastOperationStatus` are NOT on the summary — call `get_audience` for the full resource.

```json
{
  "audiences": [
    {
      "audienceId": "crm-high-value",
      "name": "High Value Customers",
      "accountId": "25",
      "status": "READY",
      "deleted": false,
      "uploadedCount": 1500,
      "matchedCount": 1200,
      "createdAt": "2026-02-24T10:00:00Z",
      "updatedAt": "2026-02-25T10:00:00Z"
    }
  ],
  "total": 1,
  "take": 50,
  "skip": 0
}
```

---

### Frequency Caps

Buyer-side frequency caps limit how often a user sees ads across all publishers/sales agents, scoped to an **advertiser**, **campaign**, or **creative**. These are distinct from publisher-side caps in the ADCP `target_overlay` — those are set by the seller on their own inventory.

**Shape of a single cap:**
```json
{ "max_impressions": 3, "window": { "interval": 1, "unit": "days" } }
```
- `max_impressions` (positive int): max impressions allowed per user within the window.
- `window.interval` (positive int) + `window.unit` (`"seconds"` | `"minutes"` | `"hours"` | `"days"` | `"campaign"`): rolling time window size. Matches the AdCP `Duration` shape.

**Multiple caps per entity are allowed and combine as AND.** For example, to enforce "no more than 3/day AND 10/hour" on a campaign, pass both:
```json
"frequencyCaps": [
  { "max_impressions": 3,  "window": { "interval": 1, "unit": "days"  } },
  { "max_impressions": 10, "window": { "interval": 1, "unit": "hours" } }
]
```

**Where to set them:** `frequencyCaps` is accepted on CREATE and UPDATE of:
- Advertisers — `create_advertiser`, `update_advertiser`
- Campaigns — `create_campaign`, `update_campaign`
- Creatives — `update_creative`

**Replace semantics (on UPDATE):**
- **Omit the field** → existing caps are left unchanged.
- **Pass `[]`** → all existing caps on that entity are cleared.
- **Pass a non-empty array** → the full set is replaced with the array supplied.

**Where to read them:**
- Single-GET (`get_advertiser`, `get_campaign`, `get_creative`) returns the `frequencyCaps` array.
- LIST endpoints (`list_advertisers`, `list_campaigns`, `list_creatives`) do NOT return `frequencyCaps` — call the single-GET to read them.

**Response cap shape** additionally includes `id`, `targetLevel` (`"ADVERTISER"` | `"CAMPAIGN"` | `"CREATIVE"`), `targetId`, `createdAt`, `updatedAt`, and `archivedAt` (null for active caps).

**Gathering inputs from the user:** if the user says "cap at 3 per day" or "no more than 10 impressions per week", translate to the shape above. Always confirm the scope (advertiser-wide vs. campaign-specific vs. creative-specific) before writing — they are independent and enforcement is the union of all applicable caps.

---

### Syndication

Syndicate audiences, event sources, or catalogs to ADCP agents. Tracks status asynchronously via webhooks.

#### Syndicate Resource

```http
POST /api/v2/buyer/advertisers/{advertiserId}/syndicate
{
  "resourceType": "AUDIENCE",
  "resourceId": "aud_12345",
  "adcpAgentIds": ["agent-abc-123", "agent-def-456"],
  "enabled": true
}
```

**Request Body:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resourceType` | string | Yes | `AUDIENCE`, `EVENT_SOURCE`, or `CATALOG` |
| `resourceId` | string | Yes | ID of the resource to syndicate |
| `adcpAgentIds` | string[] | Yes | Array of ADCP agent ID strings (min 1) |
| `enabled` | boolean | Yes | Whether to enable or disable syndication |

**Response (201):** Returns the syndication status records for each agent.

#### Query Syndication Status

```http
GET /api/v2/buyer/advertisers/{advertiserId}/syndication-status?resourceType=AUDIENCE&status=SYNCING&limit=20&offset=0
```

**Query Parameters (all optional):**
| Parameter | Type | Description |
|-----------|------|-------------|
| `resourceType` | string | Filter by `AUDIENCE`, `EVENT_SOURCE`, or `CATALOG` |
| `resourceId` | string | Filter by specific resource ID |
| `adcpAgentId` | string | Filter by ADCP agent ID |
| `enabled` | string | Filter by `true` or `false` |
| `status` | string | Filter by `PENDING`, `SYNCING`, `COMPLETED`, `FAILED`, or `DISABLED` |
| `limit` | number | Max results (1-100, default 50) |
| `offset` | number | Pagination offset (default 0) |

---

### Tasks

Async operations (audience sync, media buy creation, etc.) return a task ID that can be polled for status.

#### Get Task Status

**Operation:** `get_task`
```http
GET /api/v2/buyer/tasks/{taskId}
```
**As operation:**
```json
{ "operation": "get_task", "pathParams": { "taskId": "<id>" } }
```

**Response:**
```json
{
  "task": {
    "taskId": "550e8400-e29b-41d4-a716-446655440000",
    "taskType": "audience_sync",
    "status": "completed",
    "resourceType": "audience",
    "resourceId": "aud_12345",
    "error": null,
    "response": { "audience_id": "aud_12345", "member_count": 15000 },
    "metadata": {},
    "retryAfterSeconds": null,
    "createdAt": "2026-01-15T10:30:00.000Z",
    "updatedAt": "2026-01-15T10:35:00.000Z"
  }
}
```

**Task types:** `audience_sync`, `media_buy_create`, `creative_sync`

**Status values:** `submitted`, `working`, `completed`, `failed`, `input-required`

**Error format** (AdCP-compatible, set when status is `failed`):
```json
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid budget value",
  "field": "packages[0].budget",
  "suggestion": "Budget must be positive",
  "recovery": "correctable"
}
```

**Notes:**
- Task IDs are UUIDs returned in 202 responses from async operations
- Poll this endpoint when webhooks are unavailable — use `retryAfterSeconds` for polling interval guidance
- `response` contains the original downstream response payload (varies by task type)
- Tasks are scoped to the caller's customer — you cannot access another customer's tasks

---

### Property Lists

Property lists define which inventory an advertiser targets (include lists) or avoids (exclude lists). They accept AdCP-typed identifiers covering websites, mobile apps (iOS / Android), and CTV apps (Roku, Fire TV, Samsung). Lists are scoped to an advertiser and automatically apply to all campaigns under that brand's targeting profile.

#### Identifier types (AdCP-aligned)

| Type | Resolves via | Example value |
|------|--------------|---------------|
| `domain` | `Domain.domain` (`SITE`) | `nytimes.com` |
| `subdomain` | `Domain.domain` (`SITE`) | `news.example.com` |
| `ios_bundle` | `App.bundle` (Apple App Store) | `com.facebook.katana` |
| `android_package` | `App.bundle` (Google Play) | `com.facebook.katana` |
| `apple_tv_bundle` | `App.bundle` (Apple App Store) | `com.netflix.Netflix` |
| `bundle_id` (generic fallback) | `App.bundle` (any store) | `com.example.app` |
| `apple_app_store_id` | `Domain.domain` (`APPLE_APP_STORE`) — numeric ID | `284882215` |
| `google_play_id` | `Domain.domain` (`GOOGLE_PLAY_STORE`) | `com.example.app` |
| `roku_store_id` | `Domain.domain` (`ROKU`) | `12` |
| `fire_tv_asin` | `Domain.domain` (`AMAZON`) | `B00X4WHP5E` |
| `samsung_app_id` | `Domain.domain` (`SAMSUNG`) | `G19173000091` |

A single mobile app may appear in a property list under multiple identifier types (e.g. `ios_bundle` AND `apple_app_store_id`); each is resolved independently against the AAO property registry / local DB.

Read-side normalization: `apple_tv_bundle` and `ios_bundle` share the same `App.bundle` storage (`appStore=APPLE_APP_STORE`), so an `apple_tv_bundle` write reads back as `ios_bundle` on subsequent `get_property_list` / `list_property_lists` responses. The generic `bundle_id` fallback similarly normalizes to the resolved app row's store-typed form (`ios_bundle` or `android_package`). Submit the type that best matches the AdCP property registry; expect the response to carry the canonical store-typed form.

#### Create Property List

Create a named include or exclude list. Identifiers are resolved to internal property records; anything that cannot be resolved is returned in `unresolvedIdentifiers`. Identifiers found in the AAO registry but not yet locally targetable are returned in `registeredIdentifiers`.

**Operation:** `create_property_list`
```http
POST /api/v2/buyer/advertisers/{advertiserId}/property-lists
```

**Request body — typed identifiers (preferred for mixed inputs):**
```json
{
  "name": "Q1 Campaign - Premium Inventory",
  "purpose": "include",
  "identifiers": [
    { "type": "domain", "value": "nytimes.com" },
    { "type": "ios_bundle", "value": "com.facebook.katana" },
    { "type": "android_package", "value": "com.facebook.katana" },
    { "type": "apple_app_store_id", "value": "284882215" },
    { "type": "roku_store_id", "value": "12" }
  ]
}
```

**Request body — domains-only shorthand:**
```json
{
  "name": "Q1 Campaign - Premium Publishers",
  "purpose": "include",
  "domains": ["nytimes.com", "cnn.com", "bbc.co.uk"]
}
```

`domains` and `identifiers` may both be provided in the same request; the combined total must be 1..100,000. `domains: [...]` is shorthand equivalent to `identifiers: [{type: "domain", value: ...}]`.

> ⚠️ **Use `create_property_list` for small lists only (~500 entries or fewer).** For larger lists, especially anything in the thousands, direct the user to the multipart upload endpoint described below. Batching a single logical list across multiple `create_property_list` calls produces multiple disconnected lists, not one merged list, and is fragile when the identifier set is large.
>
> ⚠️ **Mandatory verification.** After a successful create, you MUST immediately call `list_property_lists` for the same advertiser and confirm the new `listId` appears with the expected `propertyCount`. Report the verified count to the user. If the listId is absent or the count is unexpectedly low, treat the create as failed and surface that to the user. Do not paraphrase or summarize a write as successful without this server-confirmed check.

> **Cascade on create.** Creating a property list automatically pushes it to all currently-active media buys for the same advertiser (same behavior as `update_property_list`). The `cascadeSummary` in the response shows counts. For buys created after the list exists but without the list, use `attach_property_list_to_campaign`.

**As operation:**
```json
{ "operation": "create_property_list", "pathParams": { "advertiserId": "<id>" }, "body": { "name": "Q1 Mixed", "purpose": "include", "identifiers": [{ "type": "domain", "value": "nytimes.com" }, { "type": "ios_bundle", "value": "com.facebook.katana" }] } }
```

**Response (201):**
```json
{
  "listId": "42",
  "name": "Q1 Mixed",
  "purpose": "include",
  "identifiers": [
    { "type": "domain", "value": "nytimes.com" }
  ],
  "unresolvedIdentifiers": [
    { "type": "ios_bundle", "value": "com.facebook.katana" }
  ],
  "registeredIdentifiers": [],
  "domains": ["nytimes.com"],
  "unresolvedDomains": [],
  "registeredDomains": [],
  "propertyCount": 14,
  "resolutionSummary": {
    "totalRequested": 2,
    "resolvedCount": 1,
    "registeredCount": 0,
    "unresolvedCount": 1,
    "resolutionRate": 0.5
  },
  "createdAt": "2026-03-16T10:00:00.000Z",
  "updatedAt": "2026-03-16T10:00:00.000Z"
}
```

**Response field reference:**
- `identifiers` — typed `{type, value}[]` actually resolved to local Property rows. **Persisted** as catalog membership.
- `unresolvedIdentifiers` — typed identifiers with no matching local Property record. For app types, this means no `App` row (or `Domain` row, for store-ID types) matches the value. **Transient** — see persistence note below.
- `registeredIdentifiers` — typed identifiers found in the AAO registry but not yet locally targetable. Today only `domain` types can land here — non-domain types are not auto-registered with AAO. **Transient** — see persistence note below.
- `domains` / `unresolvedDomains` / `registeredDomains` — convenience views of the above filtered to `type: "domain"`. App identifiers (any non-domain type) are NOT in these arrays.
- `resolutionSummary` — counts and `resolutionRate` (0..1) over the deduplicated, normalized input.

**Persistence note:** only `identifiers` (the resolved set) is stored on the property list — as foreign-key links to existing `Property` rows. `unresolvedIdentifiers` and `registeredIdentifiers` are **transient output of the resolution call**, not persisted state. They appear in the response of any create/update that runs resolution and are then dropped — a subsequent GET will return empty arrays for them, and a name-only PUT will not return them either. To re-surface unresolved/registered values for the same list, re-submit the identifier set on a PUT.

**Always surface `resolutionSummary` to the user** so they know how many of their submitted identifiers will actually target.

#### Upload Property List (xlsx / csv)

For large lists, especially anything in the thousands, the user should upload an xlsx or csv file rather than batch identifiers through `create_property_list`. The agent does not drive this directly because the MCP `api_call` surface is JSON-only; instead, direct the user to call the multipart endpoint.

```http
POST /api/v2/buyer/advertisers/{advertiserId}/property-lists/upload
Content-Type: multipart/form-data

file:    <the xlsx or csv file>
name:    "Q1 2026 - Global Exclusions"
purpose: "exclude"
```

Parsing rules:
- xlsx, xls, or csv accepted (max 10MB)
- first column = identifier (additional columns ignored)
- header row auto-detected (skipped when the first cell matches `domain|identifier|url|host|app`)
- blank rows and whitespace-only cells dropped
- duplicates deduplicated (first-seen order preserved)
- hard limit: 100,000 identifiers after dedup

**Response (201)** matches `create_property_list` plus an `upload` block with `filename`, `sizeBytes`, `totalRows`, `skippedHeader`, and `parsedIdentifiers`. **Always surface `resolutionSummary` AND the upload counts** to the user so they can confirm the row count matches their file. Apply the same mandatory verification as `create_property_list`: after upload, call `list_property_lists` and confirm the new `listId` is present with the expected `propertyCount`.

When a user asks the agent to ingest a large exclusion list, the agent's response should be a short instruction with the URL above, the required form fields, and the example `curl`:

```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -F file=@exclusions.xlsx \
  -F name="Q1 2026 - Global Exclusions" \
  -F purpose=exclude \
  https://api.scope3.com/api/v2/buyer/advertisers/{advertiserId}/property-lists/upload
```

#### List Property Lists

**Operation:** `list_property_lists`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/property-lists?purpose=include
```
**As operation:**
```json
{ "operation": "list_property_lists", "pathParams": { "advertiserId": "<id>" }, "params": { "purpose": "include" } }
```

**Response:** each row is the property list **summary** shape — `listId`, `name`, `purpose`, `propertyCount`, `createdAt`, `updatedAt`. The resolved `domains[]`, `unresolvedDomains[]`, `registeredDomains[]`, `filters`, `resolutionSummary`, and `cascadeSummary` are NOT on the summary — call `get_property_list` for the full resource.

#### Get Property List

**Operation:** `get_property_list`
```http
GET /api/v2/buyer/advertisers/{advertiserId}/property-lists/{listId}
```
**As operation:**
```json
{ "operation": "get_property_list", "pathParams": { "advertiserId": "<id>", "listId": "<id>" } }
```

#### Update Property List

Update name and/or replace the identifier set entirely. Both `domains` and `identifiers` are optional; if either is provided, the existing identifier set on the list is replaced. When identifiers change, the response includes a `resolutionSummary` and a `cascadeSummary` with counts of active media buys re-synced.

> ⚠️ **Each update REPLACES the full identifier set.** Do NOT split the identifier set across multiple update calls; a second call will wipe out the first batch. For thousands of entries, direct the user to re-upload via the multipart endpoint instead.
>
> ⚠️ **Mandatory verification.** After a successful update that changed identifiers, call `get_property_list` and confirm the returned `propertyCount` matches your expectation. Report the verified count to the user. Do not summarize the update as successful without this server-confirmed check.

**Operation:** `update_property_list`
```http
PUT /api/v2/buyer/advertisers/{advertiserId}/property-lists/{listId}
```

**Request body — typed identifiers:**
```json
{
  "name": "Updated List Name",
  "identifiers": [
    { "type": "domain", "value": "nytimes.com" },
    { "type": "ios_bundle", "value": "com.facebook.katana" }
  ]
}
```

**Request body — domains-only shorthand:**
```json
{
  "name": "Updated List Name",
  "domains": ["nytimes.com", "washingtonpost.com"]
}
```

#### Property Lists Applied to a Campaign

Use this to answer "is an exclusion or inclusion list actually applied to this campaign?". Aggregates the property lists referenced by every package on the campaign's media buys (via `targeting_overlay.property_list`), so the response reflects the real applied state — not just lists defined on the advertiser.

**Operation:** `get_campaign_property_lists`
```http
GET /api/v2/buyer/campaigns/{campaignId}/property-lists
```
**As operation:**
```json
{ "operation": "get_campaign_property_lists", "pathParams": { "campaignId": "<id>" } }
```

**Response:**
```json
{
  "campaignId": "cmp_xyz",
  "propertyLists": [
    {
      "listId": "1738",
      "name": "Q1 2026 - Global Exclusions",
      "purpose": "exclude",
      "propertyCount": 52309,
      "createdAt": "2026-05-13T13:33:51.000Z",
      "updatedAt": "2026-05-13T13:33:51.000Z",
      "viaMediaBuys": [
        { "mediaBuyId": "mb_abc", "packageIds": ["pkg_1", "pkg_2"] }
      ]
    }
  ],
  "summary": { "totalLists": 1, "includeCount": 0, "excludeCount": 1 }
}
```

**An empty `propertyLists` array is the authoritative answer that no lists are applied** — do not infer presence from the campaign brief, constraints, or anything else. If the user asks whether exclusions are in force, call this op and report exactly what it returns.

Alternative: `get_campaign` accepts `params.includePropertyLists=true`, which embeds the same `{ propertyLists, summary }` payload as a `propertyLists` field on the campaign response. Use that when you're already fetching the campaign and want to save a round trip.

#### Retroactively Attach a Property List to a Campaign

**Operation:** `attach_property_list_to_campaign`
```http
POST /api/v2/buyer/campaigns/{campaignId}/property-lists
```
**As operation:**
```json
{ "operation": "attach_property_list_to_campaign", "pathParams": { "campaignId": "<id>" }, "body": { "propertyListId": "<listId>" } }
```

Use this when a property list was created **after** the campaign's media buys were already active, or when buys were created without the list in their targeting overlay. Only **include lists** can be attached this way; exclude lists use a different targeting path and are not cascaded via this endpoint. The list must be an include list configured for the campaign's advertiser; attaching a list from a different advertiser or an exclude list returns a 400.

**Response (200):**
```json
{
  "campaignId": "campaign_abc",
  "propertyListId": "1756",
  "cascade": {
    "totalMediaBuys": 3,
    "updatedCount": 3,
    "failedCount": 0,
    "skippedCount": 0
  }
}
```

After calling this op, **always call `get_campaign_property_lists`** to confirm the list now appears. A `failedCount > 0` means some buys couldn't be updated: surface that to the user and tell them to retry or check that the buys are still ACTIVE.

#### Delete Property List

Archives the property list. The list remains associated with the advertiser but is no longer active.

**Operation:** `delete_property_list`
```http
DELETE /api/v2/buyer/advertisers/{advertiserId}/property-lists/{listId}
```
**As operation:**
```json
{ "operation": "delete_property_list", "pathParams": { "advertiserId": "<id>", "listId": "<id>" } }
```

**Recommended workflow:**
1. Create a property list with initial identifiers (web, mobile, CTV, or any mix).
2. Use the check endpoint (below) to validate domain entries against the AAO registry; non-domain types (mobile/CTV) are not validated upstream and are returned in the `assess` bucket.
3. Update the list based on check results (remove blocked domains, apply canonical corrections).
4. All campaigns under the advertiser automatically inherit the targeting.
5. Property lists are automatically passed to sales agents during product discovery via the ADCP `property_list` field, including all typed identifiers.

#### Resolve Property List (ADCP)

Returns a property list in ADCP `GetPropertyListResponse` format. The `identifiers` array contains every typed identifier the list resolves to (domains, mobile bundles/store IDs, CTV store IDs). Used by sales agents to resolve a `PropertyListReference` received during product discovery. Authenticated via HMAC token (not platform auth).

```http
GET /lists/{listId}
Authorization: Bearer {auth_token}
```

**Response:**
```json
{
  "list": {
    "list_id": "123",
    "name": "Premium inventory"
  },
  "identifiers": [
    { "type": "domain", "value": "nytimes.com" },
    { "type": "domain", "value": "cnn.com" },
    { "type": "ios_bundle", "value": "com.spotify.music" },
    { "type": "android_package", "value": "com.spotify.music" },
    { "type": "apple_app_store_id", "value": "324684580" }
  ],
  "resolved_at": "2026-03-17T12:00:00.000Z",
  "cache_valid_until": "2026-03-18T12:00:00.000Z"
}
```

#### Check Property List

Validate a mixed list of typed identifiers against the AAO Community Registry. For `domain` entries, identifies blocked entries (ad servers, CDNs, trackers), normalizes URLs (strips www/m prefixes), removes duplicates, and flags unknown values. Non-domain types (mobile/CTV apps) are not currently checked against AAO and are returned in the `assess` bucket pending upstream support.

**Operation:** `check_property_list`
```http
POST /api/v2/buyer/property-lists/check
```

**Request body — typed identifiers:**
```json
{
  "identifiers": [
    { "type": "domain", "value": "nytimes.com" },
    { "type": "domain", "value": "www.cnn.com" },
    { "type": "domain", "value": "doubleclick.net" },
    { "type": "ios_bundle", "value": "com.facebook.katana" }
  ]
}
```

**Request body — domains-only shorthand:**
```json
{
  "domains": ["nytimes.com", "www.cnn.com", "doubleclick.net", "unknown-site.xyz"]
}
```

`domains` and `identifiers` may both be provided in the same request; combined total must be 1..100,000.

**As operation:**
```json
{ "operation": "check_property_list", "body": { "identifiers": [{ "type": "domain", "value": "nytimes.com" }, { "type": "ios_bundle", "value": "com.facebook.katana" }] } }
```

**Response:**
```json
{
  "summary": { "total": 5, "remove": 1, "modify": 1, "assess": 2, "ok": 1 },
  "remove": [
    { "input": "doubleclick.net", "canonical": "doubleclick.net", "reason": "blocked", "domain_type": "ad_server", "identifier": { "type": "domain", "value": "doubleclick.net" } }
  ],
  "modify": [
    { "input": "www.cnn.com", "canonical": "cnn.com", "reason": "www prefix removed", "identifier": { "type": "domain", "value": "www.cnn.com" } }
  ],
  "assess": [
    { "domain": "unknown-site.xyz", "identifier": { "type": "domain", "value": "unknown-site.xyz" } },
    { "domain": "com.facebook.katana", "identifier": { "type": "ios_bundle", "value": "com.facebook.katana" } }
  ],
  "ok": [
    { "domain": "nytimes.com", "source": "registry", "identifier": { "type": "domain", "value": "nytimes.com" } }
  ],
  "reportId": "rpt_abc123",
  "reportIds": ["rpt_abc123"]
}
```

**Result buckets:**
- `remove`: Domains to remove — duplicates or blocked (ad servers, CDNs, trackers, intermediaries). Non-domain types never appear here.
- `modify`: Domains that were normalized (e.g. `www.example.com` → `example.com`). Use the `canonical` value. Non-domain types never appear here.
- `assess`: Unknown domains not in the registry and not blocked — may need manual review. **All non-domain identifiers (mobile/CTV apps) land here** (AAO does not currently check them).
- `ok`: Domains found in the registry with no issues. Non-domain types never appear here.

Each bucket entry carries an `identifier: { type, value }` field that mirrors the input — use this rather than the `domain`/`input` field to disambiguate types.

**`reportId` / `reportIds`:** present only when at least one `domain` entry was submitted. Omitted entirely when the request contained only non-domain types (no AAO call is made). When domain input is chunked into multiple registry calls, `reportIds` contains all chunk IDs and `reportId` equals `reportIds[0]` for back-compat.

**Limits:** 1–100,000 entries per request, chunked server-side into 10,000-domain registry calls.

#### Get Property Check Report

Retrieve a stored property check report by ID. Reports expire after 7 days.

**Operation:** `get_property_list_report`
```http
GET /api/v2/buyer/property-lists/reports/{reportId}
```
**As operation:**
```json
{ "operation": "get_property_list_report", "pathParams": { "reportId": "<id>" } }
```

**Response:**
```json
{
  "summary": { "total": 4, "remove": 1, "modify": 1, "assess": 1, "ok": 1 }
}
```

### Buyer Activity

Use the typed `list_buyer_activity` read for API activity questions. For buyer
accounts enrolled in the Activity Console beta, it opens the portable Activity
page inside MCP clients that support MCP Apps and returns a compact
model-readable summary. Calls is the default view and covers captured REST,
MCP, and A2A operations, workload identity, application outcome, latency,
environment, and correlation IDs. Selecting a call reveals authorized,
redacted request fields and available response/error/header/timing detail and
can hand the exact activity UID back to the conversation for Murph debugging.
For accounts outside the beta, the same read falls back to recorded Changes;
state that explicitly and never describe that feed as complete call history.

```json
{
  "view": "calls",
  "outcome": "failed",
  "surface": "mcp"
}
```

The page owns `ui://agentic-api/buyer-activity/mcp-app.html`; do not call or
invent a separate `open_buyer_activity` tool. Hosts without MCP Apps support
should summarize the typed result and may query the REST operations below.

### Changes feed

Lists audit log events for the buyer — campaign changes, creative updates, media buy transitions, advertiser edits, and other buyer-side actions. Customer-scoped.

```http
GET /api/v2/buyer/audit-logs?take=50&skip=0
```
**As operation:**
```json
{ "operation": "list_activity", "params": { "take": "50" } }
```

Query parameters (all optional):
- `startDate`, `endDate` — ISO timestamps to bound the range
- `advertiserId` — filter to a single advertiser
- `campaignId` — filter to a single campaign (includes its media buys, creatives, etc.)
- `resourceTypes` — repeated (`?resourceTypes=CAMPAIGN&resourceTypes=CREATIVE`) or comma-separated
- `take` (max 500, default 50 over REST; default 100 and max 100 through MCP), `skip`

**Response:**
```json
{
  "data": {
    "logs": [
      {
        "id": 1,
        "timestamp": "2026-04-01T00:00:00Z",
        "action": "UPDATE",
        "resourceType": "CAMPAIGN",
        "resourceId": "camp_abc123",
        "resourceName": "Spring Promo",
        "advertiserId": 42,
        "userEmail": "user@example.com",
        "description": "Updated campaign Spring Promo"
      }
    ],
    "total": 123
  },
  "meta": { "pagination": { "skip": 0, "take": 10, "total": 123, "hasMore": true } }
}
```

The `list_activity` operation is the Changes view inside Buyer Activity. It
shows who changed what across the buyer's advertisers and campaigns, whether
the actor was a person, the agent, or automation, and whether each action
succeeded, was denied, or failed. It is not a complete API-call ledger and must
not be described as “all API activity.” Use `list_buyer_activity` for “show my
activity,” API failures, workloads, and latency; answer a focused one-off
who/what/when change question directly when a Page is unnecessary.

---

## Error Handling

### Hard Failures vs. No Products Found

These are two distinct outcomes — do NOT conflate them:

**Hard failures** — the agent returned an actual error response. Examples:
- Auth or tenant context errors
- Authentication required
- Data corruption errors
- MCP endpoint not responding

These surface as non-2xx HTTP responses or error payloads. Treat as errors that need investigation.

**Soft failures / no products** — the agent responded successfully (HTTP 200) but returned 0 products. This is **not an error**. It means the brief did not match available inventory. Do NOT tell the user the agent "failed." See "When discovery returns no products" above for how to handle this.

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
  "content": [{ "type": "text", "text": "Budget is below the minimum" }],
  "structuredContent": {
    "code": "VALIDATION_ERROR",
    "message": "Budget is below the minimum",
    "field": "packages[0].budget",
    "suggestion": "Minimum budget is $100"
  },
  "isError": true
}
```

- `code` — machine-readable error code (see table below)
- `message` — human-readable description
- `field` — (optional) field path (e.g. `packages[0].budget`)
- `suggestion` — (optional) suggested fix

### Error Codes

| Code | HTTP Status | Resolution |
|------|-------------|------------|
| `VALIDATION_ERROR` | 400 | Check request body against schema |
| `UNAUTHORIZED` | 401 | Verify API key/auth |
| `ACCESS_DENIED` | 403 | Check permissions |
| `NOT_FOUND` | 404 | Verify resource ID exists |
| `CONFLICT` | 409 | Resource already exists (e.g., brand) |
| `RATE_LIMITED` | 429 | Wait and retry |
| `INVALID_STATE` | 400 | Operation not allowed in current state |
| `CUSTOMER_SCOPE_REQUIRED` | 403 | The session has access to multiple customers and a mutation was attempted without scoping. Call the `customer_switch` MCP tool with one of the `customerId`s from `details.availableCustomers` (already in the error), then retry the ORIGINAL request unchanged. Do NOT invent other causes (flag not enabled, schema mismatch, permissions, allowlist) — this error is exclusively about session scoping. Do NOT call `customer_list` first; the choices are already in the error payload. |
| `INTERNAL_ERROR` | 500 | Contact support |

---

## Notifications

Notifications are events about resources you manage — campaigns going unhealthy, creatives syncing, agents registering, etc. They follow a `resource.action` taxonomy (e.g., `campaign.unhealthy`, `creative.sync_failed`).

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
- `types` (comma-separated): Filter by event types (e.g., `campaign.unhealthy,creative.sync_failed`)
- `campaignId` (string): Filter by campaign
- `creativeId` (string): Filter by creative
- `limit` (number): Results per page (default: 50, max: 100)
- `offset` (number): Pagination offset

**Response:** each row is the notification **summary** shape — `id`, `type`, `status`, `read`, `acknowledged`, `messagePreview`, `createdAt`. The full `data` payload is NOT on the summary.

```json
{
  "notifications": [
    {
      "id": "notif_1709123456_abc123",
      "type": "campaign.unhealthy",
      "status": "warning",
      "messagePreview": "Campaign \"Q1 CTV\" is unhealthy",
      "read": false,
      "acknowledged": false,
      "createdAt": "2026-03-01T12:00:00Z"
    }
  ],
  "totalCount": 15,
  "unreadCount": 3,
  "hasMore": false
}
```

`messagePreview` is the summary text (truncated at 200 characters). The full `data` payload (campaignId, campaignName, etc.) is not returned by the list — fetch the underlying resource (campaign, creative, etc.) by ID for the full context.

### Mark Notification as Read

**Operation:** `read_notification`
```http
POST /api/v2/notifications/{notificationId}/read
```
**As operation:**
```json
{ "operation": "read_notification", "pathParams": { "notificationId": "<id>" } }
```

Marks a single notification as seen. No request body required.

### Mark Notification as Acknowledged

**Operation:** `acknowledge_notification`
```http
POST /api/v2/notifications/{notificationId}/acknowledge
```
**As operation:**
```json
{ "operation": "acknowledge_notification", "pathParams": { "notificationId": "<id>" } }
```

Marks a notification as dealt with. Acknowledged notifications are automatically cleaned up after 90 days. No request body required.

### Mark All Notifications as Read

**Operation:** `read_all_notifications`
```http
POST /api/v2/notifications/read-all
```
**As operation:**
```json
{ "operation": "read_all_notifications" }
```

**Optional body:**
```json
{ "brandAgentId": 123 }
```

If `brandAgentId` is provided, only marks notifications for that agent as read. Otherwise marks all unread notifications as read.

### Proactive Notification Setup

Unread notifications are automatically included in `help` and `ask_about_capability` tool responses. To ensure your AI agent surfaces them to users at the start of every session, add the following to your client configuration:

- **Claude Desktop**: Create a Project and add to the project instructions: `When using Scope3 tools, always start by calling the help tool. The response includes unread notifications — summarize those for the user before answering their question.`
- **Claude Code**: Add the same instruction to your `CLAUDE.md` or project instructions.
- **API / Custom Agent**: Add it to your system prompt.
- **ChatGPT Custom GPT**: Add it to your Custom GPT's instructions.

---

## Optimization Suggestions

Optimization suggestions are generated by Scope3's AI optimization model for active media buys. When `optimizationApplyMode` is `"MANUAL"` (the default), suggestions wait for human approval before being applied.

### List Optimization Suggestions
```http
GET /api/v2/buyer/optimization-suggestions?status=HIL_WAITING&limit=20&offset=0
```

**Query Parameters (all optional):**
- `campaignId` (number): Filter by campaign
- `mediaBuyId` (number): Filter by media buy
- `status` (string): Filter by status. Values: `CREATED`, `CREATED_FAILED`, `INGESTED`, `INGESTED_FAIL`, `HIL_WAITING`, `HIL_APPROVED`, `HIL_REJECTED`, `APPLIED`, `APPLIED_FAIL`, `REJECTED`, `EXPIRED`
- `limit` (number): Results per page (default: 20, max: 100)
- `offset` (number): Pagination offset

**Response:**
```json
{
  "suggestions": [
    {
      "suggestionId": "550e8400-e29b-41d4-a716-446655440000",
      "campaignId": "123",
      "mediaBuyId": "456",
      "optimizationMetric": "emissions",
      "status": "HIL_WAITING",
      "productAdjustmentCount": 3,
      "bidAdjustmentCount": 1,
      "createdAt": "2026-03-20T00:00:00.000Z",
      "statusUpdatedAt": "2026-03-20T01:00:00.000Z"
    }
  ],
  "totalCount": 5,
  "hasMore": false
}
```

**Display Requirements:** For each suggestion, show the suggestion ID, campaign/media buy IDs, optimization metric, current status, number of product and bid adjustments, and when it was created.

### Get Optimization Suggestion
```http
GET /api/v2/buyer/optimization-suggestions/{suggestionId}
```

Returns the full suggestion payload including product adjustments, bid adjustments, scaling factor, and rationale.

**Response:**
```json
{
  "suggestionId": "550e8400-e29b-41d4-a716-446655440000",
  "optimizerRunId": "run-abc123",
  "campaignId": "123",
  "mediaBuyId": "456",
  "suggestionTimestamp": "2026-03-20T00:00:00.000Z",
  "modelVersion": "v1.0",
  "campaignSummary": { "name": "Q1 CTV Campaign" },
  "optimizationMetric": "emissions",
  "mediaBuyScalingFactor": 1.2,
  "mediaBuyScalingRationale": "Scale up for improved emissions performance",
  "productAdjustments": [
    { "productId": "prod_1", "action": "add", "reason": "High performance" }
  ],
  "bidAdjustments": [
    { "productId": "prod_2", "bidChange": 0.5, "reason": "Reduce CPM" }
  ],
  "createdAt": "2026-03-20T00:00:00.000Z",
  "status": "HIL_WAITING",
  "statusReason": null,
  "statusError": null,
  "statusUpdatedAt": "2026-03-20T01:00:00.000Z"
}
```

**Display Requirements:** Show all suggestion details including product adjustments (with actions and reasons), bid adjustments, scaling factor and rationale, and the current status. Present the data clearly so the user can make an informed approve/reject decision.

### Approve Optimization Suggestion
```http
POST /api/v2/buyer/optimization-suggestions/{suggestionId}/approve
```

Approves a suggestion that is in `HIL_WAITING` status. No request body required.

**Response:**
```json
{
  "suggestionId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "HIL_APPROVED"
}
```

**Error:** Returns 409 Conflict if the suggestion is not in `HIL_WAITING` status.

### Reject Optimization Suggestion
```http
POST /api/v2/buyer/optimization-suggestions/{suggestionId}/reject
```

Rejects a suggestion that is in `HIL_WAITING` status.

**Optional body:**
```json
{ "reason": "Not aligned with current strategy" }
```

**Response:**
```json
{
  "suggestionId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "HIL_REJECTED"
}
```

**Error:** Returns 409 Conflict if the suggestion is not in `HIL_WAITING` status.

---

## Common Mistakes to Avoid

1. **Creating campaign without advertiser** — Always create/verify advertiser first
2. **Skipping product discovery** — Always use `discover_products` operation to discover products; use `browse_discovery` operation to browse more
3. **Optimization without event source** — You need an event source (`eventSourceId`) before creating a campaign with event-based optimization goals
4. **Optimization without conversion data** — System needs events logged via event sources to optimize for ROAS/conversions
5. **Forgetting to execute** — Campaigns start in DRAFT status; must use `execute_campaign` operation
6. **Wrong endpoint path** — Always use `/api/v2/buyer/` prefix
7. **Creating advertiser without brand** — `brand` is required. If brand resolution succeeds through enrichment, the advertiser already exists; show `brandWarning` and the enriched details, then offer `saveBrand: true` on `update_advertiser` only if the user wants registry persistence. If no enrichment data is found but the user confirms the advertiser name and brand domain, retry with `saveBrand: true`. Only direct the user to external registration if there is no confirmed name/domain to save or the registry save fails.
8. **Auto-selecting products for the user** — When the user wants to browse/select inventory, ALWAYS present discovery results and let them choose
9. **Defaulting to a configuration without asking** — When the user says "create a campaign" without specifying how to configure it, ask them to choose (product discovery or performance metrics)
10. **Fabricating field values** — NEVER guess or make up values for required fields. Always ask the user or use values from previous API responses
11. **Making multiple API calls in one turn** — ONE discovery/mutating call per turn. Present results, END YOUR TURN, wait for the user.
12. **Missing bid price for non-fixed pricing** — If a product's pricing option has `isFixed: false`, `bidPrice` is REQUIRED in the `add_discovery_products` request. Read it from the product's `pricingOptions` (`rate` or `floorPrice`) in the discovery response. Do NOT ask the user — the value comes from the product data.
13. **Summarizing list responses as prose** — When listing advertisers, sales agents, or campaigns, NEVER reduce the response to a sentence like "You have 13 advertisers." Always show the structured per-item details specified in the Display Requirements for that endpoint. The user needs to see each item's operational details, not a count.
14. **Using user-provided account IDs for linking** — NEVER use an account ID or account name that the user provides verbally. For official adapters, IDs must come from `list_storefront_connection_accounts` or `list_storefront_connection_account_mappings`; for legacy/third-party external AdCP sources, IDs must come from `list_available_accounts`. If an account does not appear on the appropriate discovery surface, tell the user it was not found — do NOT pretend to link it.
15. **Missing credentialId with multiple credentials** — When a customer has multiple credentials for the same agent, the `list_available_accounts` operation requires `credentialId`. If omitted, the API returns an error with available credential IDs. Present those to the user and ask which to use, then retry with the chosen `credentialId`.
