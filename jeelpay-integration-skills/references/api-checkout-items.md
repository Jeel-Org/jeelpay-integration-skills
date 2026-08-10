# JeelPay Items Checkout

Use this checkout type for **universities, institutes, higher education providers, courses, and training centers** — educational entities that don't follow traditional K-12 school settings.

If your entity is a traditional school, use `api-checkout-schooling.md` instead.

## Checkout Flow

1. Generate a UUID for the idempotency key and save to your database (status: PENDING)
2. Your server creates a checkout via `POST /v3/checkout` with `Idempotency-Key` header → receive `checkout_id` + `redirect_url`
3. Extract the `tx_id` from response headers and store it for debugging/support
4. If timeout occurs, retry with the same idempotency key
5. Redirect the buyer to `redirect_url` (JeelPay's hosted checkout page)
6. Buyer logs in / registers on JeelPay, reviews items, completes KYC and down payment
7. JeelPay redirects buyer back to your `redirect_url`
8. JeelPay sends a webhook to your `notification_url` confirming the result
9. You can also poll `GET /v3/checkout/{id}` at any time for the current status

## Create Checkout

```http
POST /v3/checkout
Authorization: Bearer {access_token}
Content-Type: application/json
Idempotency-Key: {uuid}
```

> **Idempotency:** Always generate a UUID first, save it to your database, then include it as the `Idempotency-Key` header. If a network timeout occurs, retry with the same key to prevent duplicate checkouts.

### Request Schema

```json
{
  "reference_id": "order_1234",
  "buyer": {
    "first_name": "Essa",
    "last_name": "Alshammari",
    "mobile_number": "512345678",
    "email": "essa@example.com",
    "national_id": "1234567890"
  },
  "items": [
    {
      "item_name": "Data Science Diploma",
      "quantity": 1,
      "unit_price": 3500.00,
      "total_cost": 3500.00,
      "reference_id": "course_456",
      "entity_id": "uuid-of-entity"
    }
  ],
  "urls": {
    "redirect_url": "https://yourapp.com/checkout/result",
    "notification_url": "https://yourapp.com/webhooks/jeelpay"
  },
  "metadata": {
    "student_id": "STU-001",
    "semester": "Fall 2025"
  }
}
```

### Field Reference

**Top level:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `reference_id` | string | No | Your internal order/reference ID (max 255 chars) |
| `buyer` | object | Yes | See buyer fields below |
| `items` | array | Yes | At least one item |
| `urls` | object | Yes | Redirect and notification URLs |
| `metadata` | object | No | Any key-value pairs you want echoed back |

**Buyer:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `first_name` | string | No | Up to 255 chars; omit when unavailable |
| `last_name` | string | No | Up to 255 chars; omit when unavailable |
| `mobile_number` | string | Yes | Saudi mobile: `^5[0-9]{8}$` (9 digits, starts with 5) |
| `email` | string | No | Valid email, up to 255 chars; omit when unavailable |
| `national_id` | string | No | 10 digits: `^[12][0-9]{9}$`; omit when unavailable |

**Item:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `item_name` | string | Yes | 1–50 chars |
| `quantity` | integer | Yes | Min 1 |
| `unit_price` | number | No | Per-unit price, 2 decimal places |
| `total_cost` | number | Yes | `unit_price × quantity`, 2 decimal places |
| `reference_id` | string | Yes | Your internal item ID |
| `entity_id` | UUID | Yes* | *Required if your group has multiple entities |

### Optional-field serialization

Omit an optional key completely when its value is unavailable. Do not send the key with `null`, an
empty string, or a placeholder value. Apply this recursively to `buyer`, every item, and the top-level
request. For example, a buyer without a national ID should be serialized as:

```json
{
  "mobile_number": "512345678"
}
```

not as `{"mobile_number":"512345678","national_id":""}` or with `national_id: null`. Similarly,
omit empty `metadata`, absent `reference_id`, absent `unit_price`, and `entity_id` when it is not
required for a single-entity group. If an optional field is supplied, validate it before sending.

**URLs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `redirect_url` | string | Yes | Buyer lands here after JeelPay flow completes |
| `notification_url` | string | Yes | JeelPay POSTs webhook updates here |

### Response

```json
{
  "checkout_id": "9e79d502-231d-449b-b419-a674b687df51",
  "redirect_url": "https://checkout.jeel.co/pay/9e79d502...",
  "creation_date": "2025-09-15T10:30:00Z",
  "reference_id": "order_1234",
  "metadata": {
    "student_id": "STU-001"
  }
}
```

Redirect the buyer to `redirect_url` immediately after receiving this response.

### Response Headers

All responses include a `tx_id` header for debugging and support:

```
tx_id: b7734ba3-d2a0-476d-87fe-9ba49b64ef6c
```

Extract and store this value with your transaction record for support ticket resolution.

**Node.js:**
```javascript
const txId = response.headers.get('tx_id');
```

**Python:**
```python
tx_id = response.headers.get('tx_id')
```

### Error Responses

| Status | Meaning |
|--------|---------|
| `400` | Validation error, wrong checkout type for your account, missing `entity_id` for a multi-entity group, or concurrent duplicate `IDEMPOTENCY-001` |
| `401` | Invalid or expired access token |
| `500` | JeelPay server error |

Error body:
```json
{
  "errors": [
    {
      "id": "VALIDATION_ERROR",
      "message": "mobile_number is invalid",
      "description": "mobile_number must match ^5[0-9]{8}$"
    }
  ]
}
```

## Get Checkout Status

```http
GET /v3/checkout/{checkout_id}
Authorization: Bearer {access_token}
```

### Response

```json
{
  "checkout_id": "9e79d502-231d-449b-b419-a674b687df51",
  "checkout_type": "ITEMS",
  "status": "SUCCEEDED",
  "reference_id": "order_1234",
  "metadata": {}
}
```

### Status Values

| Status | Meaning |
|--------|---------|
| `PENDING` | Checkout created, awaiting buyer action |
| `SUCCEEDED` | Buyer paid down payment; installment plan created |
| `REJECTED` | Buyer declined, wasn't eligible, or merchant cancelled |
| `EXPIRED` | No action for 2 hours — checkout timed out |

## Code Example (Node.js)

```javascript
function omitAbsentFields(object) {
  return Object.fromEntries(
    Object.entries(object).filter(([, value]) =>
      value !== undefined && value !== null && value !== ''
    )
  );
}

async function createItemsCheckout(buyer, items, referenceId, idempotencyKey) {
  // idempotencyKey should be a UUID generated before calling this function
  // and saved to your database with status: 'PENDING'
  const token = await getAccessToken(); // use cached token
  const response = await fetch(`${process.env.JEELPAY_API_URL}/v3/checkout`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey, // prevents duplicates on retry
    },
    body: JSON.stringify(omitAbsentFields({
      reference_id: referenceId,
      buyer: omitAbsentFields(buyer),
      items: items.map(omitAbsentFields),
      urls: {
        redirect_url: process.env.JEELPAY_REDIRECT_URL,
        notification_url: process.env.JEELPAY_WEBHOOK_URL,
      },
    })),
  });

  // Extract tx_id for debugging/support (present even on errors)
  const txId = response.headers.get('tx_id');

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`JeelPay checkout failed (tx_id: ${txId}): ${JSON.stringify(error.errors)}`);
  }

  const data = await response.json(); // { checkout_id, redirect_url, ... }
  return { ...data, txId }; // include tx_id for storage
}
```

## Important Notes

- **Always use idempotency keys.** Generate a UUID, save it to your database with status `PENDING`, then include it as the `Idempotency-Key` header. If a network timeout occurs, retry with the same key to prevent duplicate checkouts. Keys expire after 24 hours.
- **Always extract and store tx_id.** Every response includes a `tx_id` header for debugging. Store it with your transaction record for support ticket resolution.
- Checkouts expire after **2 hours** of inactivity (`status` becomes `EXPIRED`). Create a new checkout if the buyer returns later.
- `total_cost` must equal `unit_price × quantity` with exactly 2 decimal places.
- Omit optional fields entirely when no real value is available; never send `null`, empty strings, or
  empty placeholder objects for optional data.
- All amounts are in **SAR (Saudi Riyals)**. No unit conversion needed — just SAR with 2 decimal places (e.g., `3500.00`).
- The `notification_url` must be publicly accessible — JeelPay cannot POST to `localhost`.
- `metadata` values are echoed back in webhooks and status responses — useful for correlating with your internal records.
- Installment split: ≤ 5,000 SAR total = 4 payments; > 5,000 SAR = per agreement with JeelPay.
