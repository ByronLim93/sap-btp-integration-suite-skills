# iFlow Design Reference

Contents: [Runtime model](#runtime-model) · [Exchange model](#exchange-model) · [Step selection](#step-selection) · [Simple expressions](#simple-expression-language) · [Adapters](#adapter-notes) · [Splitting](#splitting-and-aggregation) · [Decoupling patterns](#decoupling-patterns) · [Reuse](#reuse-across-iflows) · [Storage](#storage-options) · [Anti-patterns](#anti-patterns)

## Runtime model

Cloud Integration runs on Apache Camel. Almost every unexplained behaviour makes sense once you remember that: an iFlow is a Camel route, steps are processors, and the message travelling through is a Camel Exchange. Terminology in the Web UI is BPMN-flavoured, but the semantics are Camel's.

Each tenant runs one or more runtime nodes. Flows deploy independently; a deployment failure on one flow doesn't take down others, but a runaway flow consuming memory absolutely affects its neighbours on the same node.

## Exchange model

Three places to hold data, and choosing wrong causes most of the "where did my value go" problems:

**Body** — the payload. Single value, replaced by most steps. Read it once and stash it if you need it repeatedly.

**Headers** — propagated *out* to receiver adapters. An HTTP receiver call sends exchange headers as HTTP headers. Useful for things like `Content-Type`, custom API keys the target expects, or SOAPAction. Two rules: headers must be listed in Runtime Configuration → Allowed Header(s) to survive into the outbound call, and never put anything internal or sensitive here.

**Properties** — internal to the exchange, never leave the flow. This is the default choice for state: correlation IDs, loop counters, flags, cached values, original payload copies.

Reserved things worth knowing:

| Name | Type | Meaning |
|---|---|---|
| `SAP_MessageProcessingLogID` | header | The MPL ID for this run |
| `SAP_MessageProcessingLogCustomHeaderProperty` | — | Custom search attributes shown in monitoring |
| `SAP_Sender` / `SAP_Receiver` | property | Sender/receiver shown in MPL |
| `CamelExceptionCaught` | property | The exception object, available in exception subprocess |
| `CamelHttpResponseCode` | header | Sets HTTP status of the reply (sender side); read the receiver's status (after Request-Reply) |
| `CamelHttpUri` / `CamelHttpPath` / `CamelHttpQuery` | header | Override the receiver URL at runtime |
| `CamelFileName` | header | Filename for SFTP/file receivers |
| `CamelSplitIndex` / `CamelSplitSize` / `CamelSplitComplete` | header | Splitter context |
| `SAP_ApplicationID` | property | Business key surfaced in MPL search — set this, it makes support far easier |

Setting `SAP_ApplicationID` to something meaningful (order number, employee ID, invoice number) is one of the highest-value five-minute additions to any flow. It makes messages findable in monitoring by business key instead of timestamp guessing.

## Step selection

**Content Modifier** — set body, headers, properties. Prefer it over Groovy for anything static or simple XPath. It is faster, visible in the model, and doesn't need a code review.

**Router** — conditional branching on XPath or Simple expression. Always define a Default route; a router with no matching condition and no default throws at runtime.

**Message Mapping** — graphical mapping, the default for structural transformation. See `mapping.md`.

**Script (Groovy / JavaScript)** — use Groovy; JavaScript support is legacy and slower. Reach for it when logic genuinely can't be expressed declaratively. See `groovy.md`.

**Request-Reply** — synchronous call to a receiver; response becomes the body. The receiver's fault/error response and HTTP status are available afterward.

**Send** — fire-and-forget to a receiver. No response.

**Content Enricher** — combines the current message with a lookup result (Combine or Enrich mode). Enrich mode merges by key path; genuinely useful but fiddly. If the merge logic is non-trivial, a Request-Reply into a property plus a mapping with two sources is easier to maintain.

**Process Call** — invoke a Local Integration Process. Use to structure large flows and to scope exception handling — a Local Integration Process can have its own exception subprocess, which is the only way to have more than one error handler in a flow.

**Filter** — XPath-based pruning. Rarely the right tool; mapping is usually clearer.

**Converters** — XML↔JSON, CSV↔XML, EDI. See `mapping.md` for the conversion gotchas.

**Data Store / Write, Get, Select, Delete** — tenant-local persistence, see [Storage options](#storage-options).

**Idempotent Process Call** — built-in duplicate detection against a store of seen IDs. Preferable to hand-rolling deduplication with a Data Store.

**Aggregator** — collects related messages into one. Requires a correlation expression and a completion condition; needs a JMS-backed queue in most configurations.

**Poll Enrich** — pull a message from a source (typically SFTP) mid-flow rather than at the sender.

## Simple Expression Language

Used in Content Modifiers, router conditions, and adapter fields:

```
${property.orderNumber}
${header.CamelHttpResponseCode}
${in.body}
${date:now:yyyyMMdd-HHmmss}
${date:now+1h:yyyy-MM-dd}
${exchangeId}
${camelId}
${property.count} > 0
${header.Status} contains 'ERR'
${property.list.size()} == 0
${bean:myBean}
```

XPath is also available in router conditions and content modifiers — `/Order/Header/Type = 'STD'`. XPath evaluates against the body, so it requires the body to be XML at that point in the flow, which is a frequent cause of "router always takes default" after a JSON payload arrives unconverted.

## Adapter notes

Practical points that cause the most trouble, per adapter:

**HTTPS (sender)** — endpoint address is a path appended to the tenant's runtime URL. Requires a role for authorization (default `ESBMessaging.send`); create a dedicated role and service key rather than reusing the default in production. `CSRF Protected` should be off for non-browser clients.

**HTTP (receiver)** — set the timeout explicitly. Default behaviour on 4xx/5xx is to throw; enable "Throw Exception on Failure" = off if you want to inspect the status code yourself, then read `CamelHttpResponseCode`.

**OData (V2/V4)** — the operation modeler generates the query; check the generated `$select` and `$expand` because over-selecting is a common performance problem on large entity sets. For writes, CSRF token handling is automatic in most versions but fails against some gateways — symptom is HTTP 403 with "CSRF token validation failed". Paging: enable it and process pages via a loop rather than pulling everything at once.

**SOAP** — pick the right message protocol (SOAP 1.x vs. SAP RM). SAP RM is for reliable messaging with SAP backends. WS-Security config lives on the adapter, not in a script.

**IDoc** — receiver typically over HTTP to the backend's IDoc inbound. The payload must be IDoc XML matching the segment structure exactly, including `BEGIN` attributes. IDoc control record fields drive routing in the backend; get `SNDPRN`/`RCVPRN`/`MESTYP`/`IDOCTYP` right or the IDoc lands in status 56.

**SFTP** — key-based auth needs the host key in the tenant's known hosts file and the private key in the keystore. Use `Automatically Disconnect` for infrequent polls. For "process each file once", set Post-Processing to move or delete; idle-based reading (`Change Detection`) avoids picking up half-written files. Long-running polls holding a lock are a classic cause of duplicate processing after a node restart.

**JMS** — tenant-internal queuing, the backbone of asynchronous and retry designs. Queues are a limited tenant resource, so plan queue count deliberately. Consumer flows have a configurable retry with exponential backoff, then messages move to a dead-letter state visible in Manage Message Queues.

**ProcessDirect** — in-memory call between iFlows on the same tenant. Fast, no network hop, no persistence. Excellent for reusable sub-flows (common error handler, token fetcher, logging). Note it's synchronous and shares the caller's transaction — an exception in the callee propagates to the caller.

**XI** — for connecting to SAP backends using the XI protocol, including from PI/PO migration scenarios. Supports EO/EOIO quality of service.

**Mail** — receiver for notifications. Fine for alerting, poor as an integration channel.

**SuccessFactors / Ariba / Workday / OpenConnectors** — pre-built application adapters; check the SAP Note for supported entity versions before promising a field is available.

## Splitting and aggregation

**General Splitter** — splits at a node path, streaming-capable, each split runs independently. Set `Parallel Processing` deliberately: it improves throughput but destroys ordering and multiplies load on the receiver. Number of concurrent processes should account for the receiver's capacity, not just yours.

**Iterating Splitter** — keeps the parent context (envelope) on each split. Use when the target needs the header structure repeated per item.

Splitter + Gather + Join is the aggregation path within a single flow. The Aggregator step is for correlating *across* messages and requires persistence.

Error handling inside splits deserves explicit thought: by default one failed split fails the whole message. If partial success is acceptable, wrap the per-item processing in a Local Integration Process with its own exception subprocess, collect errors into a property, and report at the end.

## Decoupling patterns

**Sender decoupling (async intake)** — sender iFlow does minimal validation, writes to JMS, returns 202 immediately. A second iFlow consumes the queue and does the real work. The sender is no longer coupled to receiver availability, and retry becomes free.

**Receiver decoupling** — Request-Reply with retry wrapped in a JMS consumer, so a temporarily unavailable target doesn't lose messages.

**Aggregation / batching** — collect items over a window into one call, for targets that penalise chattiness. Data Store or JMS + Aggregator.

**Content-based routing hub** — one entry flow, ProcessDirect out to per-receiver flows. Adding a receiver becomes a new flow rather than a change to a critical shared one.

## Reuse across iFlows

Reuse mechanisms, in rough order of preference:

1. **ProcessDirect sub-flows** — for logic (error handling, token retrieval, canonical logging)
2. **Script collections** — packaged Groovy/JS shared across flows in a package; edit once, redeploy consumers
3. **Message mapping in a package** — reference a mapping artifact from multiple flows
4. **Partner Directory** — externalized per-partner configuration (IDs, parameters, mappings) retrievable at runtime; the right answer when you'd otherwise clone a flow per partner
5. **Value mappings** — code list lookups, maintainable without redeployment

Cloning an iFlow per trading partner is the most common structural mistake in B2B scenarios. Twenty near-identical flows means twenty places to apply the next fix.

## Storage options

| Mechanism | Scope | Typical use | Notes |
|---|---|---|---|
| Data Store | Tenant, per flow or global | Staging, dedup, batch collection | Entries have an expiry; visible under Monitor → Data Stores |
| Variables (local/global) | Tenant | Last-run timestamps, counters, tokens | Long expiry if untouched; not for high-frequency writes |
| JMS queues | Tenant | Async decoupling, retry | Limited number per tenant — plan them |
| Number ranges | Tenant | Sequential IDs | Configured in the Monitor/Settings area |
| Partner Directory | Tenant | Per-partner config and payloads | Accessed via API or Groovy |

None of these are a database. Data Store is not for reporting, high-volume lookups, or anything you'd want to query flexibly. If the design starts needing joins, the data belongs somewhere else.

## Anti-patterns

- **Groovy as the whole flow.** One script that does mapping, HTTP calls and logging. Untestable, invisible in trace, and it defeats the platform.
- **Default step names.** `Content Modifier 7` in a production incident trace.
- **Hardcoded endpoints.** Blocks transport.
- **`try/catch` swallowing exceptions in Groovy.** The flow reports success while doing nothing. See `groovy.md`.
- **No exception subprocess.** Errors surface as raw stack traces, and the sender gets an unhelpful 500.
- **Payload attachments logged unconditionally.** Privacy exposure and MPL bloat.
- **Synchronous chains three flows deep.** Timeouts multiply and the failure mode is opaque.
- **Polling every minute for data that changes daily.** Costs, load, and rate limits on the source.
