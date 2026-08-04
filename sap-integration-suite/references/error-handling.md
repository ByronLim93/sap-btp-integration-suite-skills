# Error Handling and Resilience Reference

Contents: [Classify first](#classify-the-error-first) · [Exception subprocess](#exception-subprocess) · [End events](#end-events-and-message-status) · [Retry](#retry-strategies) · [Idempotency](#idempotency) · [Quality of service](#quality-of-service) · [Alerting](#alerting) · [Reprocessing](#reprocessing) · [Sync vs async](#sync-vs-async-error-behaviour)

## Classify the error first

Most badly-designed error handling comes from treating all failures identically. Three categories, three different responses:

**Technical / transient** — receiver down, timeout, network, 503, connection reset. Retry is appropriate and usually sufficient.

**Technical / permanent** — malformed payload, mapping failure, authentication failure, 400/401/403. Retry will never succeed; it just delays the alert and burns capacity. Fail fast and notify.

**Business** — the data is well-formed but rejected by the receiver's business logic (material doesn't exist, order already closed, credit check failed). Retry is wrong. This usually needs to go back to a business user, not to support.

A common failure of design is routing all three into the same dead-letter queue where nobody distinguishes them. Decide, per interface, what should happen to each — and write it into the interface spec.

## Exception subprocess

One per process scope. A main Integration Process can have one; each Local Integration Process can have its own, which is how you get differentiated handling within a single flow.

Available inside the exception subprocess:

- `${exception.message}` — the message text
- `${exception.stacktrace}` — full stack trace
- `CamelExceptionCaught` — the exception object itself (property), for typed inspection in Groovy
- All properties set before the failure; the body is the payload as it was at the point of failure, which may be mid-transformation and not what you expect

Typical structure:

```
Exception Subprocess
  → Content Modifier: capture ${exception.message} into property errorText
  → Groovy: classify the exception, set errorCategory, log with correlation ID
  → Router: business error / technical error
      → business: write to Data Store for business review, Error End Event
      → technical: build alert payload, Send to Mail/ticketing, Escalation End Event
```

Capturing the original inbound payload into a property at the *start* of the flow is worth doing on anything you'll need to reprocess — by the time you reach the exception handler, the body has usually been transformed past usefulness.

Typed classification in Groovy:

```groovy
def ex = message.getProperty("CamelExceptionCaught") as Exception
def category = "TECHNICAL_PERMANENT"
if (ex instanceof java.net.SocketTimeoutException ||
    ex instanceof java.net.ConnectException) {
    category = "TECHNICAL_TRANSIENT"
}
def status = message.getHeaders().get("CamelHttpResponseCode")
if (status in [500, 502, 503, 504]) category = "TECHNICAL_TRANSIENT"
if (status in [400, 401, 403, 404, 422]) category = "TECHNICAL_PERMANENT"
message.setProperty("errorCategory", category)
```

## End events and message status

| End event | Resulting MPL status | Use when |
|---|---|---|
| End Message | Completed | Handled successfully, nothing more to do |
| Error End Event | Failed | The message genuinely failed and should show as failed |
| Escalation End Event | Escalated | Handled the error, but flag it as needing attention |
| Terminate Message | Completed (no further processing) | Deliberate stop, e.g. filtered out |

Getting this wrong distorts monitoring. A flow that catches every error and ends with End Message shows 100% Completed while quietly dropping messages — the monitoring dashboard becomes actively misleading. Escalated is the right status for "we handled it but a human should know".

## Retry strategies

**Adapter-level retry** — some receiver adapters have built-in retry settings. Simplest option when it fits.

**JMS-based retry** — the standard pattern. The consuming flow reads from a queue; on exception, the message is returned to the queue and retried with exponential backoff up to a configured maximum, then moved to dead-letter status visible in Monitor → Manage Message Queues. This is the most robust general-purpose retry mechanism because state survives node restarts.

**Data Store polling retry** — write failed messages to a Data Store with a retry counter, and have a scheduled flow pick them up. More work than JMS, but gives full control over the retry schedule and lets you inspect the queue as data.

**JMS retry design points:**
- Set the max retry count deliberately. Infinite retry against a permanently broken receiver fills the queue.
- Only retry transient errors. Route permanent failures to dead-letter immediately rather than retrying twelve times first.
- Backoff should account for the receiver's recovery time — retrying every 5 seconds against a system doing a 30-minute maintenance window is pointless load.

**Do not retry non-idempotent operations without dedup.** See below.

## Idempotency

The question to ask on every async interface: *what happens if this message is delivered twice?*

Duplicate delivery happens through retry after ambiguous failure (the call timed out — did the receiver process it?), node restart mid-processing, and sender resends. This is not an edge case.

Options, best first:

**1. The receiver is idempotent.** An upsert keyed on a business ID, or an API that accepts an idempotency key. Verify this rather than assuming it — "our API is idempotent" is frequently untrue in practice.

**2. Idempotent Process Call.** Built-in step: give it a message ID expression, and it detects and suppresses duplicates against a persistent store. The right tool for most cases.

**3. Data Store deduplication.** Write the message ID with `overwrite = false`; a duplicate key throws, which you catch and treat as a duplicate. More manual but transparent.

**4. Query-before-write.** Check the receiver for the record before creating it. Racy under parallelism, and doubles the call volume. Last resort.

Pick the ID carefully. It must be stable across retries and unique per business event — a UUID generated inside the flow is regenerated on retry and therefore useless. Use the sender's document number, or a hash of the payload.

## Quality of service

- **Best effort** — synchronous, no persistence, no retry. Failure is immediately visible to the caller.
- **Exactly once (EO)** — asynchronous with persistence and duplicate prevention. JMS or XI adapter.
- **Exactly once in order (EOIO)** — as EO, plus ordering by a sequence key. Requires single-threaded consumption; throughput is bounded by the slowest message. Only design for this when ordering genuinely matters — it's frequently requested and rarely necessary.

If EOIO is requested, ask what breaks if messages arrive out of order. Often the answer reveals that per-key ordering (all messages for one customer, not all messages globally) is what's actually needed, which is far cheaper.

## Alerting

An interface with no alerting path is an interface that fails silently until a business user notices missing data weeks later. Options:

- **Cloud ALM** — SAP's monitoring solution; integrates with Integration Suite for exception monitoring and alerting. The strategic answer if the customer has it.
- **Mail receiver in the exception subprocess** — simplest, works everywhere. Keep the payload out of the mail body if it contains sensitive data; send the MPL ID instead so support can look it up.
- **Ticketing integration** — POST to ServiceNow/Jira from the exception subprocess. Include the MPL ID, interface name, error category and business key.
- **Monitoring API polling** — an external tool queries the MPL OData API for Failed/Escalated messages on a schedule. Good for consolidated dashboards. See `monitoring-troubleshooting.md`.

Whatever the channel, the alert must contain the MPL ID and a business key. An alert saying "Interface X failed" without either sends support hunting through monitoring for the right message.

Deduplicate alerts. A receiver outage generating 4,000 individual emails trains everyone to filter them.

## Reprocessing

Design the reprocessing path before go-live, not during the first incident:

- **JMS dead letter** — messages can be moved back for reprocessing from Manage Message Queues.
- **Data Store staging** — a reprocessing flow reads pending entries and reruns them. Give it a manual trigger endpoint.
- **Source resend** — sometimes the cleanest answer is "ask the sender to resend", but only if the sender can, and only if the receiver tolerates duplicates.

Document which applies, and who is authorised to do it. Reprocessing usually requires production authorization the on-call person may not have.

## Sync vs async error behaviour

**Synchronous:** the caller is waiting. Return a meaningful HTTP status and a structured error body. Don't return 200 with an error inside unless the API contract says so. Map internal errors to sensible codes — 400 for bad request, 502 for downstream failure, 504 for downstream timeout. Set `CamelHttpResponseCode` in the exception subprocess.

**Asynchronous:** nobody is waiting, so the error must be persisted and surfaced. This is where status choice, alerting and reprocessing matter most. The sender has already been told 202 Accepted, and that promise needs backing.

A frequent design error: an async interface that returns 202, then fails, then does nothing. The sender believes it succeeded. If you accept a message, you own it.
