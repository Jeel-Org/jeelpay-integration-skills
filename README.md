# JeelPay Integration Skill for AI Coding Agents

A skill that makes AI agents experts at integrating with [JeelPay's](https://jeel.co) "Study Now Pay Later" API for educational payments.

## What It Does

When you ask any coding agent for help integrating JeelPay, this skill guides them to:

- Ask the right questions upfront (checkout type: Items vs Schooling)
- Detect your tech stack automatically
- Generate complete, production-ready integration code:
  - **Auth client** with token caching (required by JeelPay's rate-limited auth endpoint)
  - **Checkout creation** tailored to your checkout type
  - **Webhook handler** with HMAC-SHA256 signature verification
  - **Environment-based config** for easy sandbox ↔ production switching

Works with any language or framework — Node.js, Python, PHP, Java, Go, and more.

## Install

```bash
npx skills add https://github.com/Jeel-Org/jeelpay-integration-skills
```

## Usage

Just describe what you're building:

> "Help me integrate JeelPay into my Express.js app"

> "I need a webhook handler for JeelPay in Laravel"

> "Add JeelPay schooling checkout to our Django school management system"

agent will ask which checkout type your account uses (Items for universities/institutes, Schooling for traditional schools), then generate complete integration code.

## JeelPay Checkout Types

| Type | Who uses it |
|------|-------------|
| **Schooling** (`POST /v3/checkout/schooling`) | Traditional schools, student tuition fees |
| **Items** (`POST /v3/checkout`) | Universities, institutes, courses, training centers |

Your JeelPay account is configured for one type. Using the wrong one returns a 400 error.

## What Gets Generated

A full integration includes:

1. **Auth client** — OAuth 2.1 client credentials with token caching (mandatory, auth is rate-limited)
2. **Checkout creation** — Create a checkout, get a `redirect_url`, send buyer there
3. **Webhook handler** — Receive status updates (`SUCCEEDED`, `REJECTED`, `EXPIRED`) with signature verification
4. **Status polling** — `GET /v3/checkout/{id}` for on-demand status checks
5. **Environment config** — Env vars for credentials + sandbox/production URL switching

## About JeelPay

[JeelPay](https://jeel.co) is a Saudi fintech platform supervised by the Saudi Central Bank (SAMA) that enables educational institutions to offer "Study Now Pay Later" installment plans to students and parents. Schools and universities register with JeelPay and integrate via API to offer flexible tuition payment options.

## License

MIT
