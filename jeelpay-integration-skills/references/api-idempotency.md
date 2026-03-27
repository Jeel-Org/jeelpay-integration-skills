# Idempotency Keys

Idempotency keys prevent duplicate checkout creation when network timeouts cause retries.

## When to Use

Required on checkout creation endpoints:
- `POST /v3/checkout` (Items checkout)
- `POST /v3/checkout/schooling` (Schooling checkout)

## How It Works

1. Generate a UUID **before** calling the API
2. Save it to your database with PENDING status
3. Send as `Idempotency-Key` header
4. If timeout occurs, retry with the **same** key
5. JeelPay returns the cached response, no duplicate created

Keys expire after 24 hours. Keys are scoped per endpoint — the same key on different endpoints are independent.

## Integration Flow

```
1. Generate UUID
2. Save to DB (key, status=PENDING)
3. Call POST /v3/checkout with Idempotency-Key header
4. If timeout/error → retry with same key
5. On success → update DB status
```

## UUID Generation

**Node.js:**
```javascript
const idempotencyKey = crypto.randomUUID();
```

**Python:**
```python
import uuid
idempotency_key = str(uuid.uuid4())
```

**PHP:**
```php
$idempotencyKey = wp_generate_uuid4(); // WordPress
// or
$idempotencyKey = \Ramsey\Uuid\Uuid::uuid4()->toString();
```

**Java:**
```java
String idempotencyKey = UUID.randomUUID().toString();
```

## Request Example

```http
POST /v3/checkout
Authorization: Bearer {access_token}
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "buyer": { ... },
  "items": [ ... ]
}
```

## Retry Behavior

| Scenario | Result |
|----------|--------|
| First request with key | Checkout created, response cached |
| Retry with same key | Cached response returned, no duplicate |
| New request with different key | New checkout created |
| Request without header | Processed normally (no protection) |

## Error Code

`IDEMPOTENCY-001` — A request with this idempotency key is already being processed (concurrent duplicate). Wait and retry.
