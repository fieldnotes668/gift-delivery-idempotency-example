# Failure and recovery cases

The hardest cases are the ones where the service does not know whether the provider received a send request. The table below separates observations from conclusions.

| Observation | Safe next step | Unsafe shortcut |
| --- | --- | --- |
| Client timed out after checkout | Look up the order by the original idempotency key | Create a second order |
| Worker crashed before provider call | Resume the same delivery job | Start a new checkout |
| Provider call timed out with no acknowledgement | Query by request ID or move to `needs_review` | Declare failure and resend immediately |
| Duplicate provider callback arrived | Deduplicate by provider event ID | Apply the state transition twice |
| Callback arrived out of order | Compare event time and allowed transitions | Roll a confirmed state backward |

## Reconciliation loop

1. Load the durable delivery job and its last known provider request ID.
2. If the provider supports lookup, ask for that exact request's status.
3. Record the evidence source and event ID before changing the customer-facing state.
4. If the answer remains ambiguous, keep `needs_review` and route it to a controlled support path.

The customer-facing message should explain that the current order is being checked and that another purchase is unnecessary. A timeout is a statement about missing evidence, not proof of a failed send.

## Tests worth keeping

- Send two concurrent checkout requests with the same key. Assert one order and one delivery job.
- Repeat the same key with a different amount or region. Assert a conflict and no mutation of the original order.
- Drop the provider acknowledgement after it accepts a request. Assert reconciliation before any resend.
- Deliver the same callback twice. Assert one state transition.
- Request another buyer's gift ID. Assert the response does not reveal whether that ID exists.
- Search responses and logs for redeemable codes and full recipient addresses. Assert none appear.

These checks are more valuable than a happy-path demo because they exercise the uncertainty that produces duplicate charges and duplicate sends.
