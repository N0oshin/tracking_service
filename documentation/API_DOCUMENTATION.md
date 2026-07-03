# tracking-service — API Reference

**Base URL:** `http://localhost:5002` (local) or your deployed domain (`TRACKING_SERVICE_PORT`, default `5002`)

All responses are `application/json`. All requests with a body expect `Content-Type: application/json`.

---

## Authentication

This service uses a **email OTP → short-lived JWT** flow, not admin login. There is no username/password.

1. Client calls `POST /api/trackingdetail/request-otp` with the customer's email.
2. Customer receives a 6-digit code by email.
3. Client calls `POST /api/trackingdetail/verify-otp` with that code → receives a `token`.
4. Client sends that token on every subsequent protected call:

```
Authorization: Bearer <token>
```

The token expires after `expiresIn` seconds (see `verify-otp` response) — **15 minutes by default**. After it expires, repeat steps 1–3.

Routes marked **Auth: Bearer** below require this header. Routes marked **Auth: none** are public.

> Alternative for trusted backend-to-backend calls only: if the service operator has configured `TRACKING_API_KEY`, sending header `x-api-key: <that value>` satisfies auth on any Bearer-protected route without a token, and `email` must then be passed explicitly (query param or body) since there's no token to derive it from. Frontend clients should not use this — it's for server-side integrations.

### Auth failure responses (shared by every Bearer-protected route)

Every route marked **Auth: Bearer** below (`orders` GET/POST, `tracking/:trackingNumber`, `address-change-requests` GET, `order/:orderNumber/address-change-request` POST) is gated by the same middleware, which can short-circuit with:

**Response `401`** (no `Authorization` header / no token):
```json
{ "error": "OTP verification required" }
```

**Response `401`** (token parses but its embedded email is missing/invalid):
```json
{ "error": "Invalid access token" }
```

**Response `401`** (token expired, malformed, or signature invalid):
```json
{ "error": "Invalid or expired access token" }
```

**Response `500`** (server misconfiguration — `JWT_SECRET` not set):
```json
{ "error": "Server missing JWT secret" }
```

On any `401` here, re-run the OTP flow (`request-otp` → `verify-otp`) to get a fresh token — don't retry the same request with the same token.

---

## Endpoints

| Method | Path | Auth |
|---|---|---|
| GET | `/health` | none |
| POST | `/api/trackingdetail/request-otp` | none |
| POST | `/api/trackingdetail/verify-otp` | none |
| GET | `/api/trackingdetail/orders` | Bearer |
| POST | `/api/trackingdetail/orders` | Bearer |
| POST | `/api/trackingdetail/orders-by-email` | none |
| GET | `/api/trackingdetail/tracking/:trackingNumber` | Bearer |
| GET | `/api/trackingdetail/address-change-requests` | Bearer |
| POST | `/api/trackingdetail/order/:orderNumber/address-change-request` | Bearer |

---

## Health

### GET `/health`

No auth. No request body.

**Response `200`:**
```json
{
  "ok": true,
  "service": "tracking-service",
  "db": "connected"
}
```

**Response `500`** (database unreachable):
```json
{
  "ok": false,
  "service": "tracking-service",
  "db": "disconnected",
  "error": "<db error message>"
}
```

---

## OTP Auth Flow

### POST `/api/trackingdetail/request-otp`

Sends a 6-digit OTP to the given email **only if that email has at least one order**. Always responds `{ "ok": true }` either way — the response deliberately does **not** reveal whether the email exists in the system.

**Auth:** none

**Request body:**
```json
{
  "email": "jane@example.com"
}
```

**Response `200`:**
```json
{ "ok": true }
```

**Response `400`:**
```json
{ "ok": false, "error": "Valid email is required" }
```

**Response `500`:**
```json
{ "ok": false, "error": "Failed to request OTP" }
```

---

### POST `/api/trackingdetail/verify-otp`

Verifies the OTP and, if correct, issues an access token.

**Auth:** none

**Request body:**
```json
{
  "email": "jane@example.com",
  "otp": "483920"
}
```
`otp` must be exactly 6 digits.

**Response `200`:**
```json
{
  "ok": true,
  "token": "<jwt string>",
  "expiresIn": 900
}
```
`expiresIn` is in seconds. Use `token` as the `Bearer` value on protected routes.

**Response `400`** (one of, same shape, different `error` text):
```json
{ "ok": false, "error": "Valid email is required" }
```
```json
{ "ok": false, "error": "Valid 6-digit OTP is required" }
```
```json
{ "ok": false, "error": "Invalid OTP" }
```
```json
{ "ok": false, "error": "OTP already used. Please request a new OTP." }
```
```json
{ "ok": false, "error": "OTP expired. Please request a new OTP." }
```
```json
{ "ok": false, "error": "Too many attempts. Please request a new OTP." }
```

**Response `500`:**
```json
{ "ok": false, "error": "Server missing JWT secret" }
```
```json
{ "ok": false, "error": "Failed to verify OTP" }
```

---

## Orders

Order/item/payment fields are `SELECT *` against the shared `orders`, `order_items`, and `payments` tables owned by `admin-service` — the field lists below reflect the **current** schema and may gain/lose columns over time. Don't assume the list is exhaustive or fixed; read fields defensively (e.g. `order.tracking_number ?? null`).

### GET `/api/trackingdetail/orders`

**Auth:** Bearer (or `x-api-key` + `email` query param)

**Query params:**

| Param | Type | Required | Default | Max |
|---|---|---|---|---|
| `email` | string | only if not using a Bearer token | — | — |
| `limit` | number | no | 50 | 200 |

**Response `200`:**
```json
{
  "email": "jane@example.com",
  "orders": [
    {
      "order": {
        "id": 15,
        "order_number": "ORD-20260630-163929904-273E19",
        "user_id": null,
        "customer_email": "jane@example.com",
        "customer_name": "Jane Doe",
        "customer_phone": null,
        "shipping_address": "22 Baker St",
        "shipping_city": "London",
        "shipping_state": null,
        "shipping_zip": "NW1 6XE",
        "shipping_country": "United Kingdom",
        "tracking_number": null,
        "currency": "GBP",
        "subtotal": "260.00",
        "shipping": "0.00",
        "total": "260.00",
        "status": "pending",
        "payment_status": "pending",
        "payment_method": "BPC-157 & TB-500 x2 @ £130",
        "promo_code": null,
        "promo_discount": null,
        "discount_amount": "0.00",
        "payment_rejection_reason": null,
        "admin_payment_remark": null,
        "admin_payment_screenshot_filename": null,
        "admin_payment_screenshot_url": null,
        "ibalticx_invoice_sent_at": null,
        "ibalticx_invoice_to": null,
        "ibalticx_invoice_message_id": null,
        "bank_account_used": "ibalticx",
        "created_at": "2026-06-30T16:39:28.000Z",
        "updated_at": "2026-06-30T16:39:28.026Z",
        "total_before_discount": "260.00",
        "total_after_discount": "260.00",
        "promo_discount_percent": "0.00",
        "promo_valid": 0,
        "items_text": null,
        "payment_screenshot_filename": null,
        "payment_screenshot_url": "Manual",
        "reserved_at": null,
        "submitted_at": "2026-06-30T16:39:28.000Z",
        "credits_applied": "0.00",
        "total_before_credits": "260.00",
        "credits_reserved": "0.00"
      },
      "items": [
        {
          "id": 23,
          "order_id": 15,
          "product_id": null,
          "name": "BPC-157 & TB-500",
          "sku": "bpc-157-tb-500",
          "quantity": 2,
          "unit_price": "130.00",
          "line_total": "260.00"
        }
      ],
      "payments": [
        {
          "id": 15,
          "order_id": 15,
          "user_id": null,
          "provider": "Manual",
          "provider_id": "CHECKOUT-ORD-20260630-163929904-273E19",
          "amount": "260.00",
          "currency": "GBP",
          "status": "pending",
          "webhook_received": 0,
          "final_status": null,
          "status_checked_at": null,
          "bank_name": null,
          "raw_response": null,
          "created_at": "2026-06-30T16:39:28.000Z",
          "updated_at": "2026-06-30T16:39:28.026Z"
        }
      ]
    }
  ]
}
```
`orders` is sorted newest-first (`created_at DESC`). Each entry's `items`/`payments` arrays can be empty if the order has none.

**Response `400`:**
```json
{ "error": "Valid email query param is required" }
```

**Response `500`:**
```json
{ "error": "Failed to fetch tracking details", "details": "<error message>" }
```

---

### POST `/api/trackingdetail/orders`

Identical to `GET /api/trackingdetail/orders` above, but `email`/`limit` are read from the JSON body instead of the query string. Same response shape.

**Auth:** Bearer (or `x-api-key` + `email` in body)

**Request body:**
```json
{
  "email": "jane@example.com",
  "limit": 50
}
```
`limit` optional, default 50, max 200. `email` optional if a Bearer token is used (the token's email is used instead).

**Response `200`:** same shape as `GET /api/trackingdetail/orders` above (`{ "email", "orders" }`).

**Response `400` / `500`:** same as `GET /api/trackingdetail/orders` above.

---

### POST `/api/trackingdetail/orders-by-email`

Same underlying order/item/payment lookup, but **no OTP/token required at all** — for callers that have already authenticated the user some other way. **Note the different response envelope** (`ok`/`orders`, not `email`/`orders`).

**Auth:** none

**Request body:**
```json
{
  "email": "jane@example.com",
  "limit": 50
}
```
`limit` optional, default 50.

**Response `200`:**
```json
{
  "ok": true,
  "orders": [
    { "order": { "...": "same shape as above" }, "items": [ "..." ], "payments": [ "..." ] }
  ]
}
```

**Response `400`:**
```json
{ "ok": false, "error": "Valid email is required" }
```

**Response `500`:**
```json
{ "ok": false, "error": "Failed to fetch orders" }
```

---

## Carrier Tracking

### GET `/api/trackingdetail/tracking/:trackingNumber`

Fetches live carrier tracking events from an external tracking provider.

> **Currently unconfigured in every environment shipped so far** — until the operator sets `TRACKING_EVENTS_API_BASE_URL`, this always returns the `400` "not configured" response below. Don't build against this endpoint until confirming with backend that a provider is wired up.

**Auth:** Bearer (or `x-api-key`)

**URL params:**

| Param | Type | Notes |
|---|---|---|
| `trackingNumber` | string | the carrier tracking number |

**Response `200`:**
```json
{
  "ok": true,
  "trackingNumber": "AB123456789GB",
  "carrier": "royalmail",
  "events": [
    {
      "status": "Delivered",
      "location": "London, UK",
      "timestamp": "2026-07-01T09:15:00Z",
      "raw": { "...": "original event object exactly as returned by the carrier provider" }
    }
  ]
}
```

**Response `400`:**
```json
{ "ok": false, "error": "Tracking events API is not configured" }
```
also used for a missing `trackingNumber` or a provider-side error, with a different `error` string.

**Response `500`:**
```json
{ "ok": false, "error": "Failed to fetch tracking" }
```

---

## Address Change Requests

### GET `/api/trackingdetail/address-change-requests`

Lists the requesting customer's shipping-address change requests, newest first.

**Auth:** Bearer (or `x-api-key` + `email` query param)

**Query params:**

| Param | Type | Required | Default | Max |
|---|---|---|---|---|
| `email` | string | only if not using a Bearer token | — | — |
| `limit` | number | no | 200 | 500 |

**Response `200`:**
```json
{
  "requests": [
    {
      "id": 1,
      "order_id": 15,
      "order_number": "ORD-20260630-163929904-273E19",
      "customer_email": "jane@example.com",
      "current_shipping_json": "{\"shipping_address\":\"22 Baker St\",\"shipping_city\":\"London\",\"shipping_zip\":\"NW1 6XE\",\"shipping_country\":\"United Kingdom\"}",
      "requested_shipping_json": "{\"shipping_address\":\"99 New Street\",\"shipping_city\":\"Manchester\",\"shipping_zip\":\"M1 1AA\",\"shipping_country\":\"United Kingdom\"}",
      "status": "pending",
      "admin_id": null,
      "admin_note": null,
      "decided_at": null,
      "created_at": "2026-07-03T14:28:40.000Z",
      "updated_at": "2026-07-03T14:28:40.000Z"
    }
  ]
}
```

**Important:** `current_shipping_json` and `requested_shipping_json` are **JSON-encoded strings**, not nested objects — `JSON.parse()` them client-side. Each parses to:
```json
{
  "shipping_address": "string",
  "shipping_city": "string",
  "shipping_zip": "string",
  "shipping_country": "string"
}
```

`status` is one of: `"pending"`, `"approved"`, `"rejected"` (approval/rejection happens in `admin-service`, not here).

**Response `400`:**
```json
{ "error": "Valid email is required" }
```

**Response `500`:**
```json
{ "error": "Failed to fetch address change requests" }
```

---

### POST `/api/trackingdetail/order/:orderNumber/address-change-request`

Submits a request to change an order's shipping address. Fails if the order doesn't belong to the requesting email, or if a `pending` request already exists for that order.

**Auth:** Bearer (or `x-api-key` + `email` in body)

**URL params:**

| Param | Type | Notes |
|---|---|---|
| `orderNumber` | string | the order's `order_number` |

**Request body:**
```json
{
  "email": "jane@example.com",
  "shipping_address": "99 New Street",
  "shipping_city": "Manchester",
  "shipping_zip": "M1 1AA",
  "shipping_country": "United Kingdom"
}
```
- `shipping_address` — **required**
- `shipping_city`, `shipping_zip`, `shipping_country` — optional (default to empty string if omitted)
- `email` — optional if a Bearer token is used (the token's email is used instead)

**Response `200`:**
```json
{
  "success": true,
  "id": 1
}
```
`id` is the new `order_address_change_requests` row id — use it to correlate with `GET /api/trackingdetail/address-change-requests` later.

**Response `400`:**
```json
{ "error": "Valid email is required" }
```
```json
{ "error": "orderNumber is required" }
```
```json
{ "error": "shipping_address is required" }
```

**Response `404`** (order doesn't exist, or doesn't belong to `email`):
```json
{ "error": "Order not found" }
```

**Response `409`** (a pending request already exists for this order):
```json
{ "error": "A pending address change request already exists for this order." }
```

**Response `500`:**
```json
{ "error": "Failed to submit address change request" }
```

---

## Error response shape summary

There are two error envelope styles used across this service — **check for the right one per endpoint**, they are not interchangeable:

- **`ok`-style** (OTP flow, `orders-by-email`, tracking): `{ "ok": false, "error": "..." }`
- **plain style** (`orders` GET/POST, address-change-requests, health failure aside, and every shared auth-middleware failure — see "Auth failure responses" above): `{ "error": "..." }` — except `/health`, which uses `{ "ok": false, "service", "db", "error" }`.

Refer to each endpoint section above for its exact shape.
