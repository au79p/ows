# Au79 Open Wholesaler Specification

**Version:** 0.1.0 Draft  
**Status:** Draft

This document is normative. The accompanying `openapi.yaml` describes the same HTTP contract. When this document and the OpenAPI file disagree, this document is authoritative for business semantics and the OpenAPI file is authoritative for wire syntax.

## 1. Scope

This specification defines a small contract between Au79 and an external wholesaler service. The service can provide:

- supported instruments, units, currencies, and features;
- executable price snapshots;
- optional live prices through Server-Sent Events (SSE);
- customer balances;
- synchronous or asynchronous order execution;
- order status lookup;
- optional order-status webhooks.

Provider-specific protocols such as WebSocket, Pusher, Soketi, or custom headers are outside this specification. An adapter may translate them to this contract.

## 2. Conventions

All endpoints use HTTPS and JSON unless stated otherwise. API paths are versioned with `/v1`.

All timestamps are RFC 3339 strings with an explicit UTC offset. Money, weight, quantity, and price values are exact JSON integers. Clients and providers MUST NOT use floating-point JSON numbers for these values.

The credential identifies the customer account. Requests MUST NOT send `customer_id` or another customer identity in the body to select an account.

### 2.1 Quantity and money

A quantity is an object:

```json
{
  "value": 125000,
  "unit": "mg"
}
```

Money is an object:

```json
{
  "value": 983000000,
  "currency": "ktm"
}
```

`value` is a signed 64-bit JSON integer. `unit` and `currency` are strings so providers can add supported values without changing the object shape. Supported values MUST be declared by `GET /v1/capabilities`. A client MUST reject an unsupported value clearly; it MUST NOT silently reinterpret it.

`ktm` means one thousand (1 kilo) tomans. One `ktm` equals 10,000 Iranian rials (`IRR`). `ktm` is an Au79 domain currency unit, not an ISO 4217 currency code.

### 2.2 Signed trade values

Trade direction is encoded in signed values. For a customer's account:

- a quantity that increases the account is positive;
- a quantity that decreases the account is negative;
- money paid by the account is negative;
- money received by the account is positive.

For a gold trade, receiving gold therefore produces a positive gold delta and a negative money delta. Giving gold produces a negative gold delta and a positive money delta. `side` MUST NOT appear in this specification.

Balances are absolute account values, not transaction deltas. The signed convention applies to order requests and execution values.

| Market operation             | Customer account effect         |  `quantity` sign |      `total` sign | Common market term |
| ---------------------------- | ------------------------------- | ---------------: | ----------------: | ------------------ |
| Receive gold from wholesaler | Gold increases; money decreases | Positive (`+mg`) | Negative (`-ktm`) | Buy / ask          |
| Give gold to wholesaler      | Gold decreases; money increases | Negative (`-mg`) | Positive (`+ktm`) | Sell / bid         |

## 3. Authentication

Every endpoint except the SSE stream handshake documentation uses:

```http
Authorization: Bearer <access-token>
```

Tokens MUST be sent over HTTPS, MUST NOT appear in URLs, and MUST NOT be written to logs. A provider MUST scope a token to the customer account and MUST reject requests for another account.

## 4. Capabilities

### `GET /v1/capabilities`

Returns the provider contract and supported business values. The response MUST include:

- `api_version`;
- `instruments`;
- `units`;
- `currencies`;
- `features`;
- `order_measurements`;
- `sse_support`;
- `webhook_support`.

An instrument has a stable `instrument_id`, display name, quantity units, and price currency. `features` declares `orders`, `balances`, `sse`, and `order_status_webhook`.

## 5. Prices

### `GET /v1/prices`

Returns a complete price snapshot. Each price entry MUST include:

```text
instrument_id
quantity
unit_price
quote_id
issued_at
expires_at
status
```

`quantity` is the signed quote basis. Its sign identifies the gold balance direction for which the quote is valid; its absolute value and unit identify the price basis. `unit_price` is the positive money value for one absolute quantity unit. `quote_id` identifies the quote used by an order. `expires_at` is the last time at which the quote can be submitted.

`status` is one of `open`, `expired`, or `closed`. An expired or closed quote MUST return `unit_price: null`; sentinel numbers such as `0` or `-1` MUST NOT be used for this purpose.

The snapshot MUST include `as_of` and MUST be self-contained. A client MUST replace its local price book with a new complete snapshot after a reconnect or a detected sequence gap.

### 5.1 Optional SSE stream

If `sse_support` is true, the provider MUST expose:

```http
GET /v1/prices/stream
Accept: text/event-stream
```

The stream MUST:

- send `id` on every price event;
- send an `event` name and JSON `data`;
- include `occurred_at` and a monotonic `sequence` in event data;
- send heartbeat comments or heartbeat events;
- accept `Last-Event-ID` for reconnect;
- document whether replay is supported;
- send `price.updated` and `price.expired` events at minimum.

After reconnect, a client MUST obtain a complete snapshot. A client MUST ignore an event whose sequence is not newer than the last applied sequence for that instrument and signed quote direction.

SSE carries price updates only. An order MUST still use a quote accepted by the provider's order endpoint.

## 6. Balances

### `GET /v1/balances`

Returns the current balances for the authenticated customer account:

```json
{
  "as_of": "2026-11-04T12:00:00Z",
  "balances": [
    {
      "kind": "gold",
      "instrument_id": "gold-18k",
      "amount": { "value": 150000, "unit": "mg" },
      "credit_limit": { "value": 50000, "unit": "mg" },
      "available_amount": { "value": 200000, "unit": "mg" }
    },
    {
      "kind": "money",
      "amount": { "value": 250000000, "currency": "ktm" },
      "credit_limit": { "value": 100000000, "currency": "ktm" },
      "available_amount": { "value": 350000000, "currency": "ktm" }
    }
  ]
}
```

`kind: gold` MUST include `instrument_id` and a quantity amount. `kind: money` MUST include a money amount. The array MAY contain more than one gold instrument or currency. The provider MUST return at least the account's supported gold balance and money balance needed for its order contract.

`credit_limit` is the maximum non-negative debt permitted for that balance. `available_amount` is the maximum additional amount that the account may currently decrease, after applying the credit limit, holds, pending orders, and provider rules. Both fields are optional and MUST be non-negative when present. A provider MAY omit either field when it cannot calculate it reliably.

## 7. Orders

### `POST /v1/orders`

Required headers:

```http
Authorization: Bearer <access-token>
Content-Type: application/json
Idempotency-Key: <stable-key>
```

The body MUST include `instrument_id` and `quote_id`. It MUST include exactly one of:

- `quantity`: signed fixed quantity order. Positive means gold increases; negative means gold decreases.
- `total`: signed money-target order. Positive means money increases; negative means money decreases.

`description` is optional and MUST be limited to 1000 characters.

The provider MUST compare the submitted quote with its current executable quote. A stale or mismatched quote MUST return `409` with error code `quote_mismatch`; the provider MUST NOT silently replace the submitted price.

`Idempotency-Key` identifies one logical order attempt. A retry of the same attempt MUST reuse the same key and an order with a new intent MUST use a new key. Reusing a key with a different body MUST return `409 idempotency_conflict`.

The provider MAY complete the order synchronously or return an asynchronous status. The status in the response is authoritative for that response.

### 7.1 Order status

The supported statuses are:

```text
accepted
pending
processing
completed
partially_filled
rejected
cancelled
expired
unknown
```

`unknown` means that the provider cannot currently prove whether the order was accepted or completed. A client MUST query the order before deciding whether to create a new order.

### 7.2 Execution and post-order balance

For a terminal successful order (`completed` or `partially_filled`), the response MUST include:

```text
execution.quantity
execution.total
balance_after
```

`execution` contains the exact signed quantity and money that the provider actually moved. These values MAY differ from the requested values because of rounding, partial execution, or provider rules.

`balance_after` is the customer's balance after that execution, not a balance observed at an unrelated later time. It MUST use the same balance schema as `GET /v1/balances`.

For non-final or unsuccessful statuses, `execution` and `balance_after` MAY be omitted. A provider MUST include them once a successful execution is final.

### `GET /v1/orders/{order_id}`

Returns the current status of an order owned by the authenticated customer. It is the mandatory recovery and reconciliation mechanism for timeouts, missing callbacks, and `unknown` responses.

The response MUST include `order_id`, `client_order_id`, `status`, `created_at`, and `updated_at`. `client_order_id` is the original `Idempotency-Key` when no separate client identifier exists.

## 8. Order-status webhook

The webhook is a separate outbound contract inside this specification. It is optional because `GET /v1/orders/{order_id}` is sufficient for recovery.

### `PUT /v1/webhooks/order-status`

When `webhook_support` is true, this endpoint creates or replaces the single order-status subscription:

```json
{
  "url": "https://au79.example/webhooks/wholesaler/orders",
  "secret": "a-secret-created-by-au79"
}
```

The provider MUST store the secret securely and MUST NOT return it in later responses.

### Callback request

The provider POSTs this payload to the configured URL:

```json
{
  "event_id": "evt_123",
  "event_type": "order.status_changed",
  "occurred_at": "2026-11-04T12:00:01Z",
  "order": {}
}
```

The callback MUST use HTTPS and include:

```http
X-Au79-Webhook-Timestamp: 1730721601
X-Au79-Webhook-Signature: sha256=<hex-hmac>
```

The signature is HMAC-SHA256 over the exact raw request body prefixed with the timestamp and a dot:

```text
<timestamp>.<raw-body>
```

The receiver MUST reject signatures outside its allowed clock-skew window and MUST deduplicate by `event_id`. The receiver MUST tolerate duplicate and out-of-order deliveries.

The provider MUST retry non-2xx responses with increasing delays and MUST stop according to its documented retry limit. A successful 2xx response acknowledges receipt; it does not change the order status contract.

## 9. Errors

Error responses use this envelope:

```json
{
  "code": "quote_mismatch",
  "message": "The submitted quote is no longer executable.",
  "retryable": false,
  "retry_after": null,
  "request_id": "req_123",
  "details": {
    "current_quote": {
      "instrument_id": "gold-18k",
      "quantity": { "value": 1, "unit": "mg" },
      "unit_price": { "value": 791000000, "currency": "ktm" },
      "quote_id": "q_gold_in_20261104_120005",
      "issued_at": "2026-11-04T12:00:05Z",
      "expires_at": "2026-11-04T12:00:15Z",
      "status": "open"
    }
  }
}
```

For `quote_mismatch`, `details.current_quote` SHOULD contain the newest executable quote when one is available. `details.current_quote_id` MAY also be included. Clients MAY display this quote without an additional snapshot request, but MUST still follow normal expiry and order validation rules.

Minimum status and code mapping:

| HTTP | Code                      | Meaning                                                          |
| ---- | ------------------------- | ---------------------------------------------------------------- |
| 400  | `invalid_request`         | Malformed or incomplete request                                  |
| 401  | `unauthorized`            | Missing or invalid token                                         |
| 403  | `forbidden`               | Token lacks permission or account is disabled                    |
| 404  | `not_found`               | Resource does not belong to this account                         |
| 501  | `not_implemented`         | Endpoint or requested capability is not implemented              |
| 409  | `quote_mismatch`          | Quote is stale or differs from the executable quote              |
| 409  | `idempotency_conflict`    | Same idempotency key used with a different request               |
| 422  | `business_rule`           | Account, instrument, balance, or limit rule rejected the request |
| 429  | `rate_limited`            | Rate limit exceeded                                              |
| 503  | `temporarily_unavailable` | Provider cannot safely process the request now                   |

`501 not_implemented` is for an endpoint or operation that the provider does not implement. Clients SHOULD use `GET /v1/capabilities` before calling optional features. A provider MUST NOT use `501` to report an invalid request, an unsupported account state, or a temporarily unavailable service.

`retryable` and `retry_after` MUST reflect provider behavior. Clients MUST NOT retry non-retryable errors or create a new order merely because a request timed out.
