# Trading Partner Management and Integration Advisor Reference

Contents: [When to use TPM](#when-to-use-tpm) · [Object model](#tpm-object-model) · [Type systems](#type-systems) · [MIGs](#message-implementation-guidelines) · [MAGs](#mapping-guidelines) · [Agreements](#agreement-templates-and-agreements) · [Runtime](#how-runtime-lookup-works) · [Partner Directory](#partner-directory) · [Onboarding](#partner-onboarding-workflow) · [Monitoring](#b2b-monitoring) · [Pitfalls](#pitfalls)

Names and screen locations in this area move between releases more than elsewhere in Integration Suite. Treat structure and concepts as reliable; verify exact artifact names and menu paths against the tenant.

## When to use TPM

TPM earns its complexity when you have **many partners exchanging similar documents with per-partner variations**. It replaces "clone the iFlow per partner" with generic flows plus per-partner configuration data.

Rough guidance:

- **Under ~5 partners, one or two document types, stable** — TPM is probably overhead. Parameterised iFlows plus Partner Directory may be enough.
- **10+ partners, or partners onboarding regularly, or several document types each** — TPM pays for itself quickly. Onboarding partner 30 should be configuration, not development.
- **Regulated/audited exchanges** — the agreement model gives you a documented, versioned record of what was agreed with whom, which is hard to reconstruct from thirty cloned iFlows.

The honest tradeoff: TPM front-loads effort. The first partner takes longer than a hand-built flow would. The break-even is typically somewhere between the fifth and tenth partner, sooner if document types multiply.

## TPM object model

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

## Type systems

Ships with the major standards — UN/EDIFACT across releases, ASC X12 across versions, SAP IDoc, SAP APIs. Subsets like EANCOM and ODETTE may need importing or building depending on the tenant.

You can create custom type systems from XSD, which is how proprietary partner formats and internal canonical structures enter the model. Worth doing for a canonical intermediate format if you're mapping many partners to one backend — see [Pitfalls](#pitfalls).

## Message Implementation Guidelines

A MIG takes a standard message (say EDIFACT ORDERS D.96A) and narrows it to what a specific partner or scenario actually uses:

- Set cardinality — make optional segments mandatory, or exclude unused ones entirely
- Restrict code lists to the values the partner sends
- Add qualifiers so repeated segments become distinguishable (the NAD segment appears many times; qualifying it as NAD+BY, NAD+SU, NAD+DP is what makes mapping tractable)
- Add documentation and examples per node
- Define custom fields for partner extensions

MIGs have lifecycle states — draft, then a released state once stable. Released MIGs are versioned; changing a released MIG means a new version, and consumers (MAGs, iFlows) must be pointed at it deliberately. This is a feature, not friction: a partner changing their format shouldn't silently alter what everyone else gets.

Outputs you can generate from a MIG: XSD (used by the EDI converters and validators), and human-readable documentation to send to the partner. That documentation is genuinely useful in onboarding — it's the specification you and the partner agree against.

**Qualification is the single highest-value activity in MIG work.** An unqualified MIG produces mappings full of positional guesswork that break when the partner reorders segments. Time spent qualifying up front is repaid every time the MIG is mapped.

## Mapping Guidelines

A MAG maps a source MIG to a target MIG — typically partner EDIFACT/X12 to SAP IDoc, or IDoc out to partner format.

The **proposal engine** suggests mappings based on aggregated usage across the Integration Advisor community. Proposals are a starting point with real hit rates on common pairs (ORDERS to ORDERS05, 850 to ORDERS05), not an answer. Review every proposed line — proposals reflect what is common, not what your partner's contract says.

Per-line you can apply functions, constants, conditions and value mappings, much as in Message Mapping. Complex logic still has limits; a custom function or a downstream iFlow step handles what the MAG can't.

MAGs can be used directly by a mapping step in an iFlow, or exported (XSLT) for use outside TPM. Direct reference keeps the guideline as the single source of truth, which is the point.

Like MIGs, MAGs are versioned and have lifecycle states.

## Agreement templates and agreements

The template is where the reusable design lives; the agreement is where the partner-specific values live. Getting this division right determines whether TPM actually saves you effort.

**Belongs in the template:** the set of activities, document types, transformation references, interchange defaults, acknowledgement requirements, error handling behaviour.

**Belongs in the agreement:** partner identifiers, communication channel selection, certificate references, control number ranges, any partner-specific deviations the template allows as parameters.

If you find yourself creating a new template per partner, the templates are carrying partner data that should be parameters. That's the TPM equivalent of cloning iFlows.

**Activation** is the step that publishes an agreement to the runtime. Until activated, the runtime lookup won't find it and messages fail with an agreement-not-found error. Deactivate and reactivate after changes — edits to an active agreement are not automatically live in all cases, and this catches people repeatedly.

Agreements are also the natural place to record the commercial and operational agreement: acknowledgement SLAs, test/production status, effective dates, contacts.

## How runtime lookup works

The generic B2B iFlows extract identifying information from the incoming message — sender identifier, receiver identifier, document type, and for X12/EDIFACT the version — and use that tuple to find the matching activated agreement. The agreement supplies the MIG for validation, the MAG for transformation, and the channel for the response.

SAP ships prebuilt generic flows for this (available from the content catalogue / Business Accelerator Hub); the package and flow names vary by release, so pull the current version rather than working from a blog post. You typically copy them into your own package and extend rather than modifying the shipped originals, so updates remain consumable.

The practical implication for troubleshooting: **agreement lookup failures are identifier problems.** When a message fails with no matching agreement, compare the identifier value and scheme in the message envelope character-for-character against the identifier configured on the partner profile. Case, leading zeros, and scheme qualifier are all common mismatches.

## Partner Directory

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

## Partner onboarding workflow

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

## B2B monitoring

Integration Suite provides a dedicated B2B monitor alongside the standard message monitoring, showing interchanges and the business documents within them, with acknowledgement status and links to the corresponding MPL entries.

What to watch operationally:

- Interchanges received vs. acknowledged — a growing gap means acknowledgements aren't flowing
- Negative acknowledgements — these need a business owner, not a support queue
- Per-partner failure rates — a single partner's rate jumping usually means they changed something
- Acknowledgement timeouts — nothing surfaces these unless you build a monitor for them

Correlating a partner's question ("what happened to PO 12345?") to a message requires a business key in the MPL. Set `SAP_ApplicationID` to the document number and add the partner ID and control number as custom header properties. Without it, answering partner queries means searching by timestamp.

## Pitfalls

**Cloning agreement templates per partner.** The signal that partner data has leaked into the template. Parameterise instead.

**Skipping the canonical format.** Mapping every partner format directly to every backend format is an N×M problem. Mapping partner → canonical → backend is N+M. With more than a handful of partners and more than one backend, the canonical intermediate is worth the extra hop. Build it as a custom type system so MIGs and MAGs work against it natively.

**Under-qualified MIGs.** Produces fragile mappings. Qualify segments properly the first time.

**Trusting proposals.** The proposal engine reflects community usage, not your partner's contract.

**Forgetting to activate, or to reactivate after changes.** The most common "why isn't my change live" in TPM.

**No acknowledgement timeout monitoring.** Missing acknowledgements are silent by nature.

**Certificates without an expiry register.** Every partner certificate needs a tracked expiry date and a named owner on both sides.

**Treating TPM as configuration-only.** You will still need custom iFlows for backend-specific handling, unusual partner requirements, and error routing. Plan for the escape hatches rather than being surprised by them.

**Testing with synthetic files only.** Partner-generated files contain things your generator never produces — unusual characters, unexpected optional segments, different line endings. Get real files before go-live.
