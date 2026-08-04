# Monitoring and Troubleshooting Reference

Contents: [MPL statuses](#message-statuses) · [Log levels](#log-levels) · [Trace](#using-trace) · [Diagnostic method](#diagnostic-method) · [Error decoder](#error-decoder) · [Stores](#store-monitoring) · [OData APIs](#monitoring-apis) · [Performance](#performance-investigation)

## Message statuses

| Status | Meaning |
|---|---|
| Completed | Processed successfully to an End Message event |
| Failed | Ended in an Error End Event or an unhandled exception |
| Retry | Awaiting redelivery (JMS/XI) |
| Escalated | Error occurred and was handled, but flagged for attention |
| Processing | Currently running |
| Discarded | Dropped without processing |
| Cancelled | Manually cancelled |
| Abandoned | Processing stopped abnormally, e.g. node restart |

Escalated deserves attention in reviews: a high Escalated count often means the exception subprocess is catching something routine that should be handled explicitly instead.

## Log levels

Set per iFlow in Monitor → Manage Integration Content, or at deployment.

| Level | Captures |
|---|---|
| None | Minimal, no MPL detail |
| Info | Default — MPL header, custom properties, step-level status |
| Debug | Adds more detail and script logging |
| Trace | Full exchange payload before and after every step |

Trace is the tool for "the payload isn't what I expect at step 7", and nothing else gives you that. It is also expensive: it captures every payload in full, so it is time-limited by design — it resets itself after a short window (around 10 minutes) and the captured data is retained for roughly an hour. Treat those as approximate and check current tenant behaviour.

Never leave Trace on in production. Beyond performance, it writes full payloads into monitoring where anyone with monitoring authorization can read them — a real data protection exposure if the payload contains personal data.

## Using trace

1. Set the flow's log level to Trace.
2. Trigger the message (resend, or wait for the next poll).
3. Open the MPL entry → the trace view shows each step with the message before and after.
4. Reset the log level.

What to look for, in order:
- Where does the body stop looking like what you expect? That's your failing step, not necessarily the step that threw.
- Are the properties you expect actually set at the point they're read?
- Are headers present that should have been removed, or absent that should have been added?

The step that throws the exception is frequently downstream of the step that caused the problem. Trace lets you find the actual origin instead of debugging the symptom.

## Diagnostic method

A repeatable order of investigation that avoids the common trap of guessing:

1. **Read the actual error text**, in full, including the causal chain. The root cause is usually in the last `Caused by:` clause, not the first line.
2. **Determine which side failed.** Did the flow fail internally, or did the receiver return an error? An HTTP status in the message means the call reached the receiver — that's a different investigation from a connection failure.
3. **Check whether it ever worked.** New interface → configuration or design. Previously working → something changed: certificate expiry, credential rotation, receiver deployment, payload variation, volume increase.
4. **Check whether it fails for all messages or some.** All → configuration. Some → data-dependent, so diff a failing payload against a working one.
5. **Check the boring things.** Is the flow deployed and started? Is the credential alias deployed *on this tenant*? Is the endpoint the one you think? A surprising share of incidents end here.
6. **Reproduce with Trace** if the cause still isn't clear.

## Error decoder

| Error text | Cause | Fix |
|---|---|---|
| `unable to find valid certification path to requested target` | Receiver's CA chain not in tenant keystore | Add root + intermediates |
| `SSLHandshakeException: handshake_failure` | TLS/cipher mismatch, or client cert expected | Check TLS version; configure client cert |
| `HTTP 401 Unauthorized` (receiver) | Wrong/undeployed credential alias | Verify alias exists on this tenant |
| `HTTP 403 Forbidden` (receiver) | Authorization missing in the backend | Backend role assignment |
| `CSRF token validation failed` | Token not fetched or session not reused | Check OData adapter CSRF setting; some gateways need it disabled |
| `Connection refused` / `No route to host` | Cloud Connector resource not exposed, or mapping missing | Cloud Connector access control |
| `SocketTimeoutException: Read timed out` | Receiver slower than the configured timeout | Increase timeout, or make the call async |
| `MessageBody is not available` / null body | Body consumed by an earlier step (stream read twice) | Read once, `setBody` back |
| `groovy.lang.MissingMethodException` | Wrong type passed to a method, often from `getBody` | Match the body type |
| `java.lang.OutOfMemoryError` / node instability | Large payload materialised in memory | Split earlier, stream, avoid full-body reads |
| `Content is not allowed in prolog` | Non-XML content (JSON, BOM, HTML error page) fed to an XML step | Check what the receiver actually returned |
| `Premature end of file` | Empty body reaching an XML parser | Guard for empty response |
| `Cannot cast object 'null' ...` | Property/header read before it was set | Check order; use safe navigation |
| `No route to path` / 404 on your own endpoint | Flow not deployed, wrong address, or wrong tenant URL | Check Manage Integration Content |
| `Bad Request` from an OData receiver | Malformed `$filter`, wrong key type, missing mandatory field | Test the URL directly against the service |
| `deployment failed: <artifact> ... in status ERROR` | Missing dependency, e.g. a script collection or referenced mapping | Check the deployment error detail |
| Message stuck in `Processing` | Long-running step, or a lock held after a node event | Check queues and locks; may need manual cancellation |
| Duplicate messages after a restart | Non-idempotent processing with at-least-once delivery | Add idempotency — see `error-handling.md` |

## Store monitoring

Under Monitor → Manage Stores:

- **Message Queues (JMS)** — depth, retry counts, dead-lettered messages. A growing queue depth is an early warning that the consumer can't keep up; a dead-letter count above zero means messages are being lost unless someone acts.
- **Data Stores** — entries, expiry, size. Data Stores that only grow are a leak: something writes and nothing deletes.
- **Variables** — current values, useful for confirming a last-run timestamp is what you think.
- **Number Ranges** — current values.

Check queue depth and dead-letter counts as part of any handover or health check. They're the metrics most likely to be silently unhealthy.

## Monitoring APIs

The tenant exposes an OData API for programmatic monitoring — the basis for external dashboards and alerting.

```
GET /api/v1/MessageProcessingLogs?$filter=Status eq 'FAILED' and LogEnd gt datetime'2026-08-01T00:00:00'
GET /api/v1/MessageProcessingLogs('<MPL_ID>')/CustomHeaderProperties
GET /api/v1/MessageProcessingLogs('<MPL_ID>')/ErrorInformation/$value
GET /api/v1/IntegrationRuntimeArtifacts
GET /api/v1/MessageStoreEntries
```

Design-time and deployment operations use a parallel API set:

```
GET  /api/v1/IntegrationPackages
GET  /api/v1/IntegrationDesigntimeArtifacts(Id='<id>',Version='<ver>')
POST /api/v1/DeployIntegrationDesigntimeArtifact?Id='<id>'&Version='<ver>'
```

Authenticate with OAuth client credentials from a service key on the process integration runtime service. Write operations need a CSRF token: `GET` any resource with header `X-CSRF-Token: Fetch`, then send the returned token on the `POST`.

A scheduled job polling for FAILED and ESCALATED messages and pushing them to a ticketing system is a common and worthwhile addition to any production landscape — it's what turns "the dashboard shows failures if someone looks" into actual alerting.

## Performance investigation

When a flow is slower than expected, the usual causes in rough order of frequency:

1. **Receiver latency** — the flow is fine, the target is slow. Trace timings show this immediately.
2. **Per-item calls inside a splitter** — 5,000 splits × one lookup each. Batch the lookup, or pre-load into a property.
3. **Sequential processing where parallel would do** — splitter parallel processing off. Turn it on if ordering doesn't matter and the receiver can take it.
4. **Large payload materialisation** — repeated `getBody(String)` on a 50 MB file across several scripts.
5. **Oversized OData queries** — `$expand` pulling associations you don't use, or no `$select`.
6. **Trace left enabled.**
7. **Node contention** — another flow on the same node consuming resources.

Step-level durations in the MPL are the fastest way to localise it. Establish where the time actually goes before optimising anything.
