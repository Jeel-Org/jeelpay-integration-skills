# JeelPay Schooling Checkout

Use this checkout type for **traditional schools (K-12)** with student-based tuition fees.

If your entity is a university, institute, training center, or higher education provider, use
`api-checkout-items.md` instead.

## Checkout Flow

1. Generate a UUID for the idempotency key and save to your database (status: PENDING)
2. Your server creates a checkout via `POST /v3/checkout/schooling` with `Idempotency-Key` header → receive `checkout_id` + `redirect_url`
3. Extract the `tx_id` from response headers and store it for debugging/support
4. If timeout occurs, retry with the same idempotency key
5. Redirect the buyer (parent or guardian) to `redirect_url` (JeelPay's hosted checkout page)
6. Buyer logs in / registers on JeelPay, reviews student info, completes KYC and down payment
7. JeelPay redirects buyer back to your `redirect_url`
8. JeelPay sends a webhook to your `notification_url` confirming the result
9. You can also poll `GET /v3/checkout/{id}` at any time for the current status

## Create Checkout

```http
POST /v3/checkout/schooling
Authorization: Bearer {access_token}
Content-Type: application/json
Idempotency-Key: {uuid}
```

> **Idempotency:** Always generate a UUID first, save it to your database, then include it as the `Idempotency-Key` header. If a network timeout occurs, retry with the same key to prevent duplicate checkouts.

### Request Schema

```json
{
  "reference_id": "enrollment_2025_0012",
  "buyer": {
    "first_name": "Zayed",
    "last_name": "Al-Abbad",
    "mobile_number": "512345678",
    "email": "zayed@example.com",
    "national_id": "1098765432"
  },
  "students": [
    {
      "name": "Nasser Al-jbreen",
      "national_id": "2098765432",
      "entity_id": "uuid-of-school-entity",
      "educational_year_id": "uuid-of-academic-year",
      "cost": 8500.00,
      "reference_id": "student_grade5_001"
    }
  ],
  "urls": {
    "redirect_url": "https://yourschool.edu.sa/payment/result",
    "notification_url": "https://yourschool.edu.sa/webhooks/jeelpay"
  },
  "metadata": {
    "grade": "5",
    "academic_year": "2025-2026"
  }
}
```

### Field Reference

**Top level:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `reference_id` | string | No | Your internal enrollment/order ID (max 255 chars) |
| `buyer` | object | Yes | The parent/guardian paying (see buyer fields below) |
| `students` | array | Yes | At least one student |
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

**Student:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | 1–255 chars, student's full name |
| `national_id` | string | Yes | 10 digits: `^[12][0-9]{9}$` |
| `entity_id` | UUID | Yes | The school's JeelPay entity ID |
| `educational_year_id` | UUID | Yes | Academic year ID from JeelPay (provided during onboarding) |
| `cost` | number | Yes | Total tuition fee, 2 decimal places, min 0 |
| `reference_id` | string | No | Your internal student record ID |

**URLs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `redirect_url` | string | Yes | Buyer lands here after JeelPay flow completes |
| `notification_url` | string | Yes | JeelPay POSTs webhook updates here |

### Response

```json
{
  "checkout_id": "9e79d502-231d-449b-b419-a674b687df51",
  "redirect_url": "https://checkout.jeel.co/schooling/9e79d502...",
  "creation_date": "2025-09-15T10:30:00Z",
  "reference_id": "enrollment_2025_0012",
  "metadata": {
    "grade": "5"
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

**Python:**
```python
tx_id = response.headers.get('tx_id')
```

**Node.js:**
```javascript
const txId = response.headers.get('tx_id');
```

### Error Responses

| Status | Meaning |
|--------|---------|
| `400` | Validation error or wrong checkout type for your account (items accounts can't use this endpoint) |
| `401` | Invalid or expired access token |
| `404` | Group not found (your entity isn't registered or the entity_id is wrong) |
| `500` | JeelPay server error |

Error body:
```json
{
  "errors": [
    {
      "id": "VALIDATION_ERROR",
      "message": "national_id is required",
      "description": "Each student must have a national_id"
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
  "checkout_type": "SCHOOLING",
  "status": "SUCCEEDED",
  "reference_id": "enrollment_2025_0012",
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

## Code Example (Python / Django)

```python
import os, uuid, requests

def create_schooling_checkout(buyer: dict, students: list, reference_id: str, order_record) -> dict:
    # Generate idempotency key and save to database first
    idempotency_key = str(uuid.uuid4())
    order_record.jeel_idempotency_key = idempotency_key
    order_record.status = 'PENDING'
    order_record.save()
    
    token = get_access_token()  # use cached token
    payload = {
        "reference_id": reference_id,
        "buyer": buyer,
        "students": students,
        "urls": {
            "redirect_url": os.environ["JEELPAY_REDIRECT_URL"],
            "notification_url": os.environ["JEELPAY_WEBHOOK_URL"],
        },
    }
    response = requests.post(
        f"{os.environ['JEELPAY_API_URL']}/v3/checkout/schooling",
        json=payload,
        headers={
            "Authorization": f"Bearer {token}",
            "Idempotency-Key": idempotency_key,  # prevents duplicates on retry
        },
    )
    
    # Extract tx_id for debugging/support (present even on errors)
    tx_id = response.headers.get('tx_id')
    order_record.jeel_tx_id = tx_id
    
    if not response.ok:
        order_record.save()  # save tx_id even on errors
        errors = response.json().get("errors", [])
        raise ValueError(f"JeelPay checkout failed (tx_id: {tx_id}): {errors}")
    
    data = response.json()  # { checkout_id, redirect_url, ... }
    order_record.checkout_id = data['checkout_id']
    order_record.save()
    return data
```

## Important Notes

- **Always use idempotency keys.** Generate a UUID, save it to your database with status `PENDING`, then include it as the `Idempotency-Key` header. If a network timeout occurs, retry with the same key to prevent duplicate checkouts. Keys expire after 24 hours.
- **Always extract and store tx_id.** Every response includes a `tx_id` header for debugging. Store it with your transaction record for support ticket resolution.
- Checkouts expire after **2 hours** of inactivity (`status` becomes `EXPIRED`). Create a new checkout for retries.
- All amounts are in **SAR (Saudi Riyals)** with exactly 2 decimal places (e.g., `8500.00`). No currency conversion or smallest-unit representation needed.
- `educational_year_id` must be a valid active educational year from JeelPay. Fetch the current list from the public endpoint (no auth required):
  ```
  GET https://api.sandbox.jeel.co/v1/public/educational-years
  ```
  Returns the list of active educational years with their IDs. Use the ID matching the student's current academic year.
- A single checkout can include multiple students (one parent paying for siblings, for example).
- The `notification_url` must be publicly accessible — JeelPay cannot POST to `localhost`.
- `metadata` is echoed back in webhooks and status responses — useful for correlating with your enrollment system.
- Installment split: ≤ 5,000 SAR per student = 4 payments; > 5,000 SAR = per agreement with JeelPay.
