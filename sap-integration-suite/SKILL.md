---
name: sap-integration-suite
description: Expert guidance for SAP Integration Suite — Cloud Integration (CPI) iFlows, Groovy, message mappings, XSLT, adapters, B2B/EDI with Trading Partner Management, API Management, security material, monitoring and CI/CD transport. Use whenever the user mentions SAP CPI, Cloud Integration, iFlow, Integration Suite, API Management, SAP BTP integration, message mapping, Groovy for SAP, MPL, ProcessDirect, IDoc/OData/SFTP/SuccessFactors adapters, or exception subprocess. Use it just as strongly for B2B and EDI work — EDIFACT, ANSI X12, EANCOM, ODETTE, AS2, AS4, MDN, CONTRL, 997/999, ORDERS, DESADV, INVOIC, 850/855/856/810, EDI splitter, control numbers, Trading Partner Management, TPM, trading partner agreements, Integration Advisor, MIG, MAG, Partner Directory, partner onboarding, VAN connectivity — even when SAP is not named, since EDI questions here almost always concern Integration Suite. Also use for interface specs, error-handling design, and transporting artifacts between DEV/QA/PRD tenants.
---

# SAP Integration Suite

Guidance for designing, implementing, reviewing and troubleshooting integrations on SAP Integration Suite (SAP BTP). The focus is Cloud Integration (formerly CPI), with supporting material for API Management and the surrounding lifecycle.

## How to use this skill

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

## What you can and can't produce

You cannot build a deployable `.iflw` artifact from scratch reliably — the underlying BPMN2 XML is verbose, version-specific and easily corrupted. Be honest about that boundary rather than emitting plausible-looking XML the user will waste an afternoon on.

What you *can* do well, and should offer:

- **Step-by-step build instructions** for the Web UI — palette element, tab, field, exact value. This is what most users actually want.
- **Groovy scripts, XSLT, mapping logic, policy XML, Camel Simple expressions** — real, deployable code.
- **Design and review** — pattern selection, error-handling strategy, code review against the checklist below.
- **Troubleshooting** — read an MPL error, name the likely cause, give the fix.
- **Documentation** — interface specs, runbooks, mapping sheets.

If the user has exported artifacts (`.zip` of an iFlow, `.iflw`, `.mmap`, `.xsl`, `.groovy`, `.wsdl`, `.edmx`), read them from disk. An iFlow ZIP unpacks to `src/main/resources/` containing `scenarioflows/integrationflow/*.iflw` (BPMN2 XML), `mapping/`, `script/`, `xsd/`, plus `META-INF/MANIFEST.MF` and `resources.cnt`. Parsing that XML to *review* an existing flow is realistic and useful — generating one is not.

## Core design principles

These come up in nearly every review. Apply them proactively rather than waiting to be asked.

**Properties for internal state, headers only when they must leave the flow.** Headers are propagated to receiver adapters as HTTP/protocol headers. Anything internal — flags, counters, correlation keys — belongs in exchange properties. If a header genuinely must reach the receiver, it also has to be listed under Runtime Configuration → Allowed Header(s); this silently drops headers otherwise and is the single most common "my header disappeared" ticket.

**Externalize everything environment-specific.** Hosts, credential aliases, queue names, directories, addresses. Use `{{ParameterName}}`. A flow with a hardcoded QA hostname cannot be transported, and finding that out during a go-live cutover is a bad time.

**Never log or attach payloads containing personal or credential data.** Attachments in the MPL are visible to anyone with monitoring authorization and are retained for days. Log identifiers and counts, not the full payload — or gate payload logging behind an externalized `LogLevel` parameter that is `NONE` in production.

**Stream where possible, buffer only when necessary.** Reading the body as a String materialises the whole message in memory. On large payloads use streaming-capable steps, split early with the General Splitter, and avoid multiple Groovy steps that each re-read the body.

**One flow, one responsibility.** Prefer several small iFlows connected by ProcessDirect over one flow with a dozen branches. Smaller flows deploy independently, trace more cleanly, and let you reuse common logic (logging, error handling, token retrieval).

**Design for redelivery from the start.** Ask early: what happens if this runs twice? Most consumers are not idempotent. See `references/error-handling.md`.

## Naming conventions

Consistency matters more than the specific scheme, but if the user has none, propose this:

- Package: `<Domain>_<Purpose>` — e.g. `Finance_VendorMasterReplication`
- iFlow: `<Sender>_to_<Receiver>_<BusinessObject>_<Direction>` — e.g. `S4HANA_to_Ariba_PurchaseOrder_Out`
- Steps: verb-first and descriptive — `Map PO to cXML`, `Enrich with Vendor Data`, not `Groovy Script 1`, `Content Modifier 3`
- Externalized parameters: `<Scope>_<Name>` — `Receiver_Ariba_URL`, `Sender_Credential_Alias`
- Properties: `prop_` prefix or camelCase, consistently — makes them greppable in trace
- Credential aliases: `<ENV>_<System>_<Type>` — `PRD_ARIBA_OAUTH`

Default step names are a real cost: an MPL trace full of "Content Modifier 4" is unreadable at 2am during an incident.

## Standard iFlow skeleton

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

## B2B scenarios

B2B/EDI has its own shape and its own failure modes. When a request involves trading partners, EDI formats or AS2, read `references/b2b-edi.md` and `references/tpm-integration-advisor.md` rather than reasoning from the generic patterns above.

Three things worth establishing early in any B2B conversation, because they change the entire design:

**How many partners, and how fast is that growing?** Under about five stable partners, parameterised iFlows plus Partner Directory are usually enough. Beyond ten, or with regular onboarding, Trading Partner Management pays back the front-loaded setup effort.

**Which acknowledgement do they actually mean?** MDN, CONTRL/997, and an application acknowledgement are three different layers, and "we need confirmation the order was received" doesn't say which. Conflating them produces disputes months later.

**Direct or via a canonical format?** Partner-to-backend mapping is an N×M problem; partner → canonical → backend is N+M. With several partners and more than one backend, the intermediate hop is worth it.

Two habits that prevent most B2B incidents: persist the raw interchange before splitting it, because the original bytes are the only useful evidence in a partner dispute; and separate transport testing from content testing during onboarding, because debugging a certificate problem and a mapping problem at once takes several times longer than doing them in sequence.

## Review checklist

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

## Working style for this domain

The user is usually a consultant or developer under delivery pressure. Give the concrete answer first — the script, the config value, the likely root cause — then the reasoning. Skip the "SAP Integration Suite is a cloud-based platform…" preamble entirely.

When something depends on tenant configuration, release version, or the specific adapter version, say so explicitly rather than guessing. SAP changes adapter behaviour between versions and a confidently wrong field name costs the user more time than an honest "check the adapter version — this moved in 1.6".

If the user describes a requirement, ask about the things that actually change the design and are commonly left unstated: sync or async, expected volume and payload size, error handling expectations (fail fast vs. retry vs. dead letter), and whether the receiver is idempotent. One or two of these, not an interrogation.
