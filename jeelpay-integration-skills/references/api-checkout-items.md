# JeelPay Items Checkout

Use this checkout type for **universities, institutes, higher education providers, courses, and training centers** — educational entities that don't follow traditional K-12 school settings.

If your entity is a traditional school, use `api-checkout-schooling.md` instead.

## Checkout Flow

1. Your server creates a checkout via `POST /v3/checkout` → receive `checkout_id` + `redirect_url`
2. Redirect the buyer to `redirect_url` (JeelPay's hosted checkout page)
3. Buyer logs in / registers on JeelPay, reviews items, completes KYC and down payment
4. JeelPay redirects buyer back to your `redirect_url`
5. JeelPay sends a webhook to your `notification_url` confirming the result
6. You can also poll `GET /v3/checkout/{id}` at any time for the current status

## Create Checkout

```http
POST /v3/checkout
Authorization: Bearer {access_token}
Content-Type: application/json
```

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
| `first_name` | string | Yes | 1–255 chars |
| `last_name` | string | Yes | 1–255 chars |
| `mobile_number` | string | Yes | Saudi mobile: `^5[0-9]{8}$` (9 digits, starts with 5) |
| `email` | string | No | 0–255 chars |
| `national_id` | string | No | 10 digits: `^[12][0-9]{9}$` |

**Item:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `item_name` | string | Yes | 1–50 chars |
| `quantity` | integer | Yes | Min 1 |
| `unit_price` | number | No | Per-unit price, 2 decimal places |
| `total_cost` | number | Yes | `unit_price × quantity`, 2 decimal places |
| `reference_id` | string | Yes | Your internal item ID |
| `entity_id` | UUID | Yes* | *Required if your group has multiple entities |

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

### Error Responses

| Status | Meaning |
|--------|---------|
| `400` | Validation error, wrong checkout type for your account (schooling accounts can't use this endpoint), or missing `entity_id` on items for multi-entity groups |
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
async function createItemsCheckout(buyer, items, referenceId) {
  const token = await getAccessToken(); // use cached token
  const response = await fetch(`${process.env.JEELPAY_API_URL}/v3/checkout`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      reference_id: referenceId,
      buyer,
      items,
      urls: {
        redirect_url: process.env.JEELPAY_REDIRECT_URL,
        notification_url: process.env.JEELPAY_WEBHOOK_URL,
      },
    }),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`JeelPay checkout failed: ${JSON.stringify(error.errors)}`);
  }

  return response.json(); // { checkout_id, redirect_url, ... }
}
```

## Important Notes

- Checkouts expire after **2 hours** of inactivity (`status` becomes `EXPIRED`). Create a new checkout if the buyer returns later.
- `total_cost` must equal `unit_price × quantity` with exactly 2 decimal places.
- All amounts are in **SAR (Saudi Riyals)**. No unit conversion needed — just SAR with 2 decimal places (e.g., `3500.00`).
- The `notification_url` must be publicly accessible — JeelPay cannot POST to `localhost`.
- `metadata` values are echoed back in webhooks and status responses — useful for correlating with your internal records.
- Installment split: ≤ 5,000 SAR total = 4 payments; > 5,000 SAR = per agreement with JeelPay.
