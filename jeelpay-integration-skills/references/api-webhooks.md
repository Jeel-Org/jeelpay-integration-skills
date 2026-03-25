# JeelPay Webhooks

JeelPay sends a `POST` request to your `notification_url` on every checkout status change.
Your webhook endpoint must verify the request signature before processing it.

## Webhook Body

The body is JSON. The structure is the same for both checkout types, differing only in `checkout_type`:

```json
{
  "checkout_id": "9e79d502-231d-449b-b419-a674b687df51",
  "status": "SUCCEEDED",
  "checkout_type": "SCHOOLING",
  "reference_id": "order_1234",
  "metadata": {
    "example_key_1": "example value 1",
    "example_key_2": "example value 2"
  }
}
```

| Field | Values |
|-------|--------|
| `status` | `PENDING`, `SUCCEEDED`, `REJECTED`, `EXPIRED` |
| `checkout_type` | `SCHOOLING`, `ITEMS` |

You'll receive a webhook for **every** status transition, including when a checkout moves to `PENDING`
(checkout was just opened by the buyer) and when it reaches a terminal state.

## Signature Verification

JeelPay includes a signature in the `X-Jeel-Signature` header. Always verify it before processing.

### How the Signature is Calculated

```
signature = base64( HMAC-SHA256( key=client_secret, message=webhook_body_json ) )
```

1. Take the raw webhook body as a JSON string (exactly as received — do not parse and re-serialize)
2. Compute HMAC-SHA256 using your `client_secret` as the key
3. Base64-encode the result
4. Compare with the `X-Jeel-Signature` header

If the signatures don't match, reject the request with HTTP 401.

### Why This Matters

Anyone on the internet can POST to your `notification_url`. Without signature verification, a malicious
actor could send fake "SUCCEEDED" webhooks and trick your system into activating services for unpaid orders.

### Code Examples

**JavaScript (Node.js):**
```javascript
const crypto = require('crypto');

function verifyWebhookSignature(rawBody, signature, clientSecret) {
  const hmac = crypto.createHmac('sha256', clientSecret);
  hmac.update(rawBody);
  const computed = hmac.digest('base64');
  return crypto.timingSafeEqual(
    Buffer.from(computed),
    Buffer.from(signature)
  );
}

// Express webhook handler:
app.post('/webhooks/jeelpay', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-jeel-signature'];
  const rawBody = req.body.toString();

  if (!verifyWebhookSignature(rawBody, signature, process.env.JEELPAY_CLIENT_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const payload = JSON.parse(rawBody);
  // handle payload.status: PENDING, SUCCEEDED, REJECTED, EXPIRED
  res.status(200).send('OK');
});
```

**PHP (Laravel):**
```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

public function handle(Request $request)
{
    $rawBody  = $request->getContent();
    $received = $request->header('X-Jeel-Signature');
    $computed = base64_encode(hash_hmac('sha256', $rawBody, env('JEELPAY_CLIENT_SECRET'), true));

    if (!hash_equals($computed, $received)) {
        return response('Unauthorized', 401);
    }

    $payload = json_decode($rawBody, true);
    // handle $payload['status']
    return response('OK', 200);
}
```

**Python (Django/Flask):**
```python
import hmac, hashlib, base64, json, os

def verify_signature(raw_body: bytes, received_signature: str) -> bool:
    secret = os.environ['JEELPAY_CLIENT_SECRET'].encode()
    computed = base64.b64encode(
        hmac.new(secret, raw_body, hashlib.sha256).digest()
    ).decode()
    return hmac.compare_digest(computed, received_signature)

# Django view:
@csrf_exempt
def jeelpay_webhook(request):
    if request.method != 'POST':
        return HttpResponse(status=405)
    signature = request.headers.get('X-Jeel-Signature', '')
    if not verify_signature(request.body, signature):
        return HttpResponse('Unauthorized', status=401)
    payload = json.loads(request.body)
    # handle payload['status']
    return HttpResponse('OK', status=200)
```

**Java:**
```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;

private boolean verifySignature(String rawBody, String received) throws Exception {
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(clientSecret.getBytes(), "HmacSHA256"));
    String computed = Base64.getEncoder().encodeToString(mac.doFinal(rawBody.getBytes()));
    return MessageDigest.isEqual(computed.getBytes(), received.getBytes());
}
```

## Implementation Requirements

1. **Read raw bytes** — parse the request body as raw bytes/string before any JSON parsing. Most frameworks will re-serialize JSON if you parse it first, which may change whitespace or key ordering and break the signature check.

2. **Use timing-safe comparison** (`crypto.timingSafeEqual`, `hash_equals`, `hmac.compare_digest`) — prevents timing attacks.

3. **Return 200 quickly** — acknowledge the webhook immediately, then process asynchronously if needed. JeelPay may retry if your endpoint times out.

4. **Respond with 200 for known statuses** — even if you don't act on `PENDING`. Returning non-200 may trigger retries.

5. **Handle retries idempotently** — JeelPay may deliver the same webhook more than once. Use `checkout_id` as an idempotency key.

## Status Handling

```
PENDING  → checkout opened, buyer on JeelPay platform
         → no action needed unless you want to show "payment in progress"

SUCCEEDED → buyer paid down payment, installment plan created
          → activate service, update order status to paid

REJECTED  → buyer declined / not eligible / cancelled
          → release reserved inventory, notify buyer to retry

EXPIRED   → no action for 2 hours
          → same handling as REJECTED; checkout cannot be resumed
```
