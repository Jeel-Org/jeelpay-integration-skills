# JeelPay Testing & Sandbox

JeelPay provides a full sandbox environment where you can test the complete integration without
processing real transactions or affecting live data.

## Sandbox URLs

| Service | Sandbox URL | Production URL |
|---------|-------------|----------------|
| API | `https://api.sandbox.jeel.co` | `https://api.jeel.co` |
| Auth | `https://auth.sandbox.jeel.co` | `https://auth.jeel.co` |
| Entity Dashboard | `https://entity.sandbox.jeel.co` | (production dashboard) |

## Getting Test Credentials

Contact JeelPay to receive:
- Test Client ID
- Test Client Secret
- Test Entity ID (and Test Educational Year ID for schooling integrations)

Request credentials at: **jeel.co/en/join-us**

## Test Card

Use this card when the buyer reaches the payment page in the sandbox:

| Field | Value |
|-------|-------|
| Card Type | Visa |
| Card Number | `4111 1111 1111 1111` |
| Expiry | `05/30` |
| CVV | `123` |

When redirected to the 3DS confirmation page, enter: **`Checkout1!`**

## Sandbox User Credentials

When testing the buyer flow (login/registration on JeelPay's checkout page):

| Field | Value |
|-------|-------|
| Phone number | Any valid Saudi mobile (e.g., `512345678`) |
| OTP code | `3333` |
| Passcode | `100000` |

## Switching to Production

Replace your sandbox credentials and URLs with production ones. No code changes needed if you're
using environment variables for the config:

```env
# Sandbox
JEELPAY_CLIENT_ID=sandbox_client_id
JEELPAY_CLIENT_SECRET=sandbox_client_secret
JEELPAY_AUTH_URL=https://auth.sandbox.jeel.co/oauth2/token
JEELPAY_API_URL=https://api.sandbox.jeel.co

# Production (just update these values)
JEELPAY_CLIENT_ID=prod_client_id
JEELPAY_CLIENT_SECRET=prod_client_secret
JEELPAY_AUTH_URL=https://auth.jeel.co/oauth2/token
JEELPAY_API_URL=https://api.jeel.co
```

## Useful Public Endpoints (No Auth Required)

```http
GET https://api.sandbox.jeel.co/v1/public/educational-years
```

Returns the list of active educational years. Use this to find the correct `educational_year_id`
for schooling checkout requests. Same path works on production with the production API URL.

## What You Can Test

With sandbox credentials you can:
- Create checkout requests (both Items and Schooling types)
- Simulate the full buyer flow (login → KYC → payment)
- Receive webhook notifications for all status changes
- Test redirect flows back to your platform
- Verify webhook signature validation
- Test error handling (try invalid inputs, wrong checkout types, etc.)

## Installment Plan Details

JeelPay splits payments as follows:
- Total fee ≤ **5,000 SAR** → divided into **4 equal payments**
- Total fee > **5,000 SAR** → number of payments determined by agreement with the entity

The buyer pays a down payment at checkout time; the remaining installments are collected by JeelPay
automatically per the agreed schedule.

## Common Integration Issues

**"Group type mismatch" error (400)**
Your account is configured for one checkout type but you're calling the other endpoint.
Check which endpoint matches your account: `POST /v3/checkout` (Items) vs `POST /v3/checkout/schooling` (Schooling).

**"Missing entity_id" error (400)**
For Items checkout, `entity_id` is required on each item if your group has more than one entity.
For Schooling checkout, `entity_id` is always required on each student.

**Webhook signature mismatch**
Make sure you're using the raw request body (not re-serialized JSON) and the correct `client_secret`.
The sandbox and production use different secrets.

**Token expiry in testing**
Tokens expire in ~33 minutes. If you're testing manually and see 401 errors, your cached token expired — just re-authenticate.
