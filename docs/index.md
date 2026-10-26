# Gift delivery idempotency example

Digital gift delivery crosses several boundaries: checkout, a job queue, a delivery provider, and a status view. A network timeout at any boundary can make a successful operation look like a failed one. Retrying without a stable identity can create a second order or a second send.

This reference describes a deliberately small contract for a fictional service. It is an engineering example, not an issuer integration or a claim that a provider delivered a real gift.

## What this contract aims to preserve

- A repeated checkout request with the same key resolves to the same order.
- A retry of delivery work does not create a new checkout or charge.
- A provider timeout becomes an uncertain state until reconciliation supplies evidence.
- A public status response never contains redeemable card material.

The [request contract](contract.md) defines the keys and responses. [Failure and recovery cases](failure-cases.md) turn common ambiguous outcomes into testable behavior.

## Three identities, three jobs

| Identity | Created by | Used for |
| --- | --- | --- |
| Checkout idempotency key | Buyer client | Deduplicating one intended purchase |
| Delivery job ID | Service | Tracking the durable send workflow |
| Provider request ID | Delivery adapter | Looking up one external send attempt |

These IDs should not be interchangeable. A buyer may refresh a page or retry checkout, while a worker may retry after a crash, and a provider may send duplicate callbacks. Each event needs the identity of the operation it is repeating.

## Scope

The example assumes an authenticated buyer, a durable order store, and a provider that may support idempotent requests or request-status lookup. If neither provider feature exists, the safe response to an ambiguous timeout is manual review rather than a blind resend.

This public reference is maintained by Field Notes as part of the digital-gifting engineering questions explored at [CardNaro](https://cardnaro.com/). The examples here are independent and contain no customer data or redeemable gift codes.
