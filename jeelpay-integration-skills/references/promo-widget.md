# Jeel Pay Promo Widget

Use the hosted promo widget when a partner wants customers to preview an estimated payment schedule on
a product, cart, or checkout page. The estimate is informational; hosted Jeel Pay checkout remains
authoritative and may show different amounts or dates after customer-specific checks.

Start with the [Jeel Pay Widget Studio](https://widget-studio.jeel.co) when possible. It previews the
responsive result and generates an installation snippet for the selected entity, price, language, and
environment.

## Installation

Add a target element, load the Jeel-hosted browser bundle, and initialize `JeelPromo` after the target
exists in the DOM:

```html
<div class="course-price">SAR 2,000.00</div>
<div id="JeelPayPromo"></div>
<button type="button">Continue to checkout</button>

<script src="https://widget.jeel.co/jeel-promo.js"></script>
<script>
  const promo = new JeelPromo({
    selector: "#JeelPayPromo",
    entityId: "00000000-0000-4000-8000-000000000001",
    educationalYearId: "00000000-0000-4000-8000-000000000002",
    price: "2000.00",
    lang: "en",
    source: "product",
    variant: "standard",
    theme: "light",
    environment: "production",
    checkoutUrl: "/checkout",
  });

  // When the displayed price changes:
  promo.update({ price: "2500.00" });

  // Before the host view is removed:
  promo.destroy();
</script>
```

The bundle exposes `window.JeelPromo`. It does not use a public key, client ID, access token, cookie,
customer identity, or customer secret. Never put Jeel Pay API credentials in browser code.

## Configuration

| Option | Required | Values and behavior |
| --- | --- | --- |
| `selector` | Yes | CSS selector for the host element. Immutable after initialization. |
| `entityId` | Yes | Jeel Pay entity UUID. |
| `price` | Yes | Positive SAR amount as a string or number, with at most two decimal places. |
| `environment` | No | `production` or `sandbox`; defaults to `production`. |
| `educationalYearId` | No | Contract-selection UUID. Supply it for Schooling API entities or when multiple educational-year contracts may apply. |
| `lang` | No | `en` or `ar`; otherwise follows the page's `<html lang>`. |
| `source` | No | `product`, `cart`, or `checkout`; presentation only. Defaults to `product`. |
| `variant` | No | `standard` or `compact`; cart and checkout default to `compact`. |
| `theme` | No | `light`, `dark`, or `inherit`; defaults to `light`. |
| `checkoutUrl` | No | Relative or absolute HTTP(S) URL for “Continue with Jeel Pay.” |
| `onContinue` | No | Callback for SPA/custom navigation. Cannot be combined with `checkoutUrl`. |

Omit optional configuration keys completely when unused. Do not pass empty strings or `null`.
`checkoutUrl` and `onContinue` are mutually exclusive. With neither, the dialog is informational and
shows only a close action.

Relative checkout URLs follow normal browser URL resolution. From
`https://partner.com/courses/course-name/`, `/checkout` resolves to the origin root while `checkout` and
`./checkout` resolve under `/courses/course-name/`. Use an absolute URL for another host.

## Placement and Lifecycle

- Place the widget directly below the amount it describes and before the primary checkout action when
  the layout allows it.
- Use one visible target per widget instance. For mixed-entity carts, render a separate widget beside
  each entity total because each estimate resolves one entity and one contract.
- Let the target use available width. Do not force a fixed height or hide overflow.
- Prefer `compact` in constrained cart or checkout layouts.
- In an SPA, initialize after the route renders the target and call `destroy()` before removing it.
- Destroy an existing instance before rerendering into the same target.
- Keep the badge in the page's normal reading and keyboard order.

`promo.update()` merges mutable options, closes any open dialog, clears the prior estimate, aborts the
old request, and fetches a new estimate. `selector` cannot be updated. `destroy()` is idempotent and
aborts requests, closes the dialog, restores page scrolling, and removes widget-owned content. Calling
`update()` after destruction is a programming error.

## Environments and Estimate API

The bundle permits only these API origins:

| Environment | API origin |
| --- | --- |
| `production` or omitted | `https://api.jeel.co` |
| `sandbox` | `https://api.sandbox.jeel.co` |

The widget calls the public endpoint from the browser:

```http
GET /v1/public/promo/estimate?entityId=<entity-id>&price=2000.00&educationalYearId=<educational-year-id>
```

The request has no body, uses `credentials: "omit"`, changes no server state, and returns
`Cache-Control: no-store`. If `educationalYearId` is omitted, the entity must have exactly one applicable
active contract; ambiguous selection is rejected rather than guessed.

Example response:

```json
{
  "currency": "SAR",
  "totalAmount": 2000.00,
  "numberOfPayments": 4,
  "payments": [
    {
      "sequence": 1,
      "type": "DUE_NOW",
      "amount": 500.00,
      "dueDate": "2026-07-28"
    },
    {
      "sequence": 2,
      "type": "INSTALLMENT",
      "amount": 500.00,
      "dueDate": "2026-08-27"
    }
  ],
  "fullPaymentOption": {
    "available": true,
    "discountPercentage": 2.00,
    "discountedAmount": 1960.00
  },
  "estimatedAt": "2026-07-28T09:30:00Z"
}
```

Treat returned amounts, dates, and ordering as authoritative for the estimate. Do not calculate or
rebalance installments in browser code. `fullPaymentOption` is required in a valid response; show its
promotional wording only when `available` is `true`.

The estimate follows the applicable entity contract but does not run customer eligibility checks or
create a checkout, installment request, plan, claim, or payment. It does not account for customer
exposure, available credit, payment-day selection, or checkout-only exceptions.

## Failure and Diagnostic Behavior

The badge remains visible in loading, ready, invalid-configuration, API-failure, and network-failure
states. Disable it while loading or unavailable; do not hide it or display backend errors to customers.
An update aborts the previous request so a stale response cannot overwrite the newest estimate.

Log only the HTTP status, promo error ID, and a non-sensitive widget instance identifier. Do not log
customer data.

## Accessibility

- Preserve the widget's estimate disclaimer.
- Arabic uses `lang="ar"` and RTL; English uses LTR.
- Keep the badge as a real button with its native disabled state.
- Preserve dialog focus movement, focus trapping, Escape/close/backdrop behavior, and focus restoration.
- Do not interfere with reduced-motion handling or the widget's Shadow DOM styles.
- Display bare `YYYY-MM-DD` due dates as calendar dates without timezone conversion.

## Content Security Policy

Merge the required origins into an existing CSP instead of replacing the site's full policy:

```text
default-src 'self';
script-src 'self' https://widget.jeel.co;
connect-src 'self' https://api.jeel.co;
font-src 'self' https://widget.jeel.co;
style-src 'self' 'unsafe-inline';
img-src 'self' data:;
```

For sandbox, use `https://api.sandbox.jeel.co` in `connect-src`. Prefer an HTTP response header. If a
meta policy is unavoidable, place it before the resources it controls; note that meta CSP does not
support every directive and cannot loosen a header policy. Move inline initialization into an allowed
same-origin script or authorize it with a nonce/hash where the host CSP requires that.
