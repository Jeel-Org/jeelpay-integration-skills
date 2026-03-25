---
name: jeelpay-integration-skills
description: >
  Helps developers integrate with JeelPay's "Study Now Pay Later" API for educational payments.
  Use this skill whenever the user mentions JeelPay, Jeel Pay, jeel.co, integrating SNPL for schools
  or universities, or accepting installment payments for educational fees in Saudi Arabia.
  Also trigger when the user is working with JeelPay checkout endpoints, webhook handlers,
  or building payment integration for schools, universities, courses, or educational institutions
  in Saudi Arabia, even if they don't explicitly say "JeelPay".
---

# JeelPay Integration Skill

JeelPay (jeel.co) is a "Study Now Pay Later" fintech platform in Saudi Arabia. Educational institutions
register with JeelPay, and through the API they can create checkout requests on behalf of their students
or buyers — the buyer then completes registration and payment on JeelPay's platform, and JeelPay notifies
the institution of the result via webhook and redirect.

## Checkout Types — Ask First

JeelPay has **two checkout types**, and a merchant account is configured for one or the other. Using the
wrong one will result in API errors. **Always ask the developer which type their account uses before
generating any code.**

| Type | Who uses it |
|------|-------------|
| **Schooling** | Traditional schools (K-12), student-based tuition fees |
| **Items** | Universities, institutes, higher education, courses, training centers |

> "Does your JeelPay account use **Schooling** checkout or **Items** checkout?"

If the developer is unsure, direct them to contact JeelPay support or check their entity dashboard at
`entity.sandbox.jeel.co` (sandbox) or their production dashboard.

## Integration Workflow

When a developer asks for help integrating JeelPay, follow this sequence:

1. **Confirm checkout type** — ask if not already known (see above)
2. **Detect their tech stack** — look for `package.json`, `pom.xml`, `composer.json`, `requirements.txt`,
   `go.mod`, `Gemfile`, `*.csproj`, etc. If the stack isn't clear, ask.
3. **Understand scope** — are they doing a full integration from scratch, or need help with a specific
   piece (auth, checkout creation, webhook handling, status polling)?
4. **Read the relevant reference files** (listed below) and generate tailored code

## What to Generate

For a **full integration**, deliver these components in order:

### 1. Auth Client with Token Caching
Read `references/api-auth.md` before generating.

Token caching is non-negotiable — the auth endpoint is rate-limited. Every generated auth client must:
- Check for a valid cached token before making any auth request
- Store the token with its expiration timestamp
- Refresh ~30 seconds before expiry to avoid edge cases
- Never request a new token for each API call

### 2. Checkout Creation
Read `references/api-checkout-items.md` or `references/api-checkout-schooling.md` depending on type.

The checkout flow is:
1. Merchant server calls JeelPay API to create a checkout → receives `checkout_id` + `redirect_url`
2. Merchant redirects buyer to `redirect_url` (JeelPay's platform for login/KYC/payment)
3. JeelPay redirects buyer back to merchant's `redirect_url` with result
4. JeelPay also sends a webhook to merchant's `notification_url`

### 3. Webhook Handler with Signature Verification
Read `references/api-webhooks.md` before generating.

Every webhook handler must verify the `X-Jeel-Signature` header using HMAC-SHA256 + base64.
Never skip signature verification — it's the only way to confirm the webhook is from JeelPay.

### 4. Checkout Status Polling (optional)
Use `GET /v3/checkout/{id}` if the developer needs to poll status independently of webhooks.
The status field is one of: `PENDING`, `SUCCEEDED`, `REJECTED`, `EXPIRED`.

### 5. Environment Configuration
Always structure code to support both environments via env vars:

```
JEELPAY_CLIENT_ID=...
JEELPAY_CLIENT_SECRET=...
JEELPAY_ENV=sandbox  # or: production
```

Map `JEELPAY_ENV` to the appropriate base URLs:
- Sandbox API: `https://api.sandbox.jeel.co`
- Sandbox Auth: `https://auth.sandbox.jeel.co`
- Production API: `https://api.jeel.co`
- Production Auth: `https://auth.jeel.co`

## Critical Rules

**Token caching is mandatory.** The auth endpoint is rate-limited. If you generate code that requests
a new token on every API call, the integration will break in production. Always cache with expiry.

**Always verify webhook signatures.** Generate the HMAC-SHA256 + base64 check on every incoming webhook.
Silently accepting webhooks without verification is a security vulnerability.

**Discourage the deprecated installment requests API.** If a developer asks about "installment requests"
endpoints (a different, older API), explain it's being deprecated and guide them to use the checkout API
instead (`POST /v3/checkout` or `POST /v3/checkout/schooling`).

**Use environment variables for credentials.** Never hardcode `client_id`, `client_secret`, or base URLs.
Always generate code that reads these from environment variables or a config file.

**Include Saudi-specific input validation** when generating forms or request builders:
- Saudi mobile: `^5[0-9]{8}$` (9 digits, starts with 5)
- National ID: `^[12][0-9]{9}$` (10 digits, starts with 1 or 2)

**Handle all error cases.** JeelPay returns structured errors. Always include handlers for:
- `400` — Validation error or type mismatch (wrong checkout type for the account)
- `401` — Invalid or expired token (re-authenticate and retry)
- `404` — Resource not found
- `500` — JeelPay server error (log and surface to user)

**Remind about checkout expiration.** Checkouts expire after 2 hours of inactivity (status becomes
`EXPIRED`). If your app needs to handle retries, generate a new checkout rather than reusing an expired one.

**Currency is always SAR.** JeelPay operates exclusively in Saudi Riyals (SAR). All `cost` and `total_cost`
amounts are in SAR with exactly 2 decimal places (e.g., `3500.00`). There is no smallest-unit conversion
(no paise, no halala, no cents) — just SAR with 2 decimal places.

**Installment split rule** (useful for UX messaging):
- Total ≤ 5,000 SAR → 4 equal payments
- Total > 5,000 SAR → payment count per JeelPay agreement with the entity

## Reference Files

Load the appropriate file(s) before generating code:

| Need | File |
|------|------|
| Authentication + token caching | `references/api-auth.md` |
| Items checkout (universities, courses, institutes) | `references/api-checkout-items.md` |
| Schooling checkout (traditional schools) | `references/api-checkout-schooling.md` |
| Webhook handling + signature verification | `references/api-webhooks.md` |
| Refund submission and status | `references/api-refund.md` |
| Sandbox URLs, test card, test credentials | `references/testing.md` |
