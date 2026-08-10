# JeelPay Authentication

JeelPay uses **OAuth 2.1 Client Credentials** flow. You exchange your `client_id` and `client_secret`
for a short-lived bearer token, then include that token on every API request.

## Auth Endpoints

| Environment | Auth URL |
|-------------|----------|
| Sandbox | `https://auth.sandbox.jeel.co/oauth2/token` |
| Production | `https://auth.jeel.co/oauth2/token` |

## Token Request

```http
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
client_secret=YOUR_CLIENT_SECRET
grant_type=client_credentials
```

Send the three fields as a URL-encoded form. Do not send JSON or `multipart/form-data`.

## Token Response

```json
{
  "access_token": "eyJraWQiOiJkMWZhOWEwOS0...",
  "token_type": "Bearer",
  "expires_in": 36000,
  "refresh_expires_in": 0,
  "not-before-policy": 1777382301,
  "scope": "email profile openid"
}
```

`expires_in` is in **seconds** and is the source of truth for token lifetime. Do not hardcode a
particular lifetime. Client Credentials tokens normally do not have a usable refresh token, so request
a new access token after the cached token expires.

## Using the Token

Include it as a Bearer token on all API requests:
```
Authorization: Bearer eyJraWQiOiJkMWZhOWEwOS0...
```

## Token Caching (Required)

The auth endpoint is **rate-limited**. You must cache the token and reuse it until it expires.
Never call the auth endpoint on every API request.

### Caching Logic (pseudocode)

```
function getAccessToken():
    if cachedToken exists:
        if currentTime < cachedToken.expiresAt - 30 seconds:
            return cachedToken.accessToken

    response = POST /oauth2/token with client_id, client_secret, grant_type

    store:
        cachedToken.accessToken = response.access_token
        cachedToken.expiresAt   = currentTime + response.expires_in

    return cachedToken.accessToken
```

The 30-second buffer before expiry avoids race conditions where the token expires mid-request.

### Implementation Notes by Language

**Node.js:**
```javascript
let tokenCache = { accessToken: null, expiresAt: 0 };

async function getAccessToken() {
  if (tokenCache.accessToken && Date.now() < tokenCache.expiresAt - 30000) {
    return tokenCache.accessToken;
  }
  const form = new URLSearchParams({
    client_id: process.env.JEELPAY_CLIENT_ID,
    client_secret: process.env.JEELPAY_CLIENT_SECRET,
    grant_type: 'client_credentials',
  });
  const res = await fetch(process.env.JEELPAY_AUTH_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: form,
  });
  if (!res.ok) throw new Error(`Jeel Pay authentication failed: HTTP ${res.status}`);
  const data = await res.json();
  tokenCache = {
    accessToken: data.access_token,
    expiresAt: Date.now() + data.expires_in * 1000
  };
  return tokenCache.accessToken;
}
```

**Python:** (`requests` encodes `data=` as `application/x-www-form-urlencoded`.)
```python
import os
import time

import requests

_token_cache = {"access_token": None, "expires_at": 0}

def get_access_token():
    if _token_cache["access_token"] and time.time() < _token_cache["expires_at"] - 30:
        return _token_cache["access_token"]
    resp = requests.post(
        os.environ["JEELPAY_AUTH_URL"],
        data={
            "client_id": os.environ["JEELPAY_CLIENT_ID"],
            "client_secret": os.environ["JEELPAY_CLIENT_SECRET"],
            "grant_type": "client_credentials",
        }
    )
    resp.raise_for_status()
    data = resp.json()
    _token_cache["access_token"] = data["access_token"]
    _token_cache["expires_at"] = time.time() + data["expires_in"]
    return _token_cache["access_token"]
```

**PHP:**
```php
$tokenCache = ['access_token' => null, 'expires_at' => 0];

function getAccessToken(): string {
    global $tokenCache;
    if ($tokenCache['access_token'] && time() < $tokenCache['expires_at'] - 30) {
        return $tokenCache['access_token'];
    }
    $response = Http::asForm()->post(env('JEELPAY_AUTH_URL'), [
        'client_id' => env('JEELPAY_CLIENT_ID'),
        'client_secret' => env('JEELPAY_CLIENT_SECRET'),
        'grant_type' => 'client_credentials',
    ]);
    $response->throw();
    $data = $response->json();
    $tokenCache = [
        'access_token' => $data['access_token'],
        'expires_at'   => time() + $data['expires_in'],
    ];
    return $tokenCache['access_token'];
}
```

**Java (Spring Boot):**
```java
private String accessToken;
private Instant expiresAt = Instant.EPOCH;

public synchronized String getAccessToken() {
    if (accessToken != null && Instant.now().isBefore(expiresAt.minusSeconds(30))) {
        return accessToken;
    }
    MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
    form.add("client_id", clientId);
    form.add("client_secret", clientSecret);
    form.add("grant_type", "client_credentials");
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    TokenResponse response = restTemplate.postForObject(authUrl,
        new HttpEntity<>(form, headers), TokenResponse.class);
    accessToken = response.getAccessToken();
    expiresAt = Instant.now().plusSeconds(response.getExpiresIn());
    return accessToken;
}
```

## Environment Variables

```env
JEELPAY_CLIENT_ID=your_client_id
JEELPAY_CLIENT_SECRET=your_client_secret
JEELPAY_AUTH_URL=https://auth.sandbox.jeel.co/oauth2/token
JEELPAY_API_URL=https://api.sandbox.jeel.co
```

Switch to production by updating `JEELPAY_AUTH_URL` and `JEELPAY_API_URL` — no code changes needed.

## Security Notes

- Store `client_id` and `client_secret` as environment variables or in a secrets manager — never in code
- Never expose credentials in client-side JavaScript or mobile app bundles
- All auth requests must go through your backend
- Use HTTPS only (all JeelPay endpoints enforce HTTPS)
- The access token is a JWT — treat it like a password
