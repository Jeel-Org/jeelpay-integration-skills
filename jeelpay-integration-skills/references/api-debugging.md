# Support and Debugging

Every API response includes a `tx_id` header for debugging and support.

## tx_id Header

The `tx_id` is a unique UUID generated for every request. It appears in **all** response headers regardless of status code (2xx, 4xx, 5xx).

Example:
```
tx_id: b7734ba3-d2a0-476d-87fe-9ba49b64ef6c
```

## Extract and Store

Always extract `tx_id` from response headers and store it with your transaction record.

**Node.js:**
```javascript
const response = await fetch(url, { method: 'POST', ... });
const txId = response.headers.get('tx_id');
// Store txId with your order record
order.jeelTxId = txId;
await order.save();
```

**Python:**
```python
response = requests.post(url, json=payload, headers=headers)
tx_id = response.headers.get('tx_id')
# Store tx_id with your transaction record
order.jeel_tx_id = tx_id
order.save()
```

**PHP:**
```php
$response = Http::post($url, $payload);
$txId = $response->header('tx_id');
// Store txId with your order record
$order->jeel_tx_id = $txId;
$order->save();
```

**Java:**
```java
HttpResponse<String> response = httpClient.send(request, ...);
String txId = response.headers().firstValue("tx_id").orElse(null);
// Store txId with your transaction record
order.setJeelTxId(txId);
orderRepository.save(order);
```

## Error Handling

Include `tx_id` in all error logs:

```python
try:
    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()
except requests.exceptions.HTTPError as e:
    tx_id = response.headers.get('tx_id')
    logger.error(f"API request failed. tx_id: {tx_id}, Error: {e}")
```

## Contacting Support

When contacting JeelPay support, always provide:

1. **tx_id** from the response headers
2. **Timestamp** when the request was made
3. **API endpoint** that was called
4. **Brief description** of the issue

Example support request:
```
Subject: Payment initiation failed for order #12345

tx_id: b7734ba3-d2a0-476d-87fe-9ba49b64ef6c
Timestamp: 2026-03-27 02:10:27 UTC
Endpoint: POST /v3/checkout
Issue: Received HTTP 500 error when creating checkout session.
```

## Best Practices

- Store `tx_id` for **both successful and failed** requests
- Display `tx_id` in your admin panels for easy reference
- Include `tx_id` in error notifications and alerts
- `tx_id` is permanently stored in JeelPay logs and can be used for support indefinitely
