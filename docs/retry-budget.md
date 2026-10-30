# A retry budget for delivery workers

The existing request contract protects checkout from duplicate orders. A delivery worker still needs a separate rule for **when it may try the provider again**. Retrying every error is dangerous: a timeout may mean the provider accepted the message but the acknowledgement was lost.

This page defines a small, testable policy for a fictional digital-gift delivery job. It is an example, not a promise about any real provider.

## Classify the outcome before scheduling work

| Observation | What is known | Next action |
| --- | --- | --- |
| Local validation failed before a provider call | No request was sent | Fix the job data or mark it permanently failed; do not consume a provider retry. |
| Provider returned an explicit retryable response, such as a rate limit or temporary outage | This attempt was rejected or could not complete under the provider's documented semantics | Retry the **same delivery job** after the stated delay, subject to the budget. |
| Provider returned a permanent rejection | The request was rejected | Stop automatic attempts; record the category and route the order to support. |
| Connection timed out after the request may have left the service | Outcome is unknown | Reconcile using the provider request ID. Do not send again solely because the timer expired. |
| Provider accepted the request | Submission is confirmed, delivery is not | Stop sending; wait for delivery evidence or run the normal status check. |

The classification must be based on the provider's documented behavior. A generic HTTP status alone is not enough if the provider may process a request before returning an error.

## A concrete budget

For a confirmed retryable failure, use an example limit of **four provider attempts per delivery job within 30 minutes**. The first attempt counts. Store `attempt_count`, `next_attempt_at`, `last_outcome`, and a stable request identity with the durable job; a process restart must not reset the budget. Reuse a provider idempotency key when the provider supports one. Without that guarantee, retry only outcomes the provider explicitly confirms were not accepted. A provider `Retry-After` value takes precedence when present, provided it is valid and within the service's operational limit.

Otherwise, one possible schedule after attempts 1, 2, and 3 is 30 seconds, 2 minutes, and 8 minutes, each with bounded random jitter. These are example values to tune against provider guidance and observed load, not universal defaults. Limit concurrency per provider as well as per job so a backlog cannot create a synchronized burst.

When the budget is exhausted, move the job to `needs_review`. Do not silently reset its count, create a new checkout, or issue a fresh provider request under a new ID. A human or an explicit reconciliation rule should supply new evidence before another send is authorized.

## Transaction boundary

The worker claims one durable job with a lease and records the attempt decision before contacting the provider. On a confirmed response, it commits the resulting state and next schedule atomically. If the process crashes between the external call and that commit, the lease expires into an **uncertain** state; recovery begins with provider lookup, not a blind retry.

```text
claim(job_id)
if job.state == uncertain:
    reconcile(job.provider_request_id)
elif job.attempt_count >= 4 or now >= job.created_at + 30 minutes:
    move_to_needs_review(job)
elif job.state == queued or last_outcome_is_confirmed_retryable(job):
    reserve_attempt(job)  # persist count and request identity
    call_provider(job.provider_request_id)
else:
    do_not_send(job)
```

The pseudocode leaves out locking and provider-specific response mapping. Implement both explicitly; two workers must not reserve the same attempt concurrently.

## Tests that expose unsafe retries

1. Run two workers against the same queued job. Only one may reserve an attempt.
2. Crash the worker after the provider accepts a request but before the local commit. Recovery must reconcile before another send.
3. Return a rate limit with `Retry-After`. The next attempt must respect it and the per-provider concurrency cap.
4. Return a permanent rejection. The attempt count must stop increasing.
5. Restart workers after three attempts. The fourth is the last automatic attempt; a restart cannot grant a new budget.
6. Exhaust the budget with no conclusive delivery evidence. The customer status should remain honest about uncertainty, and the support queue should contain the evidence needed to investigate.

A retry budget limits load. Outcome classification and reconciliation protect against duplicate delivery. Both are required for a safe worker.
