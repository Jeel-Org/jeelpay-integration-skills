# JeelPay Refunds

JeelPay supports refund requests for completed checkouts. A refund request is submitted to JeelPay
and goes through a review process.

## Submit a Refund

```http
POST /v1/refund
Authorization: Bearer {access_token}
Content-Type: application/json
```

You can reference either a `checkout_id` or an `installmentRequestId` (legacy):

```json
{
  "checkout_id": "9e79d502-231d-449b-b419-a674b687df51"
}
```

### Response

```json
{
  "id": "refund-uuid",
  "status": "PENDING"
}
```

## Get Refund Status

```http
GET /v1/refund/{refund_id}
Authorization: Bearer {access_token}
```

### Response

```json
{
  "id": "refund-uuid",
  "status": "APPROVED"
}
```

## Notes

- Refunds are subject to JeelPay's refund policy and review process
- Only `SUCCEEDED` checkouts can be refunded
- Store the `refund_id` from the submit response to poll status later
