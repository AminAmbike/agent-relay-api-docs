# Agent Relay — Advertiser (Business) API Contract

The complete backend contract for the **advertiser/business** side of Agent Relay (formerly
AttentionMarket). Everything a business does — sign up, log in, create & edit capabilities/campaigns,
top up a wallet, connect Stripe, withdraw earnings, and review the conversations/feedback/insights their
capability generated — is covered here.

> **Status:** authoritative as of the extraction date below. Contracts were read directly from the live
> edge-function source. Where a field name no longer matches what it holds (we pivoted from ad-serving to
> capability-serving), the **Description** column says what the field *actually* is — trust the
> description over the name.

---

## How to read this doc (humans **and** agents)

This file is the single source of truth. It is plain Markdown so an AI agent can be pointed at the raw
file and parse it directly; no renderer required.

**Every endpoint follows the same shape:**

1. A fenced ` ```yaml ` **metadata block** — `method`, `path`, `auth`, `proxy`. Machine-parseable.
2. **Purpose** — one line.
3. **Request** — a field table (`Field | Type | Req | Description`) + a canonical `ts` interface.
4. **Response** — a field table (`Field | Type | Description`) + a canonical `ts` interface.
5. **Errors** — `HTTP | code | when`.
6. **Example** — a real `curl` + JSON.

**Conventions used in tables**
- `Req` = required. `↳` prefix = a field nested inside the array/object on the row above it.
- Types are TypeScript-style. `uuid`/`int`/`cents`/`dollars` are annotations, not literal TS types.
- ⚠️ callouts flag units, ignored fields, and misleading names.

**Auth tag legend** (see [Auth model](#auth-model)):
- `jwt:uid` — advertiser resolved from **`advertisers.auth_user_id == JWT.sub`**.
- `jwt:email` — advertiser resolved from **`advertisers.contact_email == JWT.email`**.
- `public` — no auth (signup/login) or HMAC-signed state (Stripe callback).

---

## Base & conventions

- **Base URL:** `https://peruwnbrqkvmrldhpoom.supabase.co/functions/v1/<slug>`
- **Anon key required** on every call as the `apikey` header (Supabase gateway requirement), in addition
  to any `Authorization` bearer.
- **Frontend proxy:** the web app calls most functions through its own server at
  `POST /api/proxy/<slug>` (which injects the anon key and forwards the bearer). `campaign-logs`, the
  auth endpoints, and the Stripe redirects are called directly.
- **CORS:** open (`Access-Control-Allow-Origin: *`); every function answers `OPTIONS` preflight with 200.
- **Content-Type:** `application/json` for all except `upload-knowledge-docs` (multipart/form-data).
- **Error envelope:** `{ "error": "<code|message>", "message"?: "<human>" }` with an HTTP 4xx/5xx status.
- **Money units (READ THIS):** `wallet_balance` and campaign `budget`/`budget_spent` are in **DOLLARS**
  (numeric). `earnings_balance`, `lifetime_earnings`, `net_cents`/`gross_cents`/`fee_cents`, and
  `amount_cents` are in **CENTS** (integer). Divide cents by 100 for display.

---

## Auth model

Agent Relay has **three** advertiser-resolution strategies. This is the single most important thing to
get right: an advertiser account is only reachable by the strategy its endpoint uses.

### Getting a JWT
`POST /functions/v1/advertiser-login` with `{ contact_email, password }` returns
`{ access_token, refresh_token, expires_in, ... }`. Use `access_token` as the bearer on all
authenticated calls. (It is a standard Supabase Auth JWT; `sub` = the auth user id, `email` = the login
email.)

### Strategy A — `jwt:uid` (canonical)
Resolves the advertiser via **`advertisers.auth_user_id == JWT.sub`** (service-role read; the server
never trusts an `advertiser_id` from the request body — it only uses it to disambiguate multi-account
users, and ownership-checks it). Implemented by `_shared/advertiser-auth.ts › authenticateAdvertiser`.

- 401 — missing/invalid `Authorization` header, or invalid/expired token.
- 403 — no advertiser linked to this user, OR the given `advertiser_id` isn't owned, OR account inactive.
- 400 — user owns multiple advertisers and didn't pass `advertiser_id`.

> ⚠️ If an advertiser row has `auth_user_id = null`, **every `jwt:uid` endpoint returns 403** even with a
> valid login — the account must be linked. (This is a real footgun; several seed/test accounts were
> created without a link.)

### Strategy B — `jwt:email`
Resolves the advertiser via **`advertisers.contact_email == JWT.email`**. Any `advertiser_id` in the body
is ignored. 401 on bad token; 403/404 when no active advertiser matches the email.

### Strategy C — `public`
`advertiser-signup` and `advertiser-login` take no auth. `stripe-connect-callback` is a browser redirect
authenticated by an HMAC-signed `state` param, not a JWT.

### Which endpoint uses which

| Endpoint | Auth |
|---|---|
| advertiser-signup, advertiser-login | `public` |
| advertiser-profile | `jwt:uid` |
| advertiser-campaigns, campaign-update, advertiser-stats | `jwt:uid` |
| advertiser-billing, add-funds | `jwt:uid` |
| campaign-create, agent-test, upload-knowledge-docs | `jwt:email` |
| campaign-logs (all actions) | `jwt:email` |
| request-withdrawal, stripe-connect-init, stripe-connect-disconnect | `jwt:email` |
| stripe-connect-callback | `public` (HMAC state) |

> `advertiser-stats` also accepts an `X-Advertiser-Key: <api_key>` header as an alternative to the JWT
> (programmatic access).

> **Password reset for advertisers is not yet wired** (TODO — tracked separately). The `*-password` /
> `verify-email` functions in the platform target developer accounts, not advertisers.

---

## Endpoint index

| # | Endpoint | Method | Auth | Purpose |
|---|---|---|---|---|
| 1 | [advertiser-signup](#advertiser-signup) | POST | public | Create a business account |
| 2 | [advertiser-login](#advertiser-login) | POST | public | Log in, get a JWT |
| 3 | [advertiser-profile](#advertiser-profile) | POST | jwt:uid | Read/update business profile |
| 4 | [campaign-create](#campaign-create) | POST | jwt:email | Create a capability/campaign |
| 5 | [campaign-update](#campaign-update) | POST | jwt:uid | Edit / pause / archive |
| 6 | [advertiser-campaigns](#advertiser-campaigns) | POST | jwt:uid | List campaigns |
| 7 | [advertiser-stats](#advertiser-stats) | GET | jwt:uid | Dashboard KPIs |
| 8 | [agent-test](#agent-test) | POST | jwt:email | Test-drive your own capability |
| 9 | [upload-knowledge-docs](#upload-knowledge-docs) | POST | jwt:email | Attach business docs |
| 10 | [advertiser-billing](#advertiser-billing) | POST | jwt:uid | Wallet, earnings, history |
| 11 | [add-funds](#add-funds) | POST | jwt:uid | Top up wallet (Stripe) |
| 12 | [request-withdrawal](#request-withdrawal) | POST | jwt:email | Withdraw earnings |
| 13 | [stripe-connect-init](#stripe-connect-init) | POST | jwt:email | Start Stripe Connect |
| 14 | [stripe-connect-callback](#stripe-connect-callback) | GET | public | Stripe OAuth return |
| 15 | [stripe-connect-disconnect](#stripe-connect-disconnect) | POST | jwt:email | Unlink Stripe |
| 16 | [campaign-logs](#campaign-logs) | POST | jwt:email | Conversations, feedback, insights |

---

# 1. Account & auth

## advertiser-signup
```yaml
method: POST
path:   /functions/v1/advertiser-signup
auth:   public
proxy:  (called directly)
```
**Purpose.** Create a business account (advertiser row + Supabase Auth user).

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| contact_email | string | yes | Login email; becomes `advertisers.contact_email`. |
| company_name | string | yes | Business display name. |
| contact_name | string | yes | Primary contact person. |
| password | string | yes | 8–128 chars, must contain a letter and a digit. |
| website | string | no | Business URL. |
| industry | string | no | Free-text industry. |

```ts
interface SignupRequest {
  contact_email: string;
  company_name: string;
  contact_name: string;
  password: string;
  website?: string;
  industry?: string;
}
```

### Response `201`
| Field | Type | Description |
|---|---|---|
| advertiser_id | uuid | New advertiser id. |
| api_key | string | Programmatic key, format `adv_<64 hex>`. Also usable as `X-Advertiser-Key`. |
| company_name | string | Echoed. |
| status | string | `"active"`. |
| message | string | Human confirmation. |

```ts
interface SignupResponse {
  advertiser_id: string;
  api_key: string;
  company_name: string;
  status: "active";
  message: string;
}
```

### Errors
| HTTP | code | When |
|---|---|---|
| 400 | validation_error | missing field / bad email / weak password |
| 409 | duplicate_error | email already registered (body also returns `advertiser_id`) |
| 500 | auth_error / database_error / internal_error | auth-user or DB insert failed (auth user rolled back) |

---

## advertiser-login
```yaml
method: POST
path:   /functions/v1/advertiser-login
auth:   public
proxy:  (called directly)
```
**Purpose.** Verify credentials and mint a Supabase JWT for all authenticated calls.

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| contact_email | string | yes | Login email. Alias: `email` (either key accepted). |
| password | string | yes | Checked against `advertisers.password_hash` (PBKDF2). |

```ts
interface LoginRequest { contact_email?: string; email?: string; password: string }
```

### Response `200`
| Field | Type | Description |
|---|---|---|
| success | boolean | `true` on full success. |
| access_token | string | The JWT — use as `Authorization: Bearer`. |
| refresh_token | string | For token refresh. |
| expires_in | number | Access-token lifetime (seconds). |
| advertiser_id | uuid | Resolved advertiser. |
| company_name | string | Display name. |
| status | string | Account status. |

> ⚠️ **Degraded success:** if the password is valid but JWT minting fails, the endpoint still returns
> `200` with `{ advertiser_id, company_name, status, warning }` and **no** `access_token`. Treat missing
> `access_token` as "logged-in identity known, but no session token" — surface the `warning`.

```ts
interface LoginResponse {
  success?: true;
  access_token?: string;
  refresh_token?: string;
  expires_in?: number;
  advertiser_id: string;
  company_name: string;
  status: string;
  warning?: string; // present only on degraded success
}
```

### Errors
| HTTP | code | When |
|---|---|---|
| 400 | validation_error | missing email/password |
| 401 | invalid_credentials | no such advertiser or wrong password |
| 401 | password_not_set | advertiser has no `password_hash` |
| 403 | account_suspended | status ≠ `active` |
| 500 | internal_server_error | unexpected |

---

## advertiser-profile
```yaml
method: POST
path:   /functions/v1/advertiser-profile
auth:   jwt:uid
proxy:  /api/proxy/advertiser-profile
```
**Purpose.** Read or update the business profile. Read = empty body; update = send `updates`.

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| updates | object | no | Present → update mode. Only whitelisted keys apply (below). |
| ↳ company_name | string | no | |
| ↳ contact_name | string | no | |
| ↳ website | string | no | |
| ↳ industry | string | no | |
| ↳ sms_opted_in | boolean | no | SMS notifications opt-in. |
| ↳ sms_phone_number | string | no | Phone for SMS. |
| advertiser_id | uuid | no | Disambiguate multi-account only; ownership-checked. |

```ts
interface ProfileRequest {
  updates?: {
    company_name?: string; contact_name?: string; website?: string;
    industry?: string; sms_opted_in?: boolean; sms_phone_number?: string;
  };
  advertiser_id?: string;
}
```

### Response `200` (same shape for read and update)
| Field | Type | Description |
|---|---|---|
| advertiser | object | The profile. |
| ↳ id | uuid | |
| ↳ company_name, contact_name, contact_email | string | |
| ↳ website, industry | string \| null | |
| ↳ status | string | `active` etc. |
| ↳ wallet_balance | dollars | Prepaid ad-spend balance. |
| ↳ earnings_balance | cents | Withdrawable seller earnings. |
| ↳ sms_opted_in | boolean | |
| ↳ sms_phone_number | string \| null | |
| ↳ stripe_connect_status | string | `not_connected` \| `connected` \| `revoked`. |
| ↳ stripe_account_id | string \| null | Connected account id. |
| ↳ stripe_connected_at | timestamptz \| null | |
| ↳ created_at | timestamptz | |

### Errors
| HTTP | code | When |
|---|---|---|
| 405 | Method Not Allowed | non-POST |
| 401 / 403 / 400 | (auth) | see [Strategy A](#strategy-a--jwtuid-canonical); 400 also = "No editable fields in updates" |
| 500 | — | resolve/DB failure |

---

# 2. Campaign management

## campaign-create
```yaml
method: POST
path:   /functions/v1/campaign-create
auth:   jwt:email
proxy:  /api/proxy/campaign-create
```
**Purpose.** Create a capability/campaign (the wizard submit). Inserts a `campaigns` row + one `ad_units`
row and generates a matching embedding.

> ⚠️ `capability_type` is accepted but **ignored** — everything is created as `dynamic`.
> ⚠️ `trigger_contexts` / `situational_signals` sent here are **ignored**; they're generated server-side
> from your text via the discovery-signal generator.

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| name | string | yes | Campaign/capability name. |
| budget | dollars | yes | $10–$100,000. |
| bid_cpc | dollars | yes | $0.01–$100 (max bid per click). |
| intent_description | string | yes | What the capability does / who it's for. Drives matching. |
| system_prompt | string | yes | The capability agent's instructions. |
| ideal_customer | string | no | Target customer description. |
| problem_solved | string | no | |
| value_proposition | string | no | Defaults to `intent_description`. |
| body | string | no | Ad/creative body text. |
| example_queries | string[] | no | Example user asks (boosts matching). |
| negative_contexts | string[] | no | Where NOT to match. |
| negative_situations | string[] | no | |
| targeting_countries | string[] | no | Default `["US"]`. |
| targeting_languages | string[] | no | Default `["en"]`. |
| targeting_platforms | string[] | no | Default `["web","ios","android","desktop"]`. |
| agent_deliverable | string | no | What the agent returns to the caller. |
| required_inputs | Array&lt;{field_name, type?, required?, description?}&gt; | no | Fields the capability must collect. `type` ∈ text/string/file/link/image/pdf (default `text`); `required` default true. Stored as `capability_config.required_fields`. |
| api_endpoints | object[] | no | Backing API endpoints (default `[]`). |
| api_auth | object | no | Auth config for those endpoints (default `{}`). |
| upfront_response | string | no | Stored as `fallback_message`. |
| payments_enabled | boolean | no | Default false. |
| metadata.payment_amount | dollars | no | Only stored (as `deliverable_price_cents`) when `payments_enabled` is true. |
| website | string | no | Landing/booking URL → `ad_units.action_url`. |
| knowledge_sources | Array&lt;{name, type}&gt; | no | Only name/type persisted here; upload content via `upload-knowledge-docs`. |
| advertiser_id | uuid | no | Disambiguation only; ownership-checked against `contact_email`. |

### Response `200`
| Field | Type | Description |
|---|---|---|
| success | boolean | `true`. |
| campaign_id | uuid | New campaign/capability id (= `capability_id` everywhere else). |
| ad_unit_id | uuid | The created ad unit. |
| capability_type | string | Always `"dynamic"`. |
| message | string | Confirmation. |

### Errors
| HTTP | code | When |
|---|---|---|
| 400 | — | invalid JSON / `{error:"Missing required fields", missing:[...]}` / budget or bid range |
| 401 | — | missing/invalid token |
| 403 | — | not owner / not active |
| 404 | — | advertiser not found |
| 500 | — | `{error:"Internal server error", message, details}` |

---

## campaign-update
```yaml
method: POST
path:   /functions/v1/campaign-update
auth:   jwt:uid
proxy:  /api/proxy/campaign-update
```
**Purpose.** Edit, pause, resume, complete, or archive a campaign. Dispatch is by which keys appear in
`updates`. Re-embeds when a semantic field changes; syncs the ad unit.

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| campaign_id | uuid | yes | Target. |
| updates | object | yes | ≥1 recognized key (below). |
| advertiser_id | uuid | no | Disambiguation only; ownership-checked. |

**Recognized `updates` keys** (all optional):
| Key | Type | Description |
|---|---|---|
| status | enum | `active` \| `paused` \| `completed` \| `ended`. |
| name | string | ≤200 chars. |
| budget | dollars | $10–$100k. |
| bid_cpc | dollars | $0.01–$100. |
| payments_enabled | boolean | |
| intent_description | string | Re-embeds; also sets `value_proposition`. |
| problem_solved | string | Re-embeds. |
| body | string | Re-embeds. |
| example_queries | string[] | Re-embeds. |
| trigger_contexts | array | Cap 25. Re-embeds. |
| situational_signals | array | Cap 15 (stored in metadata). |
| ideal_customer | string | No re-embed. |
| negative_contexts | array | Cap 25. No re-embed. |
| website | string | → metadata + `ad_units.action_url`. |
| agent_deliverable | string | Into `capability_config`. |
| system_prompt | string | Into `capability_config`. |
| required_inputs | array | → `capability_config.required_fields` (⚠️ default type here is `string`, not `text`). |
| upfront_response | string | → `fallback_message`. |
| api_endpoints | object[] | Replaced wholesale. |
| api_auth | object | Replaced wholesale. |

```ts
interface CampaignUpdateRequest {
  campaign_id: string;
  updates: Record<string, unknown>; // keys above
  advertiser_id?: string;
}
```

### Response `200`
| Field | Type | Description |
|---|---|---|
| success | boolean | `true`. |
| campaign_id | uuid | Echoed. |
| updated_fields | string[] | Which fields actually changed. |

### Errors
| HTTP | code | When |
|---|---|---|
| 400 | — | bad JSON / missing `campaign_id` / missing-invalid `updates` / invalid status / range / "No recognized fields in updates" |
| 401 / 403 | (auth) | see Strategy A; 403 also = not owner |
| 404 | — | campaign not found |
| 500 | — | internal |

---

## advertiser-campaigns
```yaml
method: POST
path:   /functions/v1/advertiser-campaigns
auth:   jwt:uid
proxy:  /api/proxy/advertiser-campaigns
```
**Purpose.** List the advertiser's campaigns (or one by id).

> ⚠️ **Does NOT return `agent_calls` or `feedback_count`.** To show a per-campaign conversation count or
> feedback count, derive them from `campaign-logs` (see [Deriving counts](#deriving-per-campaign-counts)).

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| campaign_id | uuid | no | Present → single-campaign mode (still returned as a 1-element array). |
| advertiser_id | uuid | no | Disambiguation only. |

### Response `200`
`{ campaigns: Campaign[] }` where each `Campaign` is the **full `campaigns` row** plus a nested
`ad_units: [{ action_url }]`. Key fields:

| Field | Type | Description |
|---|---|---|
| id | uuid | Campaign/capability id (= `capability_id`). |
| advertiser_id | uuid | Owner. |
| name | string | |
| status | string | `active` \| `paused` \| `completed` \| `ended`. |
| budget | dollars | Total budget. |
| budget_spent | dollars | Spent to date. |
| bid_cpc / bid_cpm | dollars \| null | Bids. |
| impressions / clicks | int | Ad-serving counters. |
| payments_enabled | boolean | Paid-capability toggle. |
| deliverable_price_cents | cents \| null | Price when payments enabled. |
| capability_enabled | boolean | Whether the capability is live. |
| capability_config | object | Includes `required_fields`, `system_prompt`, `agent_deliverable`, `fallback_message`, `api_endpoints`, `api_auth`, `knowledge_sources`. |
| intent_description, ideal_customer, problem_solved, value_proposition | string \| null | Matching text. |
| example_queries, negative_contexts, trigger_contexts | array | Matching signals. |
| targeting_countries / _languages / _platforms | string[] | Targeting. |
| quality_score | number | 0–1 quality. |
| metadata | object | Misc (may hold `situational_signals`, `payment_amount`, `website`). |
| created_at / updated_at | timestamptz | |
| ad_units | [{ action_url }] | Landing/booking URL(s). |

> ⚠️ `intent_embedding` is present on the row (a large vector) — ignore it in the UI.

### Errors
| HTTP | code | When |
|---|---|---|
| 405 | — | non-POST |
| 401 / 403 | (auth) | Strategy A |
| 404 | — | single-campaign mode, not found/owned |
| 500 | — | DB error |

---

## advertiser-stats
```yaml
method: GET
path:   /functions/v1/advertiser-stats?advertiser_id&campaign_id&period
auth:   jwt:uid   (or X-Advertiser-Key)
proxy:  /api/proxy/advertiser-stats
```
**Purpose.** Dashboard KPI aggregates over a period. Rate-limited (60/min/IP).

### Request (query params)
| Param | Type | Req | Description |
|---|---|---|---|
| advertiser_id | uuid | no | Disambiguation only; ownership-checked. |
| campaign_id | uuid | no | Filter to one campaign. |
| period | enum | no | `today` \| `7d` \| `30d` \| `90d` \| `all` (default `30d`). |

Auth: `Authorization: Bearer <jwt>` **or** `X-Advertiser-Key: <api_key>`.

### Response `200`
| Field | Type | Description |
|---|---|---|
| advertiser_id | uuid | |
| company_name | string | |
| wallet_balance | dollars | Live balance. |
| wallet_transactions | array | Last 50: `{ id, amount_cents, type, description, created_at }`. |
| period | object | `{ start, end, label }`. |
| budget | object | `{ total, spent, remaining, currency:"USD" }` — ⚠️ `spent`/`remaining` use *estimated* spend from events, not `budget_spent`. |
| metrics | object | `{ impressions, clicks, conversions, ctr, cvr, cost_per_click, cost_per_conversion }` (last four are 2-dp strings). |
| spend_breakdown | object | `{ cpm_spend, cpc_spend, total }` (2-dp strings). |
| campaigns | array | Per-campaign: `{ campaign_id, name, status, budget, spent, bid_cpc, bid_cpm, impressions, clicks, ctr, created_at }`. |

> ⚠️ These `campaigns[]` objects are the **stats** projection — they do **not** carry a conversation
> count either.

### Errors
| HTTP | code | When |
|---|---|---|
| 429 | — | rate limit |
| 401 / 403 | auth_error | no key/JWT, or not owned |
| 500 | internal_error | — |

---

## agent-test
```yaml
method: POST
path:   /functions/v1/agent-test
auth:   jwt:email
proxy:  /api/proxy/agent-test
```
**Purpose.** Let a business test-drive its own capability agent (a real session against their capability).
`body.action` dispatch: `list | start | message | end`.

### Requests
| action | Fields | Description |
|---|---|---|
| list | `{ action:"list" }` | List the advertiser's testable capabilities. |
| start | `{ action:"start", campaign_id, initial_data?, initial_message? }` | Begin a session. |
| message | `{ action:"message", session_id, message }` | Send a turn. |
| end | `{ action:"end", session_id }` | Close + summarize. |

### Responses `200`
- **list:** `{ items: [{ campaign_id, ad_unit_id, name, status, system_prompt, required_fields, has_business_docs, has_api_endpoints, budget, budget_spent, quality_score, quality_tier, created_at }], total }`
- **start:** `{ session_id, status:"active", message, pending_fields:string[], structured_data, expires_at, tokens_used, cost_usd }`
- **message:** `{ session_id, message, structured_data, pending_fields, status:"active", tokens_used, cost_usd, total_tokens, total_cost_usd }`
- **end:** `{ session_id, status:"completed", summary, conversation_history, message_count, total_tokens, total_cost_usd }`

### Errors
`401` (auth), `403` (no advertiser / inactive), `405` (non-POST), `400 invalid_action` / `missing_params`,
`404 not_found`, `400 no_ad_unit`, `410 expired`, `500 query_failed`/`server_error`/`agent_error`.

---

## upload-knowledge-docs
```yaml
method: POST  (multipart/form-data)
path:   /functions/v1/upload-knowledge-docs
auth:   jwt:email
proxy:  (direct; multipart)
```
**Purpose.** Attach a knowledge doc (RAG source) or a paid deliverable file to a capability.

### Request (form fields)
| Field | Type | Req | Description |
|---|---|---|---|
| campaign_id | uuid | yes | Target capability. |
| file | binary | yes | ≤25 MB (knowledge) / ≤100 MB (deliverable). |
| kind | enum | no | `knowledge` (default) or `deliverable`. |
| license | string | no | Deliverable only. |

### Response `200`
- **knowledge:** `{ success:true, storage_path, file_name, file_size, processing_status:"queued", message }` — stored in `knowledge-docs` bucket; queued for embedding.
- **deliverable:** `{ success:true, kind:"deliverable", storage_path, filename, byte_size, checksum_sha256, message }` — stored in private `deliverables` bucket; recorded in `capability_deliverables`; not embedded.

### Errors
`401` (auth), `405` (non-POST), `400` (missing campaign_id/file, size exceeded), `404` (campaign not
found), `403` (not owner), `500` (upload/record failure with `details`).

---

# 3. Billing, wallet & earnings

## advertiser-billing
```yaml
method: POST
path:   /functions/v1/advertiser-billing
auth:   jwt:uid
proxy:  /api/proxy/advertiser-billing
```
**Purpose.** One call powering the billing page: wallet balance, wallet history, seller earnings, and
per-campaign earnings.

### Request
`{ advertiser_id?: string }` (optional; disambiguation only). Empty body is fine.

### Response `200`
| Field | Type | Description |
|---|---|---|
| wallet_balance | dollars | Prepaid ad-spend balance. |
| earnings_balance | cents | Withdrawable seller earnings. |
| lifetime_earnings | cents | Sum of `campaign_earnings[].net_cents` (never decreases on withdrawal). |
| campaign_earnings | array | Per-campaign earnings, sorted by `net_cents` desc. |
| ↳ campaign_id | uuid | |
| ↳ campaign_name | string | Display name (may be a uuid slice if unnamed). |
| ↳ sale_count | int | Number of paid sales. |
| ↳ net_cents | cents | Earnings after platform fee. |
| ↳ gross_cents | cents | Before fee. |
| ↳ fee_cents | cents | Platform fee. |
| ↳ last_sale_at | timestamptz | Most recent sale. |
| wallet_transactions | array | Last 50, newest first — the wallet history. |
| ↳ id | uuid | |
| ↳ amount | number | Transaction amount (sign indicates direction). |
| ↳ type | string | e.g. `topup`, `charge`, `withdrawal`. |
| ↳ status | string | e.g. `completed`. |
| ↳ description | string \| null | |
| ↳ created_at | timestamptz | |
| ↳ metadata | object | Extra context. |

```ts
interface BillingResponse {
  wallet_balance: number;    // dollars
  earnings_balance: number;  // cents
  lifetime_earnings: number; // cents
  campaign_earnings: Array<{
    campaign_id: string; campaign_name: string; sale_count: number;
    net_cents: number; gross_cents: number; fee_cents: number; last_sale_at: string;
  }>;
  wallet_transactions: Array<{
    id: string; amount: number; type: string; status: string;
    description: string | null; created_at: string; metadata: Record<string, unknown>;
  }>;
}
```

### Errors
`405` (non-POST), `401/403/404` (auth), `500` (query failure).

---

## add-funds
```yaml
method: POST
path:   /functions/v1/add-funds
auth:   jwt:uid
proxy:  /api/proxy/add-funds
```
**Purpose.** Top up the ad-spend wallet via Stripe Checkout.

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| amount_cents | cents | yes | Integer, **≥ 5000 ($50 minimum)**. |
| advertiser_id | uuid | no | Disambiguation only. |

### Response `200`
`{ checkout_url: string }` — redirect the browser here. On completion Stripe redirects to
`/advertiser/billing?status=success|cancelled`; the wallet is credited by the webhook.

### Errors
`400 Invalid JSON body` / `400 amount_cents must be an integer >= 5000`, `401/403/404` (auth),
`405` (non-POST), `500 Failed to create checkout session`.

---

## request-withdrawal
```yaml
method: POST
path:   /functions/v1/request-withdrawal
auth:   jwt:email
proxy:  /api/proxy/request-withdrawal
```
**Purpose.** Withdraw seller earnings to the connected Stripe account (hold-then-transfer).

### Request
| Field | Type | Req | Description |
|---|---|---|---|
| amount_cents | cents | yes | Integer, **≥ 100 ($1)**, ≤ `earnings_balance`. |
| client_request_id | uuid | yes | Idempotency token — reuse to safely retry. |

### Response `200`
- **success:** `{ success:true, withdrawal_id, transfer_id, new_earnings_balance }` (⚠️ `new_earnings_balance` is in **dollars**).
- **idempotent replay:** `{ success, idempotent_replay:true, withdrawal_id, status, transfer_id }`.

### Errors
| HTTP | code | When |
|---|---|---|
| 401 | — | missing/invalid token |
| 404 | — | advertiser not found for user |
| 400 | — | bad JSON / amount validation / missing `client_request_id` |
| 400 | stripe_not_connected | connect Stripe first |
| 502 | stripe_account_unavailable | Stripe account lookup failed |
| 409 | account_not_ready | account can't receive payouts yet (reason in `message`) |
| 409 | insufficient_earnings | amount exceeds withdrawable balance |
| 409 | funds_settling | card funds still clearing (~2 business days) |
| 502 | transfer_failed | payout failed; no funds moved |
| 500 | — | failed to record withdrawal |

---

## stripe-connect-init
```yaml
method: POST
path:   /functions/v1/stripe-connect-init
auth:   jwt:email
proxy:  (direct)
```
**Purpose.** Start Stripe Connect onboarding (to receive payouts). Returns an authorize URL to redirect to.

### Request
No body. The function reads the `Origin` header to build the callback return URL.

### Response `200`
| Field | Type | Description |
|---|---|---|
| url | string | `connect.stripe.com/oauth/authorize?...` — redirect the browser here. |
| already_connected | boolean | True if status is already `connected`. |
| stripe_account_id | string \| null | Existing connected account, if any. |

### Errors
`500` (missing `STRIPE_CONNECT_CLIENT_ID` / `TRACKING_HMAC_SECRET`), `401` (auth), `404` (advertiser not
found), `405 method_not_allowed`.

---

## stripe-connect-callback
```yaml
method: GET
path:   /functions/v1/stripe-connect-callback?code&state
auth:   public (HMAC-signed state)
proxy:  (browser redirect target from Stripe)
```
**Purpose.** OAuth return from Stripe. Not called by the app directly — Stripe redirects the browser here.

### Request (query params)
`code` (required), `state` (required, HMAC-signed), optional `error`, `error_description`.

### Behavior
On success: exchanges the code, persists `stripe_account_id`, sets `stripe_connect_status='connected'`
and `stripe_connected_at`, then **302-redirects** to `<returnUrl>/advertiser/settings?stripe=connected`.
On failure: redirects with `?stripe=error` (or renders an HTML error page).

### Errors
`400` (Stripe error / missing code|state / invalid state), `500` (secret missing / DB update failed),
`502` (token exchange failed).

---

## stripe-connect-disconnect
```yaml
method: POST
path:   /functions/v1/stripe-connect-disconnect
auth:   jwt:email
proxy:  (direct)
```
**Purpose.** Unlink the connected Stripe account.

### Request
No body.

### Response `200`
| Field | Type | Description |
|---|---|---|
| disconnected | boolean | `true`. |
| previous_stripe_account_id | string \| null | The account that was unlinked. |
| already_disconnected | boolean | Present/true if nothing was connected. |
| campaigns_disabled | int | How many of this advertiser's campaigns had `payments_enabled` flipped off as a side effect. |

### Errors
`401` (auth), `404` (advertiser not found), `405 method_not_allowed`,
`500 Failed to clear Stripe link` (with `details`).

---

# 4. Conversations, feedback & learned insights

## campaign-logs
```yaml
method: POST
path:   /functions/v1/campaign-logs
auth:   jwt:email
proxy:  (called directly)
```
**Purpose.** Everything about the conversations agents had with this business's capabilities: the session
index, full transcripts, the feedback agents left, and the insights the nightly roller distilled.
`body.action` dispatch. All list actions share the pagination envelope.

**Pagination envelope** (all `list_*`):
```ts
interface Paginated<T> { items: T[]; total: number; limit: number; offset: number; has_more: boolean }
```
`limit` clamps to **≤200** (default 50); `offset` default 0. Ownership is enforced: a `list_*` for a
campaign you don't own returns an empty list; `get_*` returns 403/404.

### action: `list_sessions`
Request: `{ action:"list_sessions", campaign_id?, limit?, offset?, start_date?, end_date? }`
(omit `campaign_id` for all campaigns; date filter is on `last_updated_at`, sorted desc).

| Field | Type | Description |
|---|---|---|
| session_id | uuid | Use for `get_session`; correlates with `feedback.session_id`. |
| capability_id | uuid | Which campaign the session was with. |
| ad_unit_id | uuid | Ad variant clicked. |
| agent_id | string | The consuming agent's id. |
| message_count | int | Total turns. |
| total_tokens | int | Tokens consumed. |
| total_cost_usd | number | Session cost. |
| status | string | `active` \| `completed` \| `expired` \| `cancelled`. |
| started_at / last_updated_at / ended_at | timestamptz | `ended_at` null while active. |
| auction_query | string \| null | The query that surfaced the capability (if via auction). |
| auction_result_count | int | |
| session_origin | string | `auction` \| `direct`. |

### action: `get_session`
Request: `{ action:"get_session", session_id }`. Response: `{ item: SessionDetail }` = everything from
`list_sessions` **plus**:

| Field | Type | Description |
|---|---|---|
| advertiser_id | uuid | Owner (ownership-checked). |
| conversation_history | Array&lt;{role, content}&gt; | ⚠️ **The full transcript.** `role` ∈ `user`/`assistant`; ordered. |
| structured_data | object | Fields the agent extracted during the session (open-ended). |
| task_context | string \| null | The initiating task/context. |

Errors: `400 missing_params`, `403 forbidden` (other advertiser), `404 not_found`.

### action: `list_feedback`
Request: `{ action:"list_feedback", campaign_id?, limit?, offset?, start_date?, end_date? }`
(date filter on `created_at`).

| Field | Type | Description |
|---|---|---|
| id | uuid | Feedback row id. |
| session_id | uuid | The session this feedback is about. |
| capability_id | uuid | The campaign. |
| agent_id | string | The consuming agent that left it. |
| feedback_text | string | ⚠️ Holds the **business-readable summary** (the raw dense note is replaced by a summary in this field). |
| score | int \| null | 0–100 satisfaction; null if not given. |
| bonus_amount_cents | cents \| null | Bonus paid for the feedback (paid-feedback flow), else 0. |
| resolved_at | timestamptz \| null | When the bonus was resolved. |
| created_at | timestamptz | |

### action: `get_tool_context`
Request: `{ action:"get_tool_context", campaign_id }` **(campaign_id required)**. The learned-insights
profile the nightly roller maintains for one capability.

| Field | Type | Description |
|---|---|---|
| capability_id | uuid | The capability. |
| learnings | array | The distilled insights, most-evidenced first. |
| ↳ key | string | snake_case handle. |
| ↳ learning | string | ⚠️ The **headline insight text** (human-readable; despite the terse field name). |
| ↳ description | string | Longer explanation / detail. |
| ↳ surfaced_count | int | How many sessions evidenced this. |
| ↳ status | string | `active` (others are filtered out server-side). |
| ↳ evidence_session_ids | uuid[] | Sessions that evidenced it — resolve each via `get_session`. |
| ↳ updated_at | timestamptz | Last time the roller touched it. |
| learning_count | int | `learnings.length`. |
| last_rolled_at | timestamptz \| null | Last roll for this capability. |

> ⚠️ **Shape changed in the pivot.** Older docs described `known_issues` / `knowledge_gaps` /
> `learned_facts`. The live endpoint returns the flat `learnings[]` above. Build against `learnings[]`.

Errors: `400 missing_param` (no `campaign_id`), `404 not_found` (not owned).

### actions: `list_searches` / `get_search`
`list_searches` `{ action:"list_searches", campaign_id, limit? }` → search-log index (each:
`id, decision_id, agent_id, source, query, user_context, result_count, top_result,
matched_campaign_ids, created_at`). `get_search` `{ action:"get_search", id }` → `{ item }` (full row;
403 if none of `matched_campaign_ids` is owned).

---

# Appendix

## Deriving per-campaign counts
`advertiser-campaigns` and `advertiser-stats` do **not** return a conversation count or feedback count.
To show "total conversations" or "feedback count" per campaign, call `campaign-logs` and group client-side:

```ts
// conversations per campaign
const { items } = await campaignLogs("list_sessions", { limit: 1000 });
const convosByCampaign = new Map<string, number>();
for (const s of items) convosByCampaign.set(s.capability_id, (convosByCampaign.get(s.capability_id) ?? 0) + 1);

// feedback per campaign
const fb = await campaignLogs("list_feedback", { limit: 1000 });
const feedbackByCampaign = new Map<string, number>();
for (const f of fb.items) feedbackByCampaign.set(f.capability_id, (feedbackByCampaign.get(f.capability_id) ?? 0) + 1);
```
> `list_sessions` `total` is the count across the selected scope; for per-campaign counts, group the
> `items` by `capability_id` (or call once per `campaign_id`). Remember `limit` clamps to 200 — page with
> `offset`/`has_more` if a campaign can exceed 200 sessions.

## Entity relationships
- `campaigns.id` == `capability_id` everywhere (campaign and capability are the same object).
- `feedback.session_id` → a `list_sessions`/`get_session` session (a session may have 0 or 1 feedback).
- `learnings[].evidence_session_ids` → the sessions that produced a learning (resolve via `get_session`).

## Units cheat-sheet
| Cents (÷100 to display) | Dollars |
|---|---|
| earnings_balance, lifetime_earnings, net_cents, gross_cents, fee_cents, bonus_amount_cents, amount_cents, deliverable_price_cents | wallet_balance, budget, budget_spent, total_cost_usd, new_earnings_balance |

## Fields whose name lies (pivot legacy)
| Field | Actually is |
|---|---|
| `feedback_text` (list_feedback) | the business-readable **summary**, not the raw note |
| `learning` (get_tool_context) | a human-readable headline insight |
| `capability_type` (campaign-create) | ignored — always `dynamic` |
| `agent_calls` (expected on campaigns) | **does not exist** — derive from `list_sessions` |
| `budget.spent` (advertiser-stats) | *estimated* spend from events, not `budget_spent` |

---

_Generated from live edge-function source. When an endpoint changes, update this file — it is the
contract both humans and agents build against._
