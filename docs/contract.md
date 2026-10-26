# Request contract

## Checkout request

```http
POST /v1/gifts
Authorization: Bearer <buyer-token>
Idempotency-Key: checkout_7f3a
Content-Type: application/json

{"recipient_region":"US","card_type":"example","amount":25}
```

The key identifies **one buyer intent**, not the text of the HTTP request. Scope it to the authenticated buyer and endpoint. Store a fingerprint of the normalized request body with the key. If the same key is reused with a different fingerprint, return a conflict instead of silently changing the original order.

```http
HTTP/1.1 202 Accepted
Location: /v1/gifts/g_42
Content-Type: application/json

{"gift_id":"g_42","delivery_state":"send_queued"}
```

An exact replay returns the existing `gift_id` and current state. The service must commit the key-to-order mapping atomically with the order creation decision; otherwise two concurrent requests can both believe they won.

## Delivery work

The checkout transaction creates one durable delivery job. A worker may claim the job more than once, so the job handler reads the stored job ID before calling the provider. It records the provider request ID before treating an acknowledgement as final evidence.

If the provider accepts a stable idempotency key, derive it from the delivery job and use the same value on retries of that attempt. If the provider offers status lookup, query by stored request ID after a timeout. Never derive a new checkout key from a worker retry.

## Status view

```http
GET /v1/gifts/g_42/delivery
Authorization: Bearer <buyer-token>
```

```json
{
  "gift_id": "g_42",
  "state": "provider_accepted",
  "message": "The delivery provider accepted the send request.",
  "next_check_after_seconds": 30
}
```

`provider_accepted` does not mean the recipient opened an email or redeemed a code. Those claims require separate, trustworthy events. The response omits the gift code, redemption URL, full recipient address, and raw provider payload.

## Response boundaries

| Situation | Suggested response |
| --- | --- |
| Same key and same normalized body | Existing order and its current state |
| Same key but different body | `409 Conflict` |
| Missing authentication | `401 Unauthorized` |
| Unauthorized or unknown gift ID | Same non-disclosing `404 Not Found` behavior |
| Polling too often | `429 Too Many Requests` with a retry interval |

These are example choices. A production API should document its exact retry and retention windows, authorization model, and provider-specific evidence before clients depend on them.
