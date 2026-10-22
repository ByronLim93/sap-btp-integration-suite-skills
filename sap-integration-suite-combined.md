# SAP Integration Suite — Combined Skill Reference

> Single-document extract of the `sap-integration-suite` Claude Agent Skill (13 source files combined). Generated 2026-10-22.
> The authoritative source remains the individual files under `sap-integration-suite/` — this is a readable, printable compilation.

## Contents

1. [Skill Overview](#1-skill-overview) — Routing table, design principles, naming, standard skeleton, review checklist.
2. [iFlow Design](#2-iflow-design) — Runtime/exchange model, step selection, Simple expressions, adapters, splitting, decoupling, storage, anti-patterns.
3. [Groovy Scripting](#3-groovy-scripting) — Script signature, Message API, MPL logging, platform services, recipes, pitfalls, performance, testing.
4. [Mapping (Message Mapping & XSLT)](#4-mapping-message-mapping-xslt) — Choosing an approach, contexts, node/standard/custom functions, XSLT, format conversion, value mappings.
5. [B2B and EDI](#5-b2b-and-edi) — EDIFACT/X12 structure, EDI steps, control numbers, the acknowledgement ladder, AS2/AS4, SFTP/OFTP, inbound/outbound patterns.
6. [Trading Partner Management & Integration Advisor](#6-trading-partner-management-integration-advisor) — When to use TPM, object model, type systems, MIG/MAG, agreements, runtime lookup, Partner Directory, onboarding.
7. [Error Handling & Resilience](#7-error-handling-resilience) — Classifying errors, exception subprocess, end events, retry, idempotency, QoS, alerting, reprocessing, sync vs async.
8. [Security & Connectivity](#8-security-connectivity) — Security material, inbound/outbound auth, OAuth flows, certificates/TLS, Cloud Connector, principal propagation, message-level security.
9. [API Management](#9-api-management) — Object model, flow structure, policy catalogue and examples, variables, fault handling, design guidance.
10. [Monitoring & Troubleshooting](#10-monitoring-troubleshooting) — Message statuses, log levels, trace, diagnostic method, error decoder, store monitoring, OData APIs, performance.
11. [CI/CD & Transport](#11-cicd-transport) — Tenant landscape, transport options, what doesn't transport, configure-only pattern, Git, pipelines, practices.
12. [Asset: iFlow Specification Template](#12-asset-iflow-specification-template) — Technical interface specification template for design documents and handover.
13. [Asset: B2B Partner Onboarding Checklist](#13-asset-b2b-partner-onboarding-checklist) — End-to-end trading partner onboarding checklist, discovery through cutover.


---

## 1. Skill Overview

*Source: `sap-integration-suite/SKILL.md`*

> **Skill description:** Expert guidance for SAP Integration Suite — Cloud Integration (CPI) iFlows, Groovy, message mappings, XSLT, adapters, B2B/EDI with Trading Partner Management, API Management, security material, monitoring and CI/CD transport. Use whenever the user mentions SAP CPI, Cloud Integration, iFlow, Integration Suite, API Management, SAP BTP integration, message mapping, Groovy for SAP, MPL, ProcessDirect, IDoc/OData/SFTP/SuccessFactors adapters, or exception subprocess. Use it just as strongly for B2B and EDI work — EDIFACT, ANSI X12, EANCOM, ODETTE, AS2, AS4, MDN, CONTRL, 997/999, ORDERS, DESADV, INVOIC, 850/855/856/810, EDI splitter, control numbers, Trading Partner Management, TPM, trading partner agreements, Integration Advisor, MIG, MAG, Partner Directory, partner onboarding, VAN connectivity — even when SAP is not named, since EDI questions here almost always concern Integration Suite. Also use for interface specs, error-handling design, and transporting artifacts between DEV/QA/PRD tenants.

### SAP Integration Suite

Guidance for designing, implementing, reviewing and troubleshooting integrations on SAP Integration Suite (SAP BTP). The focus is Cloud Integration (formerly CPI), with supporting material for API Management and the surrounding lifecycle.

#### How to use this skill

Read the routing table below and load only the reference files you actually need. Each is self-contained.

| The task involves | Read |
|---|---|
| iFlow structure, step selection, adapters, headers vs. properties, externalization | `references/iflow-design.md` |
| Writing or reviewing Groovy scripts | `references/groovy.md` |
| Message Mapping, XSLT, JSON↔XML, node functions | `references/mapping.md` |
| EDIFACT/X12 structure, EDI steps, AS2/AS4, acknowledgements, control numbers | `references/b2b-edi.md` |
| Trading Partner Management, agreements, MIG/MAG, Partner Directory, onboarding | `references/tpm-integration-advisor.md` |
| Exception subprocesses, retry, idempotency, alerting, EO/EOIO | `references/error-handling.md` |
| Authentication, credentials, keystore, Cloud Connector, principal propagation | `references/security-connectivity.md` |
| API proxies, policies, rate limiting, API products | `references/api-management.md` |
| MPL analysis, trace, interpreting runtime errors | `references/monitoring-troubleshooting.md` |
| Transports, Git, MTAR, pipelines, tenant landscape | `references/cicd-transport.md` |

Two assets are available as starting artifacts:

- `assets/iflow-spec-template.md` — technical interface specification, for design documents and handover
- `assets/b2b-partner-onboarding-checklist.md` — end-to-end trading partner onboarding checklist, from discovery through cutover

#### What you can and can't produce

You cannot build a deployable `.iflw` artifact from scratch reliably — the underlying BPMN2 XML is verbose, version-specific and easily corrupted. Be honest about that boundary rather than emitting plausible-looking XML the user will waste an afternoon on.

What you *can* do well, and should offer:

- **Step-by-step build instructions** for the Web UI — palette element, tab, field, exact value. This is what most users actually want.
- **Groovy scripts, XSLT, mapping logic, policy XML, Camel Simple expressions** — real, deployable code.
- **Design and review** — pattern selection, error-handling strategy, code review against the checklist below.
- **Troubleshooting** — read an MPL error, name the likely cause, give the fix.
- **Documentation** — interface specs, runbooks, mapping sheets.

If the user has exported artifacts (`.zip` of an iFlow, `.iflw`, `.mmap`, `.xsl`, `.groovy`, `.wsdl`, `.edmx`), read them from disk. An iFlow ZIP unpacks to `src/main/resources/` containing `scenarioflows/integrationflow/*.iflw` (BPMN2 XML), `mapping/`, `script/`, `xsd/`, plus `META-INF/MANIFEST.MF` and `resources.cnt`. Parsing that XML to *review* an existing flow is realistic and useful — generating one is not.

#### Core design principles

These come up in nearly every review. Apply them proactively rather than waiting to be asked.

**Properties for internal state, headers only when they must leave the flow.** Headers are propagated to receiver adapters as HTTP/protocol headers. Anything internal — flags, counters, correlation keys — belongs in exchange properties. If a header genuinely must reach the receiver, it also has to be listed under Runtime Configuration → Allowed Header(s); this silently drops headers otherwise and is the single most common "my header disappeared" ticket.

**Externalize everything environment-specific.** Hosts, credential aliases, queue names, directories, addresses. Use `{{ParameterName}}`. A flow with a hardcoded QA hostname cannot be transported, and finding that out during a go-live cutover is a bad time.

**Never log or attach payloads containing personal or credential data.** Attachments in the MPL are visible to anyone with monitoring authorization and are retained for days. Log identifiers and counts, not the full payload — or gate payload logging behind an externalized `LogLevel` parameter that is `NONE` in production.

**Stream where possible, buffer only when necessary.** Reading the body as a String materialises the whole message in memory. On large payloads use streaming-capable steps, split early with the General Splitter, and avoid multiple Groovy steps that each re-read the body.

**One flow, one responsibility.** Prefer several small iFlows connected by ProcessDirect over one flow with a dozen branches. Smaller flows deploy independently, trace more cleanly, and let you reuse common logic (logging, error handling, token retrieval).

**Design for redelivery from the start.** Ask early: what happens if this runs twice? Most consumers are not idempotent. See `references/error-handling.md`.

#### Naming conventions

Consistency matters more than the specific scheme, but if the user has none, propose this:

- Package: `<Domain>_<Purpose>` — e.g. `Finance_VendorMasterReplication`
- iFlow: `<Sender>_to_<Receiver>_<BusinessObject>_<Direction>` — e.g. `S4HANA_to_Ariba_PurchaseOrder_Out`
- Steps: verb-first and descriptive — `Map PO to cXML`, `Enrich with Vendor Data`, not `Groovy Script 1`, `Content Modifier 3`
- Externalized parameters: `<Scope>_<Name>` — `Receiver_Ariba_URL`, `Sender_Credential_Alias`
- Properties: `prop_` prefix or camelCase, consistently — makes them greppable in trace
- Credential aliases: `<ENV>_<System>_<Type>` — `PRD_ARIBA_OAUTH`

Default step names are a real cost: an MPL trace full of "Content Modifier 4" is unreadable at 2am during an incident.

#### Standard iFlow skeleton

Most synchronous request/reply flows follow this shape. Start here and remove what isn't needed:

```
Sender (HTTPS/SOAP)
  → Content Modifier: capture correlation ID, set properties from headers
  → Groovy: validate + log inbound (payload attachment gated by LogLevel)
  → Router: valid / invalid
  → Message Mapping: source → target
  → Request-Reply: Receiver adapter (OData/SOAP/HTTP)
  → Groovy: interpret response, set status property
  → Content Modifier: build caller response
Exception Subprocess
  → Content Modifier: capture ${exception.message}, CamelExceptionCaught
  → Groovy: log error with correlation ID
  → Error End Event (or Escalation End Event for async)
```

Asynchronous flows replace the direct receiver call with a JMS or Data Store hop so the sender is decoupled from receiver availability, and split into a sender-side and a processing-side iFlow.

#### B2B scenarios

B2B/EDI has its own shape and its own failure modes. When a request involves trading partners, EDI formats or AS2, read `references/b2b-edi.md` and `references/tpm-integration-advisor.md` rather than reasoning from the generic patterns above.

Three things worth establishing early in any B2B conversation, because they change the entire design:

**How many partners, and how fast is that growing?** Under about five stable partners, parameterised iFlows plus Partner Directory are usually enough. Beyond ten, or with regular onboarding, Trading Partner Management pays back the front-loaded setup effort.

**Which acknowledgement do they actually mean?** MDN, CONTRL/997, and an application acknowledgement are three different layers, and "we need confirmation the order was received" doesn't say which. Conflating them produces disputes months later.

**Direct or via a canonical format?** Partner-to-backend mapping is an N×M problem; partner → canonical → backend is N+M. With several partners and more than one backend, the intermediate hop is worth it.

Two habits that prevent most B2B incidents: persist the raw interchange before splitting it, because the original bytes are the only useful evidence in a partner dispute; and separate transport testing from content testing during onboarding, because debugging a certificate problem and a mapping problem at once takes several times longer than doing them in sequence.

#### Review checklist

Run through this whenever asked to review an iFlow, and use it as the acceptance criteria for anything you design:

- [ ] All environment-specific values externalized; no hardcoded hosts, aliases or paths
- [ ] Exception subprocess present, and it distinguishes technical from business errors
- [ ] Steps have meaningful names; the trace would be readable by someone else
- [ ] No payload logging of sensitive data in production log level
- [ ] Headers that must reach the receiver are in Allowed Header(s)
- [ ] Timeouts set explicitly on receiver adapters; not relying on defaults
- [ ] Retry behaviour defined — and safe (idempotent target, or dedup via Data Store / Idempotent Process Call)
- [ ] Large-payload path considered: splitter, streaming, no unnecessary body reads
- [ ] Groovy scripts return `message`, handle null bodies, and don't swallow exceptions
- [ ] Mapping handles missing/empty optional fields without producing invalid target structures
- [ ] Alerting path defined — how does anyone find out this failed?
- [ ] Trace-level logging is not left enabled
- [ ] Credentials use Security Material aliases, never inline values or externalized plaintext passwords

Additionally, for B2B interfaces:

- [ ] Raw interchange persisted before splitting
- [ ] Split-level error behaviour decided — one bad message shouldn't necessarily reject the interchange
- [ ] Control numbers come from a Number Range, scoped per partner and direction
- [ ] Outbound control numbers stored against business keys so acknowledgements can be correlated
- [ ] Every expected acknowledgement has a timeout monitor; missing ones are silent otherwise
- [ ] Negative acknowledgements route to a business owner, not a technical support queue
- [ ] AS2 duplicate detection enabled where the receiver isn't idempotent
- [ ] Partner certificates registered with expiry dates and a named owner
- [ ] Test/production indicator (ISA15 or equivalent) correct for the environment
- [ ] MPL carries partner ID, document number and control number as searchable properties

#### Working style for this domain

The user is usually a consultant or developer under delivery pressure. Give the concrete answer first — the script, the config value, the likely root cause — then the reasoning. Skip the "SAP Integration Suite is a cloud-based platform…" preamble entirely.

When something depends on tenant configuration, release version, or the specific adapter version, say so explicitly rather than guessing. SAP changes adapter behaviour between versions and a confidently wrong field name costs the user more time than an honest "check the adapter version — this moved in 1.6".

If the user describes a requirement, ask about the things that actually change the design and are commonly left unstated: sync or async, expected volume and payload size, error handling expectations (fail fast vs. retry vs. dead letter), and whether the receiver is idempotent. One or two of these, not an interrogation.

---

## 2. iFlow Design

*Source: `sap-integration-suite/references/iflow-design.md`*

### iFlow Design Reference

Contents: [Runtime model](#runtime-model) · [Exchange model](#exchange-model) · [Step selection](#step-selection) · [Simple expressions](#simple-expression-language) · [Adapters](#adapter-notes) · [Splitting](#splitting-and-aggregation) · [Decoupling patterns](#decoupling-patterns) · [Reuse](#reuse-across-iflows) · [Storage](#storage-options) · [Anti-patterns](#anti-patterns)

#### Runtime model

Cloud Integration runs on Apache Camel. Almost every unexplained behaviour makes sense once you remember that: an iFlow is a Camel route, steps are processors, and the message travelling through is a Camel Exchange. Terminology in the Web UI is BPMN-flavoured, but the semantics are Camel's.

Each tenant runs one or more runtime nodes. Flows deploy independently; a deployment failure on one flow doesn't take down others, but a runaway flow consuming memory absolutely affects its neighbours on the same node.

#### Exchange model

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

#### Step selection

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

#### Simple Expression Language

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

#### Adapter notes

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

#### Splitting and aggregation

**General Splitter** — splits at a node path, streaming-capable, each split runs independently. Set `Parallel Processing` deliberately: it improves throughput but destroys ordering and multiplies load on the receiver. Number of concurrent processes should account for the receiver's capacity, not just yours.

**Iterating Splitter** — keeps the parent context (envelope) on each split. Use when the target needs the header structure repeated per item.

Splitter + Gather + Join is the aggregation path within a single flow. The Aggregator step is for correlating *across* messages and requires persistence.

Error handling inside splits deserves explicit thought: by default one failed split fails the whole message. If partial success is acceptable, wrap the per-item processing in a Local Integration Process with its own exception subprocess, collect errors into a property, and report at the end.

#### Decoupling patterns

**Sender decoupling (async intake)** — sender iFlow does minimal validation, writes to JMS, returns 202 immediately. A second iFlow consumes the queue and does the real work. The sender is no longer coupled to receiver availability, and retry becomes free.

**Receiver decoupling** — Request-Reply with retry wrapped in a JMS consumer, so a temporarily unavailable target doesn't lose messages.

**Aggregation / batching** — collect items over a window into one call, for targets that penalise chattiness. Data Store or JMS + Aggregator.

**Content-based routing hub** — one entry flow, ProcessDirect out to per-receiver flows. Adding a receiver becomes a new flow rather than a change to a critical shared one.

#### Reuse across iFlows

Reuse mechanisms, in rough order of preference:

1. **ProcessDirect sub-flows** — for logic (error handling, token retrieval, canonical logging)
2. **Script collections** — packaged Groovy/JS shared across flows in a package; edit once, redeploy consumers
3. **Message mapping in a package** — reference a mapping artifact from multiple flows
4. **Partner Directory** — externalized per-partner configuration (IDs, parameters, mappings) retrievable at runtime; the right answer when you'd otherwise clone a flow per partner
5. **Value mappings** — code list lookups, maintainable without redeployment

Cloning an iFlow per trading partner is the most common structural mistake in B2B scenarios. Twenty near-identical flows means twenty places to apply the next fix.

#### Storage options

| Mechanism | Scope | Typical use | Notes |
|---|---|---|---|
| Data Store | Tenant, per flow or global | Staging, dedup, batch collection | Entries have an expiry; visible under Monitor → Data Stores |
| Variables (local/global) | Tenant | Last-run timestamps, counters, tokens | Long expiry if untouched; not for high-frequency writes |
| JMS queues | Tenant | Async decoupling, retry | Limited number per tenant — plan them |
| Number ranges | Tenant | Sequential IDs | Configured in the Monitor/Settings area |
| Partner Directory | Tenant | Per-partner config and payloads | Accessed via API or Groovy |

None of these are a database. Data Store is not for reporting, high-volume lookups, or anything you'd want to query flexibly. If the design starts needing joins, the data belongs somewhere else.

#### Anti-patterns

- **Groovy as the whole flow.** One script that does mapping, HTTP calls and logging. Untestable, invisible in trace, and it defeats the platform.
- **Default step names.** `Content Modifier 7` in a production incident trace.
- **Hardcoded endpoints.** Blocks transport.
- **`try/catch` swallowing exceptions in Groovy.** The flow reports success while doing nothing. See `groovy.md`.
- **No exception subprocess.** Errors surface as raw stack traces, and the sender gets an unhelpful 500.
- **Payload attachments logged unconditionally.** Privacy exposure and MPL bloat.
- **Synchronous chains three flows deep.** Timeouts multiply and the failure mode is opaque.
- **Polling every minute for data that changes daily.** Costs, load, and rate limits on the source.

---

## 3. Groovy Scripting

*Source: `sap-integration-suite/references/groovy.md`*

### Groovy Scripting Reference

Contents: [Signature](#script-signature) · [Message API](#message-api) · [Logging](#logging-to-the-mpl) · [Platform services](#platform-services) · [Recipes](#recipes) · [Pitfalls](#pitfalls) · [Performance](#performance) · [Testing](#testing)

#### Script signature

Every CPI Groovy script exposes one or more entry-point methods taking and returning a `Message`:

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    // ...
    return message
}
```

The method name is configurable on the Script step (default `processData`). Returning `message` is mandatory — a script that falls off the end returns `null` and the flow fails with an unhelpful NPE downstream.

A second signature exists for message-level scripts vs. exception handling, but `processData` covers virtually all cases.

#### Message API

```groovy
// Body — the type matters, see pitfalls
def body = message.getBody(java.lang.String) as String
def stream = message.getBody(java.io.InputStream)
def bytes = message.getBody(byte[])
message.setBody(newBody)

// Properties (internal state)
def props = message.getProperties()
def orderId = props.get("orderId")
message.setProperty("orderId", "4500001234")

// Headers (propagate to receivers)
def headers = message.getHeaders()
message.setHeader("Content-Type", "application/json")

// Attachments
def attachments = message.getAttachments()
```

`getProperties()` and `getHeaders()` return maps; mutating the returned map works but setting via `setProperty`/`setHeader` is clearer and safer.

#### Logging to the MPL

```groovy
def messageLog = messageLogFactory.getMessageLog(message)
if (messageLog != null) {
    messageLog.setStringProperty("OrderNumber", orderId)          // searchable in MPL
    messageLog.addAttachmentAsString("Request", body, "text/xml") // visible in MPL
}
```

`messageLogFactory` is injected — do not import or instantiate it. The null check matters: it is null when logging is disabled at the current log level, and calling methods on it will throw.

Two rules for production:

- `setStringProperty` for business keys is cheap and hugely valuable — it makes messages findable by order number instead of timestamp.
- `addAttachmentAsString` for payloads is expensive (storage, retention, privacy). Gate it:

```groovy
def logLevel = message.getProperty("LogLevel") ?: "NONE"   // externalized parameter
if (messageLog != null && logLevel in ["DEBUG", "TRACE"]) {
    messageLog.addAttachmentAsString("Payload", body, "application/json")
}
```

#### Platform services

**Secure store (credentials)**

```groovy
import com.sap.it.api.ITApiFactory
import com.sap.it.api.securestore.SecureStoreService
import com.sap.it.api.securestore.UserCredential

def service = ITApiFactory.getService(SecureStoreService.class, null)
def credential = service.getUserCredential("MY_CREDENTIAL_ALIAS")
if (credential == null) {
    throw new IllegalStateException("Credential alias MY_CREDENTIAL_ALIAS not deployed")
}
def user = credential.getUsername()
def pass = new String(credential.getPassword())
```

Use this only when the adapter can't handle the credential itself — adapter-level credential configuration is preferable because it keeps secrets out of scripts entirely.

**Secure parameter (single secret value)**

```groovy
def secret = new String(service.getUserCredential("MY_SECURE_PARAM").getPassword())
```

Secure parameters are stored as a credential with a dummy user; the value lives in the password field.

**Value mapping**

```groovy
import com.sap.it.api.mapping.ValueMappingApi

def api = ValueMappingApi.getInstance()
def target = api.getMappedValue("SourceAgency", "SourceIdentifier", sourceValue,
                                "TargetAgency", "TargetIdentifier")
```

**Data Store**

```groovy
import com.sap.it.api.asdk.datastore.*
import com.sap.it.api.asdk.runtime.*

def service = ITApiFactory.getService(DataStoreService.class, null)
def dataBean = new DataBean(dataAsArray: payload.getBytes("UTF-8"))
def dataConfig = new DataConfig(storeName: "MyStore", id: entryId, overwrite: true)
service.put(dataBean, dataConfig)
```

**Partner Directory** — `com.sap.it.api.pd.PartnerDirectoryService`, for retrieving per-partner parameters and binary content at runtime.

#### Recipes

**Parse and modify XML**

```groovy
def Message processData(Message message) {
    def body = message.getBody(java.lang.String) as String
    def root = new XmlSlurper().parseText(body)
    def orderNo = root.Header.OrderNumber.text()
    message.setProperty("orderNumber", orderNo)
    message.setBody(body)   // unchanged body still needs to be set if you consumed it
    return message
}
```

For building XML, `MarkupBuilder` is fine for small structures, but if you're constructing anything substantial the work belongs in a Message Mapping or XSLT, not here.

**Parse JSON**

```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

def payload = new JsonSlurper().parseText(message.getBody(java.lang.String) as String)
payload.status = "PROCESSED"
message.setBody(JsonOutput.toJson(payload))
```

`JsonSlurper` returns `LazyMap`; if you store it in a property and read it later after the stream is closed, you get surprises. Convert to a concrete structure or re-serialise before stashing.

**Set MPL business key**

```groovy
def Message processData(Message message) {
    def id = message.getProperty("orderNumber") ?: "UNKNOWN"
    message.setProperty("SAP_ApplicationID", id)
    def log = messageLogFactory.getMessageLog(message)
    log?.setStringProperty("OrderNumber", id)
    return message
}
```

**Read response status after a Request-Reply**

```groovy
def status = message.getHeaders().get("CamelHttpResponseCode")
if (status != null && status.toString() != "200") {
    throw new RuntimeException("Receiver returned HTTP ${status}")
}
```

**Build a Basic auth header from a credential** (only when the adapter can't)

```groovy
def cred = ITApiFactory.getService(SecureStoreService.class, null).getUserCredential("ALIAS")
def token = "${cred.getUsername()}:${new String(cred.getPassword())}".bytes.encodeBase64().toString()
message.setHeader("Authorization", "Basic ${token}")
```

Remember `Authorization` must be in Allowed Header(s) to reach the receiver.

**Iterate splits safely**

```groovy
def index = message.getHeaders().get("CamelSplitIndex") as Integer
def isLast = message.getHeaders().get("CamelSplitComplete") as Boolean
```

**Structured error for the exception subprocess**

```groovy
def ex = message.getProperty("CamelExceptionCaught") as Exception
def detail = [
    error      : ex?.getMessage() ?: "Unknown error",
    type       : ex?.getClass()?.getName(),
    orderNumber: message.getProperty("orderNumber"),
    mplId      : message.getHeaders().get("SAP_MessageProcessingLogID")
]
message.setBody(JsonOutput.toJson(detail))
```

#### Pitfalls

**Reading the body as the wrong type.** `getBody(java.lang.String)` on a large binary payload will produce garbage or blow memory. Match the type to what's actually flowing.

**Reading the body twice.** If the body is a stream, the second read returns empty. Read once into a local variable and `setBody` it back if downstream steps need it. This is the single most common Groovy bug in CPI — symptom is an empty payload appearing at a later step for no visible reason.

**Swallowing exceptions.**

```groovy
try {
    riskyThing()
} catch (Exception e) {
    // nothing here
}
```

The flow now reports Completed while having done nothing. If you catch, either handle meaningfully or rethrow. Silent catches are a recurring cause of "the interface says it worked but no data arrived" investigations.

**Assuming `messageLogFactory` is non-null.** It isn't, at lower log levels.

**Using `println`.** Goes nowhere useful. Use the message log.

**Groovy truthiness with `0` and empty strings.** `if (count)` is false when count is 0. Be explicit: `if (count != null)`.

**Date handling.** Use `java.time` (`ZonedDateTime`, `DateTimeFormatter`), not `SimpleDateFormat`, which is not thread-safe and scripts do run concurrently.

**Class name collisions across script collections.** Two scripts defining a class with the same name in the same flow will conflict. Prefix helper classes.

**External libraries.** You can't add arbitrary JARs. Standard JDK, Groovy, and the SAP-provided APIs are what you have. If a requirement needs a specific library, that logic belongs outside CPI.

#### Performance

Scripts run per message, and inefficiency compounds at volume:

- Don't parse the full payload if you only need one value — XPath in a Content Modifier is cheaper.
- Avoid regex on multi-megabyte strings.
- Don't do lookups (HTTP, Data Store) inside a per-item loop when a single batched call would do.
- Compile-once patterns: define `Pattern` as a static field rather than compiling per invocation in a hot path.
- Prefer streaming over string materialisation for large files.

#### Testing

There's no built-in unit test runner in the Web UI. Practical options:

- **Simulation** in the iFlow editor — run a section of the flow with a sample payload without deploying. Good first check.
- **Local testing** — the script API classes can be stubbed. Keep business logic in plain Groovy classes with no `Message` dependency, and have `processData` be a thin adapter. That logic is then testable in any Groovy environment with Spock or JUnit.
- **Trace level** on a deployed flow shows the exchange before and after each step, which is the fastest way to confirm what the script actually received.

Structuring scripts so the testable part is separable is worth the small extra effort on anything with real logic in it.

---

## 4. Mapping (Message Mapping & XSLT)

*Source: `sap-integration-suite/references/mapping.md`*

### Mapping Reference

Contents: [Choosing an approach](#choosing-an-approach) · [Contexts](#contexts-the-core-concept) · [Node functions](#node-functions) · [Standard functions](#standard-functions) · [Custom functions](#custom-functions) · [XSLT](#xslt) · [Format conversion](#format-conversion) · [Value mappings](#value-mappings) · [Common problems](#common-problems)

#### Choosing an approach

| Situation | Use |
|---|---|
| Structural XML→XML transformation, business-readable | Message Mapping |
| Complex recursion, grouping, sorting, or template reuse | XSLT |
| Simple field extraction or a single derived value | Content Modifier with XPath |
| Logic that isn't really mapping (calls, validation, state) | Groovy |
| JSON in, JSON out, no structural change | Groovy or JSON-native handling |

Message Mapping is the default because a functional consultant can read it. Reach for XSLT when the mapping starts needing more than three or four chained node functions to express one relationship — at that point the graphical version has stopped being readable and XSLT is genuinely clearer.

Message Mapping operates on XML. JSON payloads must be converted to XML first (JSON to XML Converter) and usually back afterwards.

#### Contexts — the core concept

Almost every mapping bug traces back to contexts. A source field's values arrive not as a flat list but as a list grouped by the context node — the ancestor element the field is currently associated with.

Consider:

```xml
<Order>
  <Item><Material>A</Material></Item>
  <Item><Material>B</Material></Item>
</Order>
<Order>
  <Item><Material>C</Material></Item>
</Order>
```

With context `Order`, `Material` produces `[A, B]`, `[C]` — two contexts. With context set to the root, it produces `[A, B, C]` — one context. The same mapping produces a completely different target structure depending on which you pick, and the editor gives no warning.

Right-click a source field → **Context** to change it. When a target node repeats the wrong number of times, this is almost always why.

#### Node functions

| Function | Effect | Typical use |
|---|---|---|
| `removeContexts` | Flattens all contexts into one | "I want all values regardless of parent" |
| `splitByValue` | Inserts context changes (each value / on value change) | Creating one target node per source value |
| `collapseContexts` | Keeps first value of each context | Deduplicating per group |
| `useOneAsMany` | Replicates a single value across many | Copying a header field onto every item |
| `sort` / `sortByKey` | Orders values (optionally by a second field) | Grouping items before mapping |
| `mapWithDefault` | Supplies a value when the context is empty | Guaranteeing a target node exists |
| `formatByExample` | Reshapes one list's contexts to match another's | Rebuilding grouping after flattening |
| `exists` | Boolean per context | Conditional target node creation |

The two that unlock most real scenarios: `useOneAsMany` (header-to-item replication) and `sortByKey` + `splitByValue` (grouping items by a key into parent nodes).

**Header onto items:** map `OrderNumber` → `useOneAsMany` with the item-level field as the count driver → target item's `OrderRef`.

**Group items by material:** `sortByKey(Material, Item)` then `splitByValue(value change)` on `Material` to create a group node per distinct material.

#### Standard functions

Grouped as in the editor:

- **Arithmetic** — add, subtract, multiply, divide, round, formatNumber
- **Boolean** — and, or, not, equals, greater, if, ifWithoutElse, ifS
- **Constants** — constant, copyValue, sum, count, index
- **Conversions** — fixValues (code list), value mapping
- **Date** — dateTrans (format conversion), dateBefore, dateAfter, currentDate
- **Text** — substring, concat, replaceString, trim, toUpperCase, length, indexOf
- **Node functions** — as above

`ifWithoutElse` matters more than it looks: with plain `if`, an unmatched condition still produces an empty target node. With `ifWithoutElse`, the node is suppressed entirely. Optional target fields that must be absent rather than empty need this.

`dateTrans` uses Java date patterns. SAP dates arriving as `yyyyMMdd` and targets wanting `yyyy-MM-dd` is the most common instance.

#### Custom functions

Message Mapping supports Groovy custom functions when the standard set falls short:

```groovy
def String buildKey(String company, String doc, MappingContext context) {
    return company + "-" + doc.padLeft(10, "0")
}
```

Signature options:
- **Simple** — scalar in, scalar out, called per value
- **Advanced (queue)** — `String[] input, Output output, MappingContext context` — gives access to the whole context queue, so you can implement custom node functions

`MappingContext` provides `getHeader(name)` and `getProperty(name)`, which is how a mapping reads values set upstream in the flow. Handy for injecting a run timestamp or a parameter without threading it through the payload.

Keep custom functions small and pure. A custom function making HTTP calls is a design smell — that belongs in a Content Enricher or a lookup step.

#### XSLT

Full XSLT 1.0 and 2.0/3.0 support depending on the processor configured. Use it for:

- Recursive structures (BOMs, nested hierarchies)
- Grouping — `xsl:for-each-group` in XSLT 2.0 is dramatically simpler than sortByKey/splitByValue chains
- Reusable named templates across similar mappings
- Wholesale restructuring where field-by-field mapping would be hundreds of lines of clicking

Accessing exchange properties and headers requires declaring parameters:

```xml
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:param name="orderNumber"/>

  <xsl:template match="/">
    <Result>
      <Ref><xsl:value-of select="$orderNumber"/></Ref>
      <xsl:for-each-group select="//Item" group-by="Material">
        <Group material="{current-grouping-key()}">
          <xsl:for-each select="current-group()">
            <Qty><xsl:value-of select="Quantity"/></Qty>
          </xsl:for-each>
        </Group>
      </xsl:for-each-group>
    </Result>
  </xsl:template>
</xsl:stylesheet>
```

Parameters are populated from exchange headers/properties of the same name, configured on the XSLT Mapping step.

Always declare and use namespaces explicitly. Silently empty output from an XSLT is namespace mismatch in the overwhelming majority of cases — the source has a default namespace that the XPath doesn't account for.

#### Format conversion

**XML to JSON** — watch for: single-element arrays collapsing to objects (the converter can't know an element is a collection from one instance), attributes getting `@` prefixes, and numeric/boolean values becoming strings. Configure "Suppress JSON Root Element" and the array handling options deliberately.

**JSON to XML** — JSON allows names XML doesn't (leading digits, special characters). The converter has a "JSON to XML Name Conversion" option; without it, invalid element names cause a parse failure downstream.

**CSV to XML / XML to CSV** — the converter handles delimiters, headers and field names. Quoted fields containing the delimiter are handled, but embedded newlines within quoted fields are a known trouble spot; if the source has them, pre-process.

**EDI** — the EDI to XML / XML to EDI converters plus Integration Advisor for the message implementation guidelines. B2B scenarios also involve Trading Partner Management for partner-specific configuration.

**Base64 / encoding** — encoder/decoder steps handle Base64 and GZip. Character encoding mismatches (payload declared UTF-8 but actually ISO-8859-1) produce mojibake or parse errors; the Content Modifier can set `CamelCharsetName`.

#### Value mappings

For code list translations — plant codes, unit of measure, country codes, partner IDs. Maintained as a separate artifact and changeable without touching the mapping.

Structure: source agency + source identifier + value → target agency + target identifier + value.

Use them when the list changes independently of the interface logic, or is maintained by business users. Don't use them for two-entry conditions — a `fixValues` function is simpler and stays visible in the mapping.

Access from Groovy via `ValueMappingApi` (see `groovy.md`) when the lookup is needed outside a mapping step.

#### Common problems

**Target node repeats the wrong number of times** — context setting on a source field. Check the context of every field feeding that node.

**Target field is empty but source has data** — namespace mismatch, or the field is in a different context than the target expects.

**"Cannot produce target element" / mapping fails on optional data** — a mandatory target field has no source value. Use `mapWithDefault` or make the source path optional-safe.

**Empty target nodes appearing** — `if` where you wanted `ifWithoutElse`.

**Mapping works in simulation but fails at runtime** — the runtime payload differs from the test file. Usually namespace declarations, encoding, or an envelope the test file lacked. Pull a real payload from the MPL and simulate with that.

**Performance degradation at volume** — mapping a very large document in one pass. Split before mapping.

---

## 5. B2B and EDI

*Source: `sap-integration-suite/references/b2b-edi.md`*

### B2B and EDI Reference

Contents: [EDIFACT structure](#edifact-structure) · [X12 structure](#x12-structure) · [EDI steps](#edi-steps-in-cloud-integration) · [Control numbers](#control-numbers) · [Acknowledgements](#the-acknowledgement-ladder) · [AS2](#as2) · [AS4](#as4) · [SFTP and OFTP](#sftp-and-other-transports) · [Inbound pattern](#reference-inbound-pattern) · [Outbound pattern](#reference-outbound-pattern) · [Failure modes](#b2b-specific-failure-modes)

For Trading Partner Management, MIGs and MAGs, see `tpm-integration-advisor.md`.

#### EDIFACT structure

Three nested envelopes, the middle one optional:

```
UNA:+.? '                      Service String Advice (optional, declares delimiters)
UNB+UNOC:3+SENDERID:14+RECEIVERID:14+250804:1030+000000123'   Interchange header
  UNG+ORDERS+...'              Functional group header (optional, rare in practice)
    UNH+000000001+ORDERS:D:96A:UN'   Message header
      BGM+220+PO12345+9'
      DTM+137:20250804:102'
      NAD+BY+...'
      LIN+1++9781234567890:EN'
      ...
    UNT+25+000000001'          Message trailer: segment count + reference
  UNE+1+...'                   Group trailer
UNZ+1+000000123'               Interchange trailer: message/group count + reference
```

Default delimiters (overridable via UNA): `+` element, `:` component, `'` segment terminator, `?` release/escape, `.` decimal.

The release character matters more than people expect. A partner name containing `+` must be escaped as `?+`; unescaped, it silently shifts every subsequent element by one position. Data-dependent parsing failures on one partner's messages are frequently this.

**Version identification** lives in UNH element 2: `ORDERS:D:96A:UN` — message type, version, release, controlling agency. `D.96A` and `D.01B` dominate retail and automotive; verify per partner, because a MIG built against the wrong release will reject valid messages.

**Common messages:** ORDERS (purchase order), ORDRSP (order response), ORDCHG (change), DESADV (despatch advice / ASN), RECADV (receiving advice), INVOIC (invoice), DELFOR (delivery forecast), DELJIT (JIT call-off), PRICAT (price catalogue), INVRPT (inventory report), SLSRPT (sales report), IFTMIN (transport instruction), CONTRL (syntax acknowledgement), APERAK (application error/acknowledgement).

Subsets exist and matter: EANCOM (retail, GS1), ODETTE and VDA (automotive), EDIFICE (electronics). A partner saying "standard EDIFACT ORDERS" usually means their EANCOM subset with mandatory fields the base standard treats as optional.

#### X12 structure

```
ISA*00*          *00*          *ZZ*SENDERID       *ZZ*RECEIVERID     *250804*1030*U*00401*000000123*0*P*>~
  GS*PO*SENDERID*RECEIVERID*20250804*1030*1*X*004010~
    ST*850*0001~
      BEG*00*SA*PO12345**20250804~
      ...
    SE*25*0001~
  GE*1*1~
IEA*1*000000123~
```

The ISA segment is **fixed-width** — 106 characters, every field padded to exact length. This trips up hand-built test files constantly; a one-character deviation makes the whole interchange unparseable. ISA16 declares the component separator, and the character immediately after ISA16 is the segment terminator, so the parser learns the delimiters from ISA itself.

ISA15 is the test/production indicator (`T`/`P`). Partners routinely forget to flip it at go-live, and a production system rejecting everything as test data is a common first-day incident.

**Common transaction sets:** 850 (PO), 855 (PO acknowledgement), 860 (PO change), 865 (change ack), 856 (ASN), 810 (invoice), 820 (payment/remittance), 830 (planning schedule), 862 (shipping schedule), 846 (inventory advice), 940/945 (warehouse shipping order/advice), 214 (transport status), 997 (functional acknowledgement), 999 (implementation acknowledgement), 824 (application advice).

Versions are `004010`, `005010`, etc., in GS08. HIPAA scenarios pin 005010 variants specifically.

#### EDI steps in Cloud Integration

Found in the palette under the B2B group. Availability depends on tenant capabilities being enabled.

**EDI Splitter** — takes an interchange and emits individual messages. Configure per format (EDIFACT / X12). Key settings:
- *Create Message Header* — surfaces envelope fields as exchange headers, which is how you get sender/receiver IDs and control numbers for routing and agreement lookup
- *Validate Message* — schema validation during split
- *Transaction Mode* / interchange handling — whether one bad message fails the whole interchange or only itself
- Acknowledgement generation options

The split-level error decision is a design choice worth making consciously: partners generally prefer a CONTRL/997 that accepts 19 of 20 messages and rejects one, over a blanket rejection of the interchange.

**EDI Extractor** — pulls envelope metadata without splitting.

**EDI to XML Converter / XML to EDI Converter** — format conversion driven by an XSD generated from a MIG (or a standard schema). The XML representation is what mappings and MIGs work against.

**EDI Validator** — validates against the guideline, separately from conversion, so you can validate then decide what to do with failures rather than just throwing.

#### Control numbers

UNB reference / ISA13 (interchange), UNG / GS06 (group), UNH / ST02 (message). Partners often require strict monotonic increment and will reject gaps or repeats.

Use **Number Ranges** (Monitor → Manage Stores → Number Ranges) rather than timestamps or random values. Define one range per partner per direction — a shared range across partners produces gaps from each partner's perspective, which some VANs and partners treat as lost messages.

Store the outbound control number against the message in a Data Store if you need to correlate an inbound 997/CONTRL back to what it acknowledges. Without that correlation, an acknowledgement telling you interchange 000000123 was rejected is useless.

Rollover: ISA13 is 9 digits, UNB reference is 14. Plan for wraparound on high-volume partners, and confirm the partner tolerates reset to 1.

#### The acknowledgement ladder

Three independent layers, frequently conflated in requirements discussions. Ask which one the partner actually means:

| Layer | EDIFACT | X12 | AS2 | Means |
|---|---|---|---|---|
| Transport | — | — | MDN | The bytes arrived and decrypted/verified |
| Syntax | CONTRL | 997 / 999 | — | The EDI structure parsed and is valid |
| Application | APERAK, ORDRSP | 824, 855 | — | The business system accepted the content |

An MDN is not confirmation that anything was processed. A 997 is not confirmation the order was accepted. Interfaces where the partner believes a 997 means "order accepted" produce disputes months later; make the distinction explicit in the interface spec.

**Timing expectations** are part of the trading agreement: MDN typically synchronous or within minutes, 997/CONTRL commonly within an hour to a day, application acknowledgement per business SLA. Build the timeout monitor — an acknowledgement that never arrives is silent unless something is watching for it.

**Negative acknowledgements** must be routed to someone who can act. A 997 with AK501 = R for a rejected PO is a business event, not a technical error, and it belongs in front of the order desk rather than in a support queue.

#### AS2

The dominant internet EDI transport. Configuration is symmetric and both sides must match exactly.

| Setting | Notes |
|---|---|
| AS2 ID (sender/receiver) | Case-sensitive, exchanged during onboarding |
| Signing | Own private key; partner verifies with your public cert |
| Encryption | Partner's public cert; they decrypt with their private key |
| MIC algorithm | SHA-256 standard now; SHA-1 legacy and still common |
| Encryption algorithm | AES-128/256; 3DES legacy |
| Compression | Optional, before or after signing — must agree which |
| MDN | Synchronous (same HTTP response) or asynchronous (separate callback) |
| Signed MDN | Usual requirement; carries the MIC for non-repudiation |
| Async MDN URL | Your publicly reachable endpoint — needs its own iFlow |

**Certificate direction is the single most common onboarding error.** You sign with *your* private key and encrypt with *their* public key. Partners send certificates without saying which is for signing and which for encryption, and a mismatch produces an opaque decryption failure at the far end.

**MIC mismatch in the MDN** means the content changed in transit or the algorithms disagree — usually compression order or a signing algorithm mismatch, not actual corruption.

**Duplicate detection**: AS2 Message-Id persistence rejects replayed messages. Enable it. Partners retry on missing MDNs, and a duplicate ASN that creates a second goods receipt is an expensive mistake.

**Asynchronous MDN** requires a separate inbound flow to receive it, plus correlation back to the original message and a timeout monitor. Prefer synchronous MDN unless the partner insists.

Certificate expiry is the classic B2B outage. Track every partner certificate with its expiry and an owner, and start renewal conversations weeks ahead — partner-side change requests move slowly.

#### AS4

ebMS3-based, gaining ground in energy markets and European regulated exchanges. Supports reliable messaging, WS-Security, and receipts (analogous to MDN). Configuration is driven by a PMode agreement between parties. Where AS2 has a handful of settings, AS4 has a formal profile — get the partner's PMode document rather than guessing.

#### SFTP and other transports

Plenty of B2B still runs over SFTP, and VANs remain common.

- **File naming conventions** are part of the agreement — often encoding partner, document type and sequence. Getting the pattern wrong means files sit unprocessed.
- **Partial file reads**: use change detection or a done-file convention so you don't pick up a file mid-upload. This causes intermittent, unreproducible parse errors.
- **Post-processing**: move to an archive directory rather than delete, at least initially. Being able to reprocess yesterday's file is worth the storage.
- **VAN mailboxes** typically behave as SFTP or AS2 endpoints; the VAN handles partner-side routing, which simplifies onboarding but adds a dependency and a cost per kilocharacter.
- **OFTP2** is standard in European automotive; check current adapter availability for the tenant rather than assuming.

#### Reference inbound pattern

```
AS2 / SFTP sender
  → EDI Splitter (create message header, validate)
      per message:
      → Extract sender/receiver ID, message type, control numbers to properties
      → Agreement lookup (TPM) or Partner Directory lookup
      → EDI to XML Converter (partner-specific MIG schema)
      → Validate against MIG
      → Mapping (MAG or Message Mapping) → IDoc / OData / API payload
      → Send to backend (async via JMS)
  → Generate CONTRL / 997 from split results
  → Send acknowledgement back over the partner's outbound channel
```

Persist the raw interchange before splitting. When a partner disputes what they sent, the original bytes are the only useful evidence, and the transformed version isn't.

#### Reference outbound pattern

```
Backend event (IDoc / API / scheduled extract)
  → Determine partner from the business data
  → Agreement lookup → partner-specific mapping and channel
  → Mapping → EDI XML structure
  → Assign control numbers from Number Range
  → XML to EDI Converter
  → Envelope (UNB/UNZ or ISA/IEA)
  → AS2 / SFTP receiver
  → Store control number + business key for acknowledgement correlation
  → (separate flow) Await MDN and 997; alert on timeout
```

#### B2B-specific failure modes

| Symptom | Likely cause |
|---|---|
| Partner rejects interchange, no detail | Delimiter or envelope mismatch; compare against their sample byte-for-byte |
| Every field shifted by one from a certain point | Unescaped release character in data |
| ISA parse failure | Fixed-width padding wrong in ISA |
| Partner sees test data in production | ISA15 set to `T` |
| Intermittent parse failures on one partner | Partial file read; character encoding; or an optional segment appearing for the first time |
| AS2 decryption failure at partner | Wrong certificate used for encryption |
| MIC mismatch in MDN | Compression/signing order or algorithm mismatch |
| Duplicate goods receipts / invoices | AS2 duplicate detection disabled, partner retried |
| 997 says rejected, nobody notices | No routing of negative acknowledgements to a business owner |
| Control number gap complaints | Shared number range across partners, or a failed message consuming a number |
| Works for partner A, fails for B with same doc type | Partner-specific subset differences — this is why per-partner MIGs exist |
| Certificate expiry outage | No expiry tracking |

The pattern underneath most of these: B2B failures are usually *agreement* failures, not technical ones. Before debugging code, get the partner's specification and a real sample file, and diff against what you're producing.

---

## 6. Trading Partner Management & Integration Advisor

*Source: `sap-integration-suite/references/tpm-integration-advisor.md`*

### Trading Partner Management and Integration Advisor Reference

Contents: [When to use TPM](#when-to-use-tpm) · [Object model](#tpm-object-model) · [Type systems](#type-systems) · [MIGs](#message-implementation-guidelines) · [MAGs](#mapping-guidelines) · [Agreements](#agreement-templates-and-agreements) · [Runtime](#how-runtime-lookup-works) · [Partner Directory](#partner-directory) · [Onboarding](#partner-onboarding-workflow) · [Monitoring](#b2b-monitoring) · [Pitfalls](#pitfalls)

Names and screen locations in this area move between releases more than elsewhere in Integration Suite. Treat structure and concepts as reliable; verify exact artifact names and menu paths against the tenant.

#### When to use TPM

TPM earns its complexity when you have **many partners exchanging similar documents with per-partner variations**. It replaces "clone the iFlow per partner" with generic flows plus per-partner configuration data.

Rough guidance:

- **Under ~5 partners, one or two document types, stable** — TPM is probably overhead. Parameterised iFlows plus Partner Directory may be enough.
- **10+ partners, or partners onboarding regularly, or several document types each** — TPM pays for itself quickly. Onboarding partner 30 should be configuration, not development.
- **Regulated/audited exchanges** — the agreement model gives you a documented, versioned record of what was agreed with whom, which is hard to reconstruct from thirty cloned iFlows.

The honest tradeoff: TPM front-loads effort. The first partner takes longer than a hand-built flow would. The break-even is typically somewhere between the fifth and tenth partner, sooner if document types multiply.

#### TPM object model

**Company Profile** — your own organisation. One per tenant, usually. Holds your identifiers, systems and communication channels.

**Trading Partner Profile** — one per partner. Same structure as the company profile.

Within each profile:

- **Identifiers** — how this party is identified in message envelopes, each with a scheme/agency: GLN, DUNS, AS2 ID, ILN, internal partner number. A partner typically has several, used in different contexts (AS2 header vs. UNB vs. backend partner number). Getting the identifier scheme wrong is the most common cause of agreement lookup failing at runtime.
- **Systems** — the backend or partner-side systems participating, with a system type. Used to decide which mapping and which channel apply.
- **Communication channels** — sender and receiver channels with adapter configuration (AS2, SFTP, HTTPS, AS4). This is where the partner's endpoint, credentials and certificates live.
- **Certificates** — signing and encryption certificates, per direction.

**Type Systems** — the vocabulary of business document standards (EDIFACT, X12, IDoc, and custom ones you import).

**Message Implementation Guideline (MIG)** — a partner-specific or scenario-specific subset of a standard message.

**Mapping Guideline (MAG)** — a mapping between two MIGs.

**Agreement Template** — a reusable blueprint describing a B2B scenario: which activities run, which transactions they carry, which interchange settings apply.

**Trading Partner Agreement** — a template instantiated for a specific partner, with that partner's identifiers, channels and any deviations. Activating it publishes the configuration to the runtime.

**Activities** — the units inside a template/agreement: typically an inbound or outbound flow of one document type, referencing the transformation (MAG or custom) and the channel.

**Custom transformations / custom activities** — hooks to call your own iFlow when a partner needs something the generic model doesn't cover. Useful escape hatch; use it sparingly, since heavy use erodes the reason for adopting TPM.

#### Type systems

Ships with the major standards — UN/EDIFACT across releases, ASC X12 across versions, SAP IDoc, SAP APIs. Subsets like EANCOM and ODETTE may need importing or building depending on the tenant.

You can create custom type systems from XSD, which is how proprietary partner formats and internal canonical structures enter the model. Worth doing for a canonical intermediate format if you're mapping many partners to one backend — see [Pitfalls](#pitfalls).

#### Message Implementation Guidelines

A MIG takes a standard message (say EDIFACT ORDERS D.96A) and narrows it to what a specific partner or scenario actually uses:

- Set cardinality — make optional segments mandatory, or exclude unused ones entirely
- Restrict code lists to the values the partner sends
- Add qualifiers so repeated segments become distinguishable (the NAD segment appears many times; qualifying it as NAD+BY, NAD+SU, NAD+DP is what makes mapping tractable)
- Add documentation and examples per node
- Define custom fields for partner extensions

MIGs have lifecycle states — draft, then a released state once stable. Released MIGs are versioned; changing a released MIG means a new version, and consumers (MAGs, iFlows) must be pointed at it deliberately. This is a feature, not friction: a partner changing their format shouldn't silently alter what everyone else gets.

Outputs you can generate from a MIG: XSD (used by the EDI converters and validators), and human-readable documentation to send to the partner. That documentation is genuinely useful in onboarding — it's the specification you and the partner agree against.

**Qualification is the single highest-value activity in MIG work.** An unqualified MIG produces mappings full of positional guesswork that break when the partner reorders segments. Time spent qualifying up front is repaid every time the MIG is mapped.

#### Mapping Guidelines

A MAG maps a source MIG to a target MIG — typically partner EDIFACT/X12 to SAP IDoc, or IDoc out to partner format.

The **proposal engine** suggests mappings based on aggregated usage across the Integration Advisor community. Proposals are a starting point with real hit rates on common pairs (ORDERS to ORDERS05, 850 to ORDERS05), not an answer. Review every proposed line — proposals reflect what is common, not what your partner's contract says.

Per-line you can apply functions, constants, conditions and value mappings, much as in Message Mapping. Complex logic still has limits; a custom function or a downstream iFlow step handles what the MAG can't.

MAGs can be used directly by a mapping step in an iFlow, or exported (XSLT) for use outside TPM. Direct reference keeps the guideline as the single source of truth, which is the point.

Like MIGs, MAGs are versioned and have lifecycle states.

#### Agreement templates and agreements

The template is where the reusable design lives; the agreement is where the partner-specific values live. Getting this division right determines whether TPM actually saves you effort.

**Belongs in the template:** the set of activities, document types, transformation references, interchange defaults, acknowledgement requirements, error handling behaviour.

**Belongs in the agreement:** partner identifiers, communication channel selection, certificate references, control number ranges, any partner-specific deviations the template allows as parameters.

If you find yourself creating a new template per partner, the templates are carrying partner data that should be parameters. That's the TPM equivalent of cloning iFlows.

**Activation** is the step that publishes an agreement to the runtime. Until activated, the runtime lookup won't find it and messages fail with an agreement-not-found error. Deactivate and reactivate after changes — edits to an active agreement are not automatically live in all cases, and this catches people repeatedly.

Agreements are also the natural place to record the commercial and operational agreement: acknowledgement SLAs, test/production status, effective dates, contacts.

#### How runtime lookup works

The generic B2B iFlows extract identifying information from the incoming message — sender identifier, receiver identifier, document type, and for X12/EDIFACT the version — and use that tuple to find the matching activated agreement. The agreement supplies the MIG for validation, the MAG for transformation, and the channel for the response.

SAP ships prebuilt generic flows for this (available from the content catalogue / Business Accelerator Hub); the package and flow names vary by release, so pull the current version rather than working from a blog post. You typically copy them into your own package and extend rather than modifying the shipped originals, so updates remain consumable.

The practical implication for troubleshooting: **agreement lookup failures are identifier problems.** When a message fails with no matching agreement, compare the identifier value and scheme in the message envelope character-for-character against the identifier configured on the partner profile. Case, leading zeros, and scheme qualifier are all common mismatches.

#### Partner Directory

The underlying runtime store for partner-specific configuration. TPM writes to it on agreement activation, and you can also use it directly without TPM — which is the right lightweight option for a handful of partners.

Holds per-partner:
- **String parameters** — endpoints, IDs, flags
- **Binary parameters** — certificates, XSLTs, schemas
- **Alternative partners** — identifier aliases, so one partner known by several IDs resolves to one configuration
- **Authorized users** — for inbound authentication mapping

Managed via the Partner Directory OData API (`/api/v1/PartnerDirectory` with entity sets for Partners, StringParameters, BinaryParameters, AlternativePartners, AuthorizedUsers). There's no full-featured UI for direct maintenance in all releases, so scripted maintenance is normal.

Reading a parameter in Groovy:

```groovy
import com.sap.it.api.ITApiFactory
import com.sap.it.api.pd.PartnerDirectoryService

def service = ITApiFactory.getService(PartnerDirectoryService.class, null)
def endpoint = service.getParameter("Receiver_URL", partnerId, String.class)
```

Verify the exact factory method (`getService` vs `getApi`) against your tenant's API version — both appear in SAP examples.

Binary parameters can hold an XSLT, letting one generic iFlow apply a different transformation per partner without any branching in the flow. It's an elegant pattern, and also one that makes the flow's behaviour invisible in the model — document it clearly or the next person cannot work out where the mapping comes from.

#### Partner onboarding workflow

A repeatable sequence, once the templates exist:

1. **Collect the partner specification** — document types, versions, subsets, sample files, identifiers, transport, certificates, acknowledgement expectations, test/production timeline.
2. **Create the trading partner profile** — identifiers with correct schemes, systems, communication channels, certificates.
3. **Build or reuse MIGs** — start from an existing MIG for that document type and adjust for this partner's deviations. Reuse is the whole point; a partner-specific MIG per partner per document is normal, but they should be derived from a common baseline.
4. **Build or reuse MAGs** — map partner MIG to your canonical/backend MIG.
5. **Instantiate the agreement** from the template; fill in partner-specific values.
6. **Activate** in the test environment.
7. **Connectivity test** — transport only. AS2 handshake and MDN, or SFTP file drop. Do this before any content testing; it isolates certificate and firewall issues from data issues.
8. **Content test** — real sample files both directions, including acknowledgements and at least one deliberately invalid message to confirm the rejection path works.
9. **Parallel run** if replacing an existing connection, comparing output against the incumbent.
10. **Cut over** — flip ISA15/test indicators, activate in production, monitor closely for the first days.

Steps 7 and 8 being separate matters. Debugging a certificate problem and a mapping problem simultaneously takes several times as long as debugging them in sequence.

Keep a per-partner onboarding checklist artefact. It becomes the handover document to support.

#### B2B monitoring

Integration Suite provides a dedicated B2B monitor alongside the standard message monitoring, showing interchanges and the business documents within them, with acknowledgement status and links to the corresponding MPL entries.

What to watch operationally:

- Interchanges received vs. acknowledged — a growing gap means acknowledgements aren't flowing
- Negative acknowledgements — these need a business owner, not a support queue
- Per-partner failure rates — a single partner's rate jumping usually means they changed something
- Acknowledgement timeouts — nothing surfaces these unless you build a monitor for them

Correlating a partner's question ("what happened to PO 12345?") to a message requires a business key in the MPL. Set `SAP_ApplicationID` to the document number and add the partner ID and control number as custom header properties. Without it, answering partner queries means searching by timestamp.

#### Pitfalls

**Cloning agreement templates per partner.** The signal that partner data has leaked into the template. Parameterise instead.

**Skipping the canonical format.** Mapping every partner format directly to every backend format is an N×M problem. Mapping partner → canonical → backend is N+M. With more than a handful of partners and more than one backend, the canonical intermediate is worth the extra hop. Build it as a custom type system so MIGs and MAGs work against it natively.

**Under-qualified MIGs.** Produces fragile mappings. Qualify segments properly the first time.

**Trusting proposals.** The proposal engine reflects community usage, not your partner's contract.

**Forgetting to activate, or to reactivate after changes.** The most common "why isn't my change live" in TPM.

**No acknowledgement timeout monitoring.** Missing acknowledgements are silent by nature.

**Certificates without an expiry register.** Every partner certificate needs a tracked expiry date and a named owner on both sides.

**Treating TPM as configuration-only.** You will still need custom iFlows for backend-specific handling, unusual partner requirements, and error routing. Plan for the escape hatches rather than being surprised by them.

**Testing with synthetic files only.** Partner-generated files contain things your generator never produces — unusual characters, unexpected optional segments, different line endings. Get real files before go-live.

---

## 7. Error Handling & Resilience

*Source: `sap-integration-suite/references/error-handling.md`*

### Error Handling and Resilience Reference

Contents: [Classify first](#classify-the-error-first) · [Exception subprocess](#exception-subprocess) · [End events](#end-events-and-message-status) · [Retry](#retry-strategies) · [Idempotency](#idempotency) · [Quality of service](#quality-of-service) · [Alerting](#alerting) · [Reprocessing](#reprocessing) · [Sync vs async](#sync-vs-async-error-behaviour)

#### Classify the error first

Most badly-designed error handling comes from treating all failures identically. Three categories, three different responses:

**Technical / transient** — receiver down, timeout, network, 503, connection reset. Retry is appropriate and usually sufficient.

**Technical / permanent** — malformed payload, mapping failure, authentication failure, 400/401/403. Retry will never succeed; it just delays the alert and burns capacity. Fail fast and notify.

**Business** — the data is well-formed but rejected by the receiver's business logic (material doesn't exist, order already closed, credit check failed). Retry is wrong. This usually needs to go back to a business user, not to support.

A common failure of design is routing all three into the same dead-letter queue where nobody distinguishes them. Decide, per interface, what should happen to each — and write it into the interface spec.

#### Exception subprocess

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

#### End events and message status

| End event | Resulting MPL status | Use when |
|---|---|---|
| End Message | Completed | Handled successfully, nothing more to do |
| Error End Event | Failed | The message genuinely failed and should show as failed |
| Escalation End Event | Escalated | Handled the error, but flag it as needing attention |
| Terminate Message | Completed (no further processing) | Deliberate stop, e.g. filtered out |

Getting this wrong distorts monitoring. A flow that catches every error and ends with End Message shows 100% Completed while quietly dropping messages — the monitoring dashboard becomes actively misleading. Escalated is the right status for "we handled it but a human should know".

#### Retry strategies

**Adapter-level retry** — some receiver adapters have built-in retry settings. Simplest option when it fits.

**JMS-based retry** — the standard pattern. The consuming flow reads from a queue; on exception, the message is returned to the queue and retried with exponential backoff up to a configured maximum, then moved to dead-letter status visible in Monitor → Manage Message Queues. This is the most robust general-purpose retry mechanism because state survives node restarts.

**Data Store polling retry** — write failed messages to a Data Store with a retry counter, and have a scheduled flow pick them up. More work than JMS, but gives full control over the retry schedule and lets you inspect the queue as data.

**JMS retry design points:**
- Set the max retry count deliberately. Infinite retry against a permanently broken receiver fills the queue.
- Only retry transient errors. Route permanent failures to dead-letter immediately rather than retrying twelve times first.
- Backoff should account for the receiver's recovery time — retrying every 5 seconds against a system doing a 30-minute maintenance window is pointless load.

**Do not retry non-idempotent operations without dedup.** See below.

#### Idempotency

The question to ask on every async interface: *what happens if this message is delivered twice?*

Duplicate delivery happens through retry after ambiguous failure (the call timed out — did the receiver process it?), node restart mid-processing, and sender resends. This is not an edge case.

Options, best first:

**1. The receiver is idempotent.** An upsert keyed on a business ID, or an API that accepts an idempotency key. Verify this rather than assuming it — "our API is idempotent" is frequently untrue in practice.

**2. Idempotent Process Call.** Built-in step: give it a message ID expression, and it detects and suppresses duplicates against a persistent store. The right tool for most cases.

**3. Data Store deduplication.** Write the message ID with `overwrite = false`; a duplicate key throws, which you catch and treat as a duplicate. More manual but transparent.

**4. Query-before-write.** Check the receiver for the record before creating it. Racy under parallelism, and doubles the call volume. Last resort.

Pick the ID carefully. It must be stable across retries and unique per business event — a UUID generated inside the flow is regenerated on retry and therefore useless. Use the sender's document number, or a hash of the payload.

#### Quality of service

- **Best effort** — synchronous, no persistence, no retry. Failure is immediately visible to the caller.
- **Exactly once (EO)** — asynchronous with persistence and duplicate prevention. JMS or XI adapter.
- **Exactly once in order (EOIO)** — as EO, plus ordering by a sequence key. Requires single-threaded consumption; throughput is bounded by the slowest message. Only design for this when ordering genuinely matters — it's frequently requested and rarely necessary.

If EOIO is requested, ask what breaks if messages arrive out of order. Often the answer reveals that per-key ordering (all messages for one customer, not all messages globally) is what's actually needed, which is far cheaper.

#### Alerting

An interface with no alerting path is an interface that fails silently until a business user notices missing data weeks later. Options:

- **Cloud ALM** — SAP's monitoring solution; integrates with Integration Suite for exception monitoring and alerting. The strategic answer if the customer has it.
- **Mail receiver in the exception subprocess** — simplest, works everywhere. Keep the payload out of the mail body if it contains sensitive data; send the MPL ID instead so support can look it up.
- **Ticketing integration** — POST to ServiceNow/Jira from the exception subprocess. Include the MPL ID, interface name, error category and business key.
- **Monitoring API polling** — an external tool queries the MPL OData API for Failed/Escalated messages on a schedule. Good for consolidated dashboards. See `monitoring-troubleshooting.md`.

Whatever the channel, the alert must contain the MPL ID and a business key. An alert saying "Interface X failed" without either sends support hunting through monitoring for the right message.

Deduplicate alerts. A receiver outage generating 4,000 individual emails trains everyone to filter them.

#### Reprocessing

Design the reprocessing path before go-live, not during the first incident:

- **JMS dead letter** — messages can be moved back for reprocessing from Manage Message Queues.
- **Data Store staging** — a reprocessing flow reads pending entries and reruns them. Give it a manual trigger endpoint.
- **Source resend** — sometimes the cleanest answer is "ask the sender to resend", but only if the sender can, and only if the receiver tolerates duplicates.

Document which applies, and who is authorised to do it. Reprocessing usually requires production authorization the on-call person may not have.

#### Sync vs async error behaviour

**Synchronous:** the caller is waiting. Return a meaningful HTTP status and a structured error body. Don't return 200 with an error inside unless the API contract says so. Map internal errors to sensible codes — 400 for bad request, 502 for downstream failure, 504 for downstream timeout. Set `CamelHttpResponseCode` in the exception subprocess.

**Asynchronous:** nobody is waiting, so the error must be persisted and surfaced. This is where status choice, alerting and reprocessing matter most. The sender has already been told 202 Accepted, and that promise needs backing.

A frequent design error: an async interface that returns 202, then fails, then does nothing. The sender believes it succeeded. If you accept a message, you own it.

---

## 8. Security & Connectivity

*Source: `sap-integration-suite/references/security-connectivity.md`*

### Security and Connectivity Reference

Contents: [Security material](#security-material) · [Inbound auth](#inbound-authentication) · [Outbound auth](#outbound-authentication) · [OAuth flows](#oauth-flows) · [Certificates](#certificates-and-tls) · [Cloud Connector](#cloud-connector) · [Principal propagation](#principal-propagation) · [Message-level security](#message-level-security) · [Common errors](#common-connectivity-errors)

#### Security material

Managed under Monitor → Manage Security → Security Material. Types:

| Type | Contains | Referenced by |
|---|---|---|
| User Credentials | username + password | Adapter credential name, `SecureStoreService` |
| OAuth2 Credentials | client ID/secret, token URL, scope | Adapter, or fetched in script |
| Secure Parameter | a single secret string | `SecureStoreService` (value in password field) |
| Known Hosts | SSH host keys | SFTP adapter |
| Keystore | certificates and private keys | TLS, signing, encryption |
| PGP keys | public/secret keyrings | PGP encryptor/decryptor steps |

Everything is referenced by **alias**. Externalize the alias name so DEV/QA/PRD flows point at environment-appropriate credentials without changing the flow.

Never put a password in an externalized parameter — externalized parameters are stored and displayed in plain text in the configuration screen.

#### Inbound authentication

How callers authenticate *to* your iFlow's sender endpoint:

**Client certificate** — mutual TLS. The client cert's subject/issuer DN pair is mapped to a role in the tenant's Certificate-to-User Mappings. Strongest option and common in SAP-to-SAP scenarios.

**OAuth 2.0 client credentials** — the caller obtains a token from the tenant's token endpoint using a service key, then presents the bearer token. This is the standard approach for modern API consumers and what you should default to.

**Basic authentication** — user in the BTP identity provider with the required role. Acceptable for internal or legacy callers; increasingly restricted by SAP for new tenants.

Authorization is role-based. The default role for message sending is `ESBMessaging.send`. In production, create dedicated roles per consumer group rather than granting every caller the default role — otherwise any authenticated client can invoke any endpoint on the tenant. This is a genuinely common finding in security reviews.

The endpoint address configured on the HTTPS sender is a path suffix; the full URL is `https://<tenant-runtime-host>/http/<your-path>`.

#### Outbound authentication

How your iFlow authenticates *to* receivers:

| Method | Configuration |
|---|---|
| Basic | User Credentials alias on the adapter |
| Client certificate | Private key alias in keystore, selected on the adapter |
| OAuth2 Client Credentials | OAuth2 Credentials artifact, adapter handles token fetch and caching |
| OAuth2 SAML Bearer Assertion | For principal propagation to SAP systems |
| OAuth2 Authorization Code | Rarely applicable — requires interactive consent |
| Token from a custom flow | Fetch in a sub-flow, cache in a variable, set Authorization header |

Prefer adapter-level configuration over scripted token handling. The adapter caches tokens and handles refresh; a hand-rolled implementation usually doesn't, and re-authenticates on every message until someone notices the token endpoint rate-limiting.

When you genuinely must fetch tokens yourself (unusual auth schemes, token exchange), cache the token in a global variable with its expiry and only refresh when needed:

```groovy
def expiry = message.getProperty("tokenExpiry") as Long
if (expiry == null || System.currentTimeMillis() > expiry - 60000) {
    // fetch new token, store it and expiry - 60s safety margin
}
```

#### OAuth flows

**Client Credentials** — machine-to-machine, no user context. The default for system integration.

**SAML Bearer Assertion** — the flow asserts a user identity to the target, letting the target apply that user's authorizations. Requires trust configuration on both sides and a signing key in the tenant keystore. Used for principal propagation to SAP backends.

**JWT Bearer** — similar, with a JWT instead of a SAML assertion.

**Authorization Code** — needs a browser and user consent. Not workable for unattended integration; if a requirement seems to need it, the actual answer is usually a long-lived refresh token obtained once out-of-band, or a different grant.

#### Certificates and TLS

The tenant keystore holds:
- **Own certificates** (with private key) — used to identify the tenant to receivers, and to sign
- **Trusted certificates / root CAs** — used to validate receivers' certificates

Practical points:

- Server certificate validation requires the receiver's **full chain** — root and any intermediates — in the keystore. Missing intermediate is the most common cause of `SSLHandshakeException: unable to find valid certification path`.
- Certificate expiry is a scheduled outage waiting to happen. The tenant surfaces expiring certificates in monitoring; put a calendar reminder in independently. A production interface failing at 03:00 on renewal day is a classic.
- SAP rotates its own load balancer root certificates periodically, and consumers calling your tenant need the current root. Watch the relevant SAP Notes.
- Key pairs used for signing (SAML, PGP, message-level) should be documented with their purpose — keystores accumulate mystery entries nobody dares delete.

#### Cloud Connector

The tunnel from BTP to on-premise systems. The iFlow addresses a **virtual host:port**, which Cloud Connector maps to the internal host:port.

Configuration checklist when a connection fails:

1. Is the Cloud Connector connected to the correct subaccount? (Green in the CC admin UI)
2. Is the system mapping present, with the right virtual host and protocol?
3. Are the **resources** exposed? Access control is path-based and defaults to nothing. A mapping with no accessible resource returns a failure that looks like a routing problem.
4. Does the path match — including the trailing-slash and sub-path settings?
5. For RFC: is the resource an exposed function module or prefix?
6. Is the on-premise system reachable *from the Cloud Connector host*? Test there, not from your laptop.

In the iFlow, set Proxy Type = `On-Premise` and Location ID if multiple Cloud Connectors serve the subaccount. Forgetting the Location ID with multiple connectors produces intermittent failures as requests hit the wrong one.

#### Principal propagation

Carrying the calling user's identity through to the backend so the backend applies that user's authorizations, rather than everything running as a technical user.

Chain: caller authenticates to the iFlow → the flow obtains an assertion for that user → the receiver accepts it and establishes the user session. Requires trust configured at each hop (BTP subaccount trust, Cloud Connector principal propagation settings, backend certificate rules in `STRUST`/`SNC`), and the sender adapter must be configured to preserve the user context.

It is a genuinely fiddly setup and each hop has its own failure mode. When a customer asks for it, confirm it's actually required — often the real requirement is auditability of *which* user triggered a change, which can be satisfied by passing the user ID as a data field, at a fraction of the effort.

#### Message-level security

Beyond transport security, for cases where the payload must be protected end-to-end or non-repudiation is needed:

- **PGP** — encrypt/decrypt and sign/verify steps, keys in the PGP keyrings. Standard in B2B file exchange.
- **XML Digital Signature / XML Encryption** — for SOAP and XML payloads.
- **Simple Signer / Verifier** — PKCS#7/CMS signatures.
- **AS2 / AS4** — protocol-level security with MDN receipts, for EDI partners.

Sign-then-encrypt is the conventional order. Whatever you choose, document the key exchange process — the operational cost of message-level security is almost entirely in key lifecycle, not in the flow configuration.

#### Common connectivity errors

| Symptom | Likely cause |
|---|---|
| `unable to find valid certification path` | Missing root/intermediate CA in tenant keystore |
| `SSLHandshakeException: Received fatal alert: handshake_failure` | TLS version or cipher mismatch; or receiver requires a client cert you're not sending |
| HTTP 401 from receiver | Wrong credential alias, or alias not deployed to this tenant |
| HTTP 403 from receiver | Authenticated but not authorized — backend role missing |
| HTTP 403 "CSRF token validation failed" | CSRF fetch failed or token not reused on the same session; check the OData adapter's CSRF setting |
| `Connection refused` / `No route to host` on-premise | Cloud Connector mapping or resource not exposed |
| Sender returns 401 to caller | Caller's role assignment, or client cert not mapped to a user |
| Sender returns 403 to caller | Authenticated but lacking the role on the endpoint |
| Intermittent on-premise failures | Multiple Cloud Connectors, missing Location ID |
| `PKIX path building failed` after months of working | Certificate rotated at the receiver |
| Works in DEV, 401 in PRD | Credential alias deployed in DEV only — check Security Material per tenant |

That last row is worth internalising: security material is per-tenant and is not transported with the iFlow. Every transport to a new tenant needs the aliases created there first, or the flow deploys cleanly and fails on first message.

---

## 9. API Management

*Source: `sap-integration-suite/references/api-management.md`*

### API Management Reference

Contents: [Model](#object-model) · [Flow structure](#flow-structure) · [Policies](#policy-catalogue) · [Policy examples](#policy-examples) · [Variables](#variables) · [Fault handling](#fault-handling) · [Design guidance](#design-guidance)

SAP API Management is Apigee Edge under the hood, so Apigee documentation and policy semantics largely apply.

#### Object model

**API Provider** — the backend system definition (host, port, auth, or a Cloud Integration/on-premise target).

**API Proxy** — the managed façade. Holds the endpoints, flows and policies. This is where the work happens.

**API Product** — a bundle of proxies (or specific resources) exposed as a purchasable/subscribable unit. Rate plans attach here.

**Application** — a consumer's registration against products; issues the API key.

**Developer / Developer Portal** — consumer-facing catalogue, self-service key management, documentation.

Consumers subscribe to Products, not Proxies. If a consumer can't call an API despite a valid key, check that the resource path is actually included in the Product — a Product with the wrong resource path returns 403 with a valid key, which is a confusing combination.

#### Flow structure

Each proxy has two endpoints, each with three flow phases:

```
ProxyEndpoint (faces the consumer)
  PreFlow      → always runs first
  Conditional Flows → matched by path + verb
  PostFlow     → always runs last
        ↓
TargetEndpoint (faces the backend)
  PreFlow
  Conditional Flows
  PostFlow
        ↓ (backend call)
  Response flows run in reverse
```

Request order: ProxyEndpoint PreFlow → ProxyEndpoint Conditional → ProxyEndpoint PostFlow → TargetEndpoint PreFlow → Conditional → PostFlow → backend.

Response order: TargetEndpoint PreFlow → Conditional → PostFlow → ProxyEndpoint PreFlow → Conditional → PostFlow → consumer.

Placement rules that matter:
- Security policies (Verify API Key, OAuth, threat protection) go in **ProxyEndpoint PreFlow** — reject bad requests before doing any work.
- Spike Arrest goes early, before Quota — cheap check first.
- Backend-specific transformation goes in the TargetEndpoint.
- Response caching goes in the ProxyEndpoint so cache hits skip the backend entirely.

#### Policy catalogue

**Traffic management**
- *Quota* — allowed request count per time interval, per app/developer/product. Business-level limit.
- *Spike Arrest* — smooths bursts by enforcing an inter-request interval. Protection limit. Note it's per message processor, so the effective rate in a multi-node setup is higher than the configured number.
- *Concurrent Rate Limit* — caps simultaneous backend connections.
- *Response Cache* — caches backend responses by a key expression.
- *Reset Quota* — administrative reset.

**Security**
- *Verify API Key* — validates the app key.
- *OAuth v2.0* — generate/verify access tokens; the proxy acts as token endpoint.
- *Verify JWT / Generate JWT / Decode JWT* — JWT handling.
- *Basic Authentication* — encode/decode Basic credentials, typically to inject backend credentials.
- *XML Threat Protection / JSON Threat Protection* — structural limits (depth, element count, string length) against payload-based attacks.
- *Regular Expression Protection* — blocks injection patterns.
- *Access Control* — IP allow/deny lists.

**Mediation**
- *Assign Message* — construct or modify request/response: set headers, query params, payload, verb, status.
- *Extract Variables* — pull values from payload/header/path into variables via XPath, JSONPath or URI pattern.
- *JSON to XML* / *XML to JSON* — format conversion.
- *XSL Transform* — XML transformation.
- *Raise Fault* — return a custom error response and abort the flow.
- *Service Callout* — call an external service mid-flow (lookups, token exchange).
- *Key Value Map Operations* — read/write configuration and secrets from a KVM.
- *Access Entity* — retrieve app/developer/product metadata.

**Extension**
- *JavaScript* — custom logic. Available but should be the exception; policies are faster and declaratively visible.
- *Python Script*
- *Message Logging* — send log entries to an external syslog endpoint.
- *Statistics Collector* — capture custom dimensions for analytics.

#### Policy examples

**Verify API Key**

```xml
<VerifyAPIKey async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <APIKey ref="request.header.APIKey"/>
</VerifyAPIKey>
```

**Quota — 1000 calls per hour per app**

```xml
<Quota async="false" continueOnError="false" enabled="true" type="calendar" xmlns="http://www.sap.com/apimgmt">
    <Identifier ref="client_id"/>
    <Allow count="1000"/>
    <Interval>1</Interval>
    <TimeUnit>hour</TimeUnit>
    <Distributed>true</Distributed>
    <Synchronous>true</Synchronous>
</Quota>
```

`Distributed` and `Synchronous` both true gives an accurate shared count across nodes at some latency cost. False is faster but approximate.

**Spike Arrest**

```xml
<SpikeArrest async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <Rate>30ps</Rate>
    <Identifier ref="client_id"/>
</SpikeArrest>
```

`30ps` means one request per ~33ms, not 30 requests in any given second — bursts are smoothed, not counted.

**Assign Message — inject a backend header and strip the consumer's key**

```xml
<AssignMessage async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <Set>
        <Headers>
            <Header name="X-Correlation-Id">{messageid}</Header>
        </Headers>
    </Set>
    <Remove>
        <Headers>
            <Header name="APIKey"/>
        </Headers>
    </Remove>
    <AssignTo createNew="false" transport="http" type="request"/>
</AssignMessage>
```

Removing the consumer's credential before forwarding is a routine hardening step that's frequently skipped.

**Extract Variables from a JSON body**

```xml
<ExtractVariables async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <Source>request</Source>
    <JSONPayload>
        <Variable name="orderId">
            <JSONPath>$.order.id</JSONPath>
        </Variable>
    </JSONPayload>
    <VariablePrefix>ext</VariablePrefix>
</ExtractVariables>
```

Referenced afterwards as `ext.orderId`.

**Raise Fault**

```xml
<RaiseFault async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <FaultResponse>
        <Set>
            <Payload contentType="application/json">
{"error":"INVALID_REQUEST","message":"orderId is required"}
            </Payload>
            <StatusCode>400</StatusCode>
            <ReasonPhrase>Bad Request</ReasonPhrase>
        </Set>
    </FaultResponse>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</RaiseFault>
```

**Response Cache**

```xml
<ResponseCache async="false" continueOnError="false" enabled="true" xmlns="http://www.sap.com/apimgmt">
    <CacheKey>
        <KeyFragment ref="request.uri" type="string"/>
    </CacheKey>
    <ExpirySettings>
        <TimeoutInSec>300</TimeoutInSec>
    </ExpirySettings>
    <SkipCacheLookup>request.header.Cache-Control = "no-cache"</SkipCacheLookup>
</ResponseCache>
```

Include everything that varies the response in the cache key — auth context especially. A cache keyed only on URI in a multi-tenant API will serve one customer's data to another, which is the worst class of bug this product can produce.

#### Variables

Flow variables available in conditions and `ref` attributes:

```
request.verb, request.path, request.uri, request.queryparam.<name>
request.header.<name>, request.content
response.status.code, response.header.<name>, response.content
client_id, developer.email, apiproduct.name, apiproxy.name
target.url, messageid, system.time.year
```

Conditions on flows use the same variables:

```xml
<Condition>(proxy.pathsuffix MatchesPath "/orders/*") and (request.verb = "GET")</Condition>
```

#### Fault handling

`FaultRules` on the endpoint catch errors and let you shape the response. Without them, consumers get raw internal error structures that leak implementation detail.

```xml
<FaultRules>
    <FaultRule name="quota-exceeded">
        <Step><Name>RF-QuotaExceeded</Name></Step>
        <Condition>(fault.name = "QuotaViolation")</Condition>
    </FaultRule>
</FaultRules>
<DefaultFaultRule name="default">
    <Step><Name>RF-GenericError</Name></Step>
    <AlwaysEnforce>true</AlwaysEnforce>
</DefaultFaultRule>
```

A consistent error contract across all proxies — same JSON shape, same fields — is worth defining once at the start of a programme. Retrofitting it across thirty proxies later is painful and consumers will already have coded against the inconsistent versions.

#### Design guidance

**Don't put business logic in the proxy.** API Management is for mediation, security and traffic control. Transformation of any complexity belongs in Cloud Integration behind it.

**Version in the base path** — `/v1/orders`. Header-based versioning is defensible but harder for consumers and for you.

**Decide the proxy-per-API granularity early.** One proxy per business API, not one per backend endpoint, and not one giant proxy for everything.

**Secure by default.** Every proxy gets, at minimum: key or token verification, threat protection, spike arrest. Make it a template rather than a per-proxy decision.

**Use KVMs for environment config**, not hardcoded target URLs — same transport argument as externalized parameters in Cloud Integration.

**Fronting Cloud Integration with API Management** is a common and sound pattern: the proxy handles consumer-facing auth, rate limiting, and the public contract; the iFlow handles orchestration and transformation. Configure the proxy's target as the CPI endpoint with its own credentials, and don't expose the CPI runtime URL to consumers directly.

---

## 10. Monitoring & Troubleshooting

*Source: `sap-integration-suite/references/monitoring-troubleshooting.md`*

### Monitoring and Troubleshooting Reference

Contents: [MPL statuses](#message-statuses) · [Log levels](#log-levels) · [Trace](#using-trace) · [Diagnostic method](#diagnostic-method) · [Error decoder](#error-decoder) · [Stores](#store-monitoring) · [OData APIs](#monitoring-apis) · [Performance](#performance-investigation)

#### Message statuses

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

#### Log levels

Set per iFlow in Monitor → Manage Integration Content, or at deployment.

| Level | Captures |
|---|---|
| None | Minimal, no MPL detail |
| Info | Default — MPL header, custom properties, step-level status |
| Debug | Adds more detail and script logging |
| Trace | Full exchange payload before and after every step |

Trace is the tool for "the payload isn't what I expect at step 7", and nothing else gives you that. It is also expensive: it captures every payload in full, so it is time-limited by design — it resets itself after a short window (around 10 minutes) and the captured data is retained for roughly an hour. Treat those as approximate and check current tenant behaviour.

Never leave Trace on in production. Beyond performance, it writes full payloads into monitoring where anyone with monitoring authorization can read them — a real data protection exposure if the payload contains personal data.

#### Using trace

1. Set the flow's log level to Trace.
2. Trigger the message (resend, or wait for the next poll).
3. Open the MPL entry → the trace view shows each step with the message before and after.
4. Reset the log level.

What to look for, in order:
- Where does the body stop looking like what you expect? That's your failing step, not necessarily the step that threw.
- Are the properties you expect actually set at the point they're read?
- Are headers present that should have been removed, or absent that should have been added?

The step that throws the exception is frequently downstream of the step that caused the problem. Trace lets you find the actual origin instead of debugging the symptom.

#### Diagnostic method

A repeatable order of investigation that avoids the common trap of guessing:

1. **Read the actual error text**, in full, including the causal chain. The root cause is usually in the last `Caused by:` clause, not the first line.
2. **Determine which side failed.** Did the flow fail internally, or did the receiver return an error? An HTTP status in the message means the call reached the receiver — that's a different investigation from a connection failure.
3. **Check whether it ever worked.** New interface → configuration or design. Previously working → something changed: certificate expiry, credential rotation, receiver deployment, payload variation, volume increase.
4. **Check whether it fails for all messages or some.** All → configuration. Some → data-dependent, so diff a failing payload against a working one.
5. **Check the boring things.** Is the flow deployed and started? Is the credential alias deployed *on this tenant*? Is the endpoint the one you think? A surprising share of incidents end here.
6. **Reproduce with Trace** if the cause still isn't clear.

#### Error decoder

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

#### Store monitoring

Under Monitor → Manage Stores:

- **Message Queues (JMS)** — depth, retry counts, dead-lettered messages. A growing queue depth is an early warning that the consumer can't keep up; a dead-letter count above zero means messages are being lost unless someone acts.
- **Data Stores** — entries, expiry, size. Data Stores that only grow are a leak: something writes and nothing deletes.
- **Variables** — current values, useful for confirming a last-run timestamp is what you think.
- **Number Ranges** — current values.

Check queue depth and dead-letter counts as part of any handover or health check. They're the metrics most likely to be silently unhealthy.

#### Monitoring APIs

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

#### Performance investigation

When a flow is slower than expected, the usual causes in rough order of frequency:

1. **Receiver latency** — the flow is fine, the target is slow. Trace timings show this immediately.
2. **Per-item calls inside a splitter** — 5,000 splits × one lookup each. Batch the lookup, or pre-load into a property.
3. **Sequential processing where parallel would do** — splitter parallel processing off. Turn it on if ordering doesn't matter and the receiver can take it.
4. **Large payload materialisation** — repeated `getBody(String)` on a 50 MB file across several scripts.
5. **Oversized OData queries** — `$expand` pulling associations you don't use, or no `$select`.
6. **Trace left enabled.**
7. **Node contention** — another flow on the same node consuming resources.

Step-level durations in the MPL are the fastest way to localise it. Establish where the time actually goes before optimising anything.

---

## 11. CI/CD & Transport

*Source: `sap-integration-suite/references/cicd-transport.md`*

### CI/CD and Transport Reference

Contents: [Landscape](#landscape) · [Transport options](#transport-options) · [What doesn't transport](#what-does-not-transport) · [Configure-only pattern](#the-configure-only-pattern) · [Git](#git-and-version-control) · [Pipelines](#pipeline-automation) · [Practices](#practices)

#### Landscape

A typical setup is three tenants — DEV, QA (or TEST), PRD — each a separate subaccount, each with its own runtime URL, security material and configuration values.

Some customers add a sandbox for prototyping and training. Some run a single tenant with package-level separation; this is cheaper and consistently regretted, because there's no way to test a change without risking production.

The unit of transport is the **integration package**, though individual artifacts can be moved.

#### Transport options

**1. Cloud Transport Management (cTMS) + Content Agent Service** — the SAP-endorsed route. The source tenant hands the content to the Content Agent Service, which creates a transport request in cTMS; approvers release it into the next node. Gives an audit trail and approval gates, which is usually what governance actually wants.

**2. CTS+** — routes Integration Suite content through an existing ABAP-based transport landscape. Chosen when the customer already runs CTS+ and wants integration content in the same change process as ABAP transports.

**3. Manual export/import** — download the package as a `.zip` from the source tenant, upload to the target. Fine for a two-tenant proof of concept, unacceptable as a production process: no audit trail, no approval, easy to import the wrong version.

**4. API-based / pipeline** — use the design-time OData API to pull the artifact from source and deploy to target from a CI/CD pipeline, with Git as the source of truth. Most control, most setup effort. See [Pipeline automation](#pipeline-automation).

Which to recommend depends on the customer's existing change management. If they have cTMS or CTS+ running for other BTP or ABAP content, extend it. If they're building a modern pipeline-driven landscape and have the skills to maintain it, go API-based. Don't propose a Jenkins pipeline to a team with no pipeline experience and no one to own it.

#### What does not transport

The recurring cause of "it worked in DEV" incidents. None of the following move with the package:

- **Security Material** — credentials, OAuth artifacts, keystore entries, known hosts, PGP keys. Must be created on each tenant with the *same alias names*.
- **Configured externalized parameter values** — the parameters transport; the values are per-tenant configuration and need setting after import.
- **JMS queues** — created on deployment, but existing queue contents obviously don't move.
- **Data Store / Variable / Number Range contents.**
- **Certificate-to-user mappings, role assignments, service keys.**
- **Cloud Connector configuration** — separate per subaccount.
- **API Management KVMs**, and API products/applications/developer registrations.

Maintain a per-tenant checklist of these. On a first transport to a new tenant, walking that list before deploying saves an afternoon of confusing 401s.

#### The configure-only pattern

The discipline that makes transports uneventful: after import, the only thing anyone touches on QA and PRD is the **Configure** screen. No editing of the flow, no adjusting a script "just for production".

This requires that every environment-specific value is externalized:

- Receiver hosts, ports, paths
- Credential alias names
- Queue names, Data Store names
- SFTP directories and file patterns
- Scheduler settings (poll intervals often differ per environment)
- Log level
- Toggle flags for optional behaviour

A flow that needs editing in production is a flow whose DEV and PRD versions have diverged, and from then on nobody can be sure what's actually running where.

#### Git and version control

Cloud Integration has built-in versioning per artifact, but it is not source control: no branching, no diff across artifacts, no link to a change request, no history once a package is deleted.

Practical approach:

- Export packages to Git as the system of record. The `.zip` unpacks to a readable tree — `.iflw` (BPMN2 XML), `.groovy`, `.xsl`, `.mmap`, `.wsdl`, `.xsd` — so scripts and stylesheets diff perfectly well.
- `.iflw` diffs are noisy (coordinates, generated IDs move when you drag a step), but still useful for spotting unintended changes.
- Store the per-tenant parameter values as versioned config files alongside, so a deployment is reproducible.
- Automate the export rather than relying on someone remembering.

For Groovy and XSLT specifically, keeping the canonical version in Git and pushing to the tenant is far better than treating the tenant as the master copy — it gives you real code review on the parts of the integration where bugs actually live.

#### Pipeline automation

The relevant APIs:

```
# Design-time
GET  /api/v1/IntegrationPackages
GET  /api/v1/IntegrationPackages('<pkg>')/IntegrationDesigntimeArtifacts
GET  /api/v1/IntegrationDesigntimeArtifacts(Id='<id>',Version='<ver>')/$value   # downloads zip
POST /api/v1/IntegrationDesigntimeArtifacts                                     # upload/update
POST /api/v1/DeployIntegrationDesigntimeArtifact?Id='<id>'&Version='<ver>'

# Runtime
GET  /api/v1/IntegrationRuntimeArtifacts('<id>')
GET  /api/v1/MessageProcessingLogs
```

Auth: OAuth 2.0 client credentials from a service key on the Process Integration Runtime service, with the appropriate roles (`WorkspacePackagesEdit`, `IntegrationOperationServer.deploy`, etc.). Write calls need a CSRF token — fetch with `X-CSRF-Token: Fetch` on a GET, reuse on the POST within the same session.

Tooling options:
- **Project "Piper"** — SAP's Jenkins shared library, with ready-made steps for uploading and deploying integration artifacts. Least custom code if you're on Jenkins.
- **GitHub Actions / Azure DevOps / GitLab CI** — straightforward to script against the APIs directly. Many teams do this with a few dozen lines of shell or Python.
- **Custom scripts** — fine, but factor the token handling and CSRF dance into one reusable module.

A realistic pipeline shape:

```
commit → validate (lint Groovy/XSLT, check no hardcoded hosts)
       → upload artifact to DEV
       → deploy to DEV
       → smoke test (POST a canned payload, assert response + MPL status)
       → on approval: upload + deploy to QA
       → regression tests
       → on approval: PRD
```

The smoke test step is what makes it worth building. A deployment that succeeds but produces a flow that fails on the first message is not a successful deployment, and the API tells you the difference.

#### Practices

**Version deliberately.** Cloud Integration artifacts carry a version; bump it for meaningful changes rather than editing in place, so you can identify what's running.

**Test data that isn't production data.** Copying production payloads into DEV for testing is common, convenient, and a data protection problem when the payload contains personal data. Mask it.

**Deploy during a window for stateful flows.** Redeploying a flow that's mid-processing can abandon in-flight messages. For high-volume async flows, drain the queue or accept the retry.

**Keep a deployment record.** Which artifact version, which tenant, when, by whom, for which change request. The API can generate this automatically; do that rather than relying on a spreadsheet.

**Document the rollback.** For most flows, redeploying the previous version is enough. For flows involving stateful stores or partially-consumed queues, it isn't — work out the answer before you need it.

**Review the transport list before a release.** The set of artifacts changed is rarely exactly what people remember. Diff the package against the target tenant.

---

## 12. Asset: iFlow Specification Template

*Source: `sap-integration-suite/assets/iflow-spec-template.md`*

### Interface Specification — [Interface ID] [Interface Name]

| | |
|---|---|
| **Interface ID** | INT-XXX-001 |
| **Version / Date** | 1.0 / YYYY-MM-DD |
| **Author** | |
| **Status** | Draft / Review / Approved |
| **Business owner** | |
| **Technical owner** | |

#### 1. Business context

What business process this supports and why it exists. Two or three sentences — enough that someone finding this in two years understands the purpose without asking around.

**Business trigger:** what causes a message to flow.

#### 2. Interface overview

| Attribute | Value |
|---|---|
| Sender system | |
| Receiver system(s) | |
| Direction | Outbound / Inbound / Bidirectional |
| Pattern | Synchronous / Asynchronous / Batch |
| Quality of service | Best effort / Exactly once / Exactly once in order |
| Trigger | Event / Scheduled / On demand |
| Frequency | |
| Expected volume | X msg/day, peak Y msg/hour |
| Typical / max payload size | |
| Business criticality | High / Medium / Low |
| Availability requirement | |

#### 3. Technical design

**Integration package:** 
**iFlow name:** 
**Runtime tenant(s):** 

##### Sender

| Attribute | Value |
|---|---|
| Adapter | |
| Endpoint / address | |
| Authentication | |
| Authorization (role) | |
| Message format | |

##### Receiver

| Attribute | Value |
|---|---|
| Adapter | |
| Endpoint (per environment) | DEV: / QA: / PRD: |
| Authentication | |
| Credential alias | |
| Timeout | |
| Connectivity | Direct / Cloud Connector (Location ID: ) |
| Message format | |

##### Processing steps

| # | Step | Type | Description |
|---|---|---|---|
| 1 | | | |
| 2 | | | |

Include a flow diagram if one exists.

#### 4. Message structure and mapping

**Source structure:** (reference WSDL/XSD/JSON schema/IDoc type)
**Target structure:**

| Source field | Target field | Rule | Mandatory | Default | Notes |
|---|---|---|---|---|---|
| | | | | | |

**Value mappings used:**

| Value mapping | Source agency/ID | Target agency/ID | Maintained by |
|---|---|---|---|
| | | | |

**Lookups / enrichment:** what is fetched from where, and what happens if the lookup returns nothing.

#### 5. Error handling

| Error type | Detection | Handling | Retry | Notification | Owner |
|---|---|---|---|---|---|
| Connection failure | | | | | |
| Authentication failure | | | | | |
| Mapping / validation error | | | | | |
| Business rejection | | | | | |
| Timeout | | | | | |

**Retry policy:** mechanism, max attempts, interval, backoff.

**Dead letter handling:** where failed messages go and who acts on them.

**Idempotency:** is the receiver idempotent? If not, what prevents duplicate processing, and what is the deduplication key?

**Alerting:** channel, recipients, content, deduplication.

**Reprocessing procedure:** step by step, including required authorizations.

#### 6. Security and data protection

- Transport security:
- Message-level security (signing/encryption):
- Credential aliases required per tenant:
- Certificates used, with expiry dates and renewal owner:
- Personal data in the payload: yes/no — if yes, what fields, retention, and logging restrictions
- Payload logging policy per environment:

#### 7. Configuration parameters

| Parameter | DEV | QA | PRD | Notes |
|---|---|---|---|---|
| Receiver_URL | | | | |
| Credential_Alias | | | | |
| LogLevel | DEBUG | DEBUG | NONE | |
| | | | | |

#### 8. Dependencies and prerequisites

- Upstream/downstream interfaces:
- Required security material per tenant:
- Cloud Connector mappings:
- Backend configuration (roles, RFC destinations, partner profiles):
- Third-party services:

#### 9. Testing

| # | Scenario | Test data | Expected result |
|---|---|---|---|
| 1 | Happy path | | |
| 2 | Missing mandatory field | | |
| 3 | Receiver unavailable | | |
| 4 | Duplicate message | | |
| 5 | Maximum volume / payload | | |
| 6 | Business rejection | | |

#### 10. Operations

- **Monitoring:** where to look, what normal looks like
- **Key metrics:** expected daily volume, acceptable failure rate, latency SLA
- **Known issues and workarounds:**
- **Escalation path:** L1 → L2 → L3 with contacts
- **Maintenance windows:**

#### 11. Change log

| Version | Date | Author | Change | Change request |
|---|---|---|---|---|
| 1.0 | | | Initial version | |

---

## 13. Asset: B2B Partner Onboarding Checklist

*Source: `sap-integration-suite/assets/b2b-partner-onboarding-checklist.md`*

### Trading Partner Onboarding Checklist — [Partner Name]

| | |
|---|---|
| **Partner name** | |
| **Partner ID (internal)** | |
| **Onboarding owner** | |
| **Target go-live** | |
| **Status** | Discovery / Build / Test / Parallel / Live |

#### 1. Discovery — collect before building

- [ ] Partner technical contact (name, email, phone, timezone, working hours)
- [ ] Partner business contact for content questions
- [ ] Escalation contact for production issues
- [ ] Document types in scope, each direction
- [ ] Standard, version and subset per document (e.g. EDIFACT ORDERS D.96A EANCOM, X12 850 005010)
- [ ] Partner's implementation guideline / specification document received
- [ ] Real sample files for every document type, both directions
- [ ] Expected volumes: daily average, peak, largest interchange
- [ ] Character encoding and any national character requirements
- [ ] Acknowledgement expectations per layer (transport / syntax / application) with SLAs
- [ ] Test window and production cutover date agreed
- [ ] Do they use a VAN? If so, which, and who holds the contract?

#### 2. Identifiers

| Context | Scheme / qualifier | Partner value | Our value |
|---|---|---|---|
| AS2 ID | | | |
| Interchange (UNB / ISA) | | | |
| Group (UNG / GS) | | | |
| Backend partner number | | | |
| GLN / DUNS | | | |

- [ ] Identifier schemes confirmed in writing, not inferred from samples
- [ ] Case and leading zeros confirmed exactly
- [ ] Alternative identifiers registered if the partner uses more than one

#### 3. Transport

**Protocol:** AS2 / AS4 / SFTP / VAN / HTTPS

##### If AS2
- [ ] Partner AS2 ID and URL received; ours provided
- [ ] Our signing certificate sent to partner; their verification confirmed
- [ ] Their encryption certificate received and imported
- [ ] Signing algorithm agreed (SHA-256 unless legacy)
- [ ] Encryption algorithm agreed (AES-256 unless legacy)
- [ ] Compression: on/off, and order relative to signing
- [ ] MDN mode: synchronous / asynchronous
- [ ] If asynchronous: our MDN receipt endpoint built and reachable from their network
- [ ] Signed MDN required?
- [ ] Duplicate detection (Message-Id check) enabled
- [ ] Firewall/IP allowlisting completed both directions

##### If SFTP
- [ ] Host, port, directory structure (in / out / archive / error)
- [ ] Authentication: key or password; keys exchanged
- [ ] Host key added to known hosts
- [ ] File naming convention documented
- [ ] Partial-write protection: done-file, temp extension, or change detection
- [ ] Post-processing behaviour agreed (move to archive, not delete)
- [ ] Retention on both sides

##### Certificates register

| Purpose | Subject | Issuer | Expiry | Owner |
|---|---|---|---|---|
| Our signing | | | | |
| Their encryption | | | | |
| TLS server | | | | |

- [ ] All expiry dates added to the renewal calendar with owners

#### 4. Content design

- [ ] MIG created or reused per document type, per direction
- [ ] Segments properly qualified
- [ ] Code lists restricted to partner-supported values
- [ ] Partner-specific extensions modelled
- [ ] MIG released and version recorded
- [ ] MIG documentation exported and shared with partner for confirmation
- [ ] MAG created; every proposed line reviewed against the partner spec
- [ ] Value mappings identified (partner codes ↔ internal codes) and owners assigned
- [ ] Canonical format used where applicable

| Document | Direction | MIG | Version | MAG | Version |
|---|---|---|---|---|---|
| | | | | | |

#### 5. Configuration

- [ ] Trading partner profile created with identifiers, systems, channels
- [ ] Agreement instantiated from template (template name: )
- [ ] Partner-specific values populated
- [ ] Number range defined for outbound control numbers (per partner, per direction)
- [ ] Agreement activated in test
- [ ] Backend configuration complete (partner profiles, output determination, EDI partner setup)

#### 6. Testing

##### Transport test (do this first, alone)
- [ ] Connectivity established
- [ ] Empty or trivial payload delivered end to end
- [ ] MDN / receipt returned successfully
- [ ] Reverse direction confirmed

##### Content test
| # | Scenario | Direction | Result | Date |
|---|---|---|---|---|
| 1 | Happy path, each document type | | | |
| 2 | Maximum realistic interchange size | | | |
| 3 | Deliberately invalid syntax — confirm rejection and CONTRL/997 | | | |
| 4 | Valid syntax, business rejection — confirm routing to business owner | | | |
| 5 | Duplicate message — confirm suppression | | | |
| 6 | Special characters requiring release/escape | | | |
| 7 | Acknowledgement timeout — confirm alert fires | | | |
| 8 | Partner-side resend after missing MDN | | | |

- [ ] Partner has confirmed acceptance of our output files in writing
- [ ] We have confirmed acceptance of their files

#### 7. Operations readiness

- [ ] Test/production indicator (ISA15 or equivalent) switched for production
- [ ] Alerting configured; recipients confirmed
- [ ] Acknowledgement timeout monitor in place
- [ ] Negative acknowledgement routing to a named business owner
- [ ] MPL custom properties set: partner ID, document number, control number
- [ ] Reprocessing procedure documented and tested
- [ ] Support team briefed; runbook handed over
- [ ] Interface specification completed and stored
- [ ] Entry added to the partner register with certificates and contacts

#### 8. Cutover

- [ ] Production agreement activated
- [ ] Parallel run period (if replacing an existing connection): from ___ to ___
- [ ] Output compared against incumbent — differences resolved
- [ ] Partner notified of go-live date and time
- [ ] Heightened monitoring for first ___ days, owner: ___
- [ ] Old connection decommissioned, date: ___
- [ ] Post-go-live review held

#### Notes and deviations
