# JeelPay Refunds

JeelPay supports refunding installment plans through `POST /v1/refund`. Refund approval is not
guaranteed: JeelPay evaluates each request based on the payment method, timing of the original
transaction, payment provider behavior, and bank processing rules.

Some refunds complete immediately. Others remain `PENDING` while JeelPay reviews or processes them.
Integrations must treat refund completion as asynchronous unless the API immediately returns a terminal
status.

## Submit a Refund

```http
POST /v1/refund
Authorization: Bearer {access_token}
Content-Type: application/json
```

Request body:

```json
{
  "installmentRequestId": "9e79d502-231d-449b-b419-a674b687df51",
  "amount": 1250.00,
  "reason": "Student withdrew from the course",
  "referenceId": "refund-order-1234"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `installmentRequestId` | string (UUID) | Yes | Checkout or installment request ID to refund |
| `amount` | number | Yes | Refund amount in SAR with 2 decimal places |
| `reason` | string | Yes | Reason for the refund request |
| `referenceId` | string | No | Your own refund correlation reference; echoed back in refund webhooks when provided |

Omit `referenceId` completely when you do not have a real correlation value. Do not send it as `null`
or an empty string.

The `amount` controls refund type:

- **Full refund:** use the full refundable amount for the installment request.
- **Partial refund:** use a smaller valid amount.

### Response

```json
{
  "withdrawalRequestId": "793ffb54-f5e7-47f5-b161-425ec775f14a",
  "status": "PENDING",
  "referenceId": "refund-order-1234"
}
```

The `status` can be `DONE`, `PENDING`, or `REJECTED`. Store `withdrawalRequestId` so your system can
correlate the refund request and support follow-up investigations.

`POST /v1/refund` returns HTTP `400` for missing or invalid fields, an invalid refund amount, or an
amount that exceeds the paid amount. Store the response `tx_id` header for both successful and failed
requests.

## Get Refund Status

```http
GET /v1/refund/{id}
Authorization: Bearer {access_token}
```

`{id}` can be either an installment request ID or an integration checkout ID.

### Response

```json
{
  "status": "PENDING",
  "rejectionReason": null,
  "referenceId": "refund-order-1234"
}
```

The status endpoint returns `400` for an invalid request and `404` when the refund request cannot be
found. On `REJECTED`, persist `rejectionReason` when present.

## Refund Statuses

| Status | Meaning |
|--------|---------|
| `DONE` | Refund was successfully processed and completed |
| `PENDING` | Refund is under review or still being processed |
| `REJECTED` | Refund was rejected; check `rejectionReason` where available |

The initial `POST` response can return `DONE` or `REJECTED` immediately, but most integrations should
expect `PENDING` and wait for a webhook or poll as fallback.

## Refund Webhooks

When the refund reaches a final state, JeelPay sends a webhook to the same `notification_url` configured
for the original schooling or items checkout.

Example payload:

```json
{
  "id": "793ffb54-f5e7-47f5-b161-425ec775f14a",
  "status": "DONE",
  "rejectionReason": null,
  "referenceId": "refund-793ffb54-f5e7-47f5-b161-425ec775f14a"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Refund or installment request identifier from JeelPay |
| `status` | string | `DONE`, `PENDING`, or `REJECTED` |
| `rejectionReason` | string or null | Reason for rejection, if rejected |
| `referenceId` | string or null | Reference ID provided during refund creation, when available |

Refund webhooks use the same `X-Jeel-Signature` header and HMAC-SHA256 + base64 verification process
as checkout webhooks. Always verify the raw request body before parsing it.

When writing webhook handlers, distinguish checkout status values (`PENDING`, `SUCCEEDED`, `REJECTED`,
`EXPIRED`) from refund status values (`DONE`, `PENDING`, `REJECTED`). The shared `PENDING` and `REJECTED`
names mean different business events depending on the payload shape.

## Integration Requirements

- Always handle `PENDING`; do not mark a refund complete until status is `DONE`.
- Treat `REJECTED` as terminal and store/display `rejectionReason` when provided.
- Poll `GET /v1/refund/{id}` periodically only as a fallback if the webhook is delayed or missed.
- Inform end users that refunds may be instant or may take weeks depending on card type, provider, bank,
  and transaction timing.
- Avoid promising a fixed completion deadline in UI copy.
- Verify every refund webhook signature with the same raw-body HMAC process used for checkout webhooks.
