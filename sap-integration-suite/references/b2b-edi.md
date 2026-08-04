# B2B and EDI Reference

Contents: [EDIFACT structure](#edifact-structure) · [X12 structure](#x12-structure) · [EDI steps](#edi-steps-in-cloud-integration) · [Control numbers](#control-numbers) · [Acknowledgements](#the-acknowledgement-ladder) · [AS2](#as2) · [AS4](#as4) · [SFTP and OFTP](#sftp-and-other-transports) · [Inbound pattern](#reference-inbound-pattern) · [Outbound pattern](#reference-outbound-pattern) · [Failure modes](#b2b-specific-failure-modes)

For Trading Partner Management, MIGs and MAGs, see `tpm-integration-advisor.md`.

## EDIFACT structure

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

## X12 structure

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

## EDI steps in Cloud Integration

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

## Control numbers

UNB reference / ISA13 (interchange), UNG / GS06 (group), UNH / ST02 (message). Partners often require strict monotonic increment and will reject gaps or repeats.

Use **Number Ranges** (Monitor → Manage Stores → Number Ranges) rather than timestamps or random values. Define one range per partner per direction — a shared range across partners produces gaps from each partner's perspective, which some VANs and partners treat as lost messages.

Store the outbound control number against the message in a Data Store if you need to correlate an inbound 997/CONTRL back to what it acknowledges. Without that correlation, an acknowledgement telling you interchange 000000123 was rejected is useless.

Rollover: ISA13 is 9 digits, UNB reference is 14. Plan for wraparound on high-volume partners, and confirm the partner tolerates reset to 1.

## The acknowledgement ladder

Three independent layers, frequently conflated in requirements discussions. Ask which one the partner actually means:

| Layer | EDIFACT | X12 | AS2 | Means |
|---|---|---|---|---|
| Transport | — | — | MDN | The bytes arrived and decrypted/verified |
| Syntax | CONTRL | 997 / 999 | — | The EDI structure parsed and is valid |
| Application | APERAK, ORDRSP | 824, 855 | — | The business system accepted the content |

An MDN is not confirmation that anything was processed. A 997 is not confirmation the order was accepted. Interfaces where the partner believes a 997 means "order accepted" produce disputes months later; make the distinction explicit in the interface spec.

**Timing expectations** are part of the trading agreement: MDN typically synchronous or within minutes, 997/CONTRL commonly within an hour to a day, application acknowledgement per business SLA. Build the timeout monitor — an acknowledgement that never arrives is silent unless something is watching for it.

**Negative acknowledgements** must be routed to someone who can act. A 997 with AK501 = R for a rejected PO is a business event, not a technical error, and it belongs in front of the order desk rather than in a support queue.

## AS2

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

## AS4

ebMS3-based, gaining ground in energy markets and European regulated exchanges. Supports reliable messaging, WS-Security, and receipts (analogous to MDN). Configuration is driven by a PMode agreement between parties. Where AS2 has a handful of settings, AS4 has a formal profile — get the partner's PMode document rather than guessing.

## SFTP and other transports

Plenty of B2B still runs over SFTP, and VANs remain common.

- **File naming conventions** are part of the agreement — often encoding partner, document type and sequence. Getting the pattern wrong means files sit unprocessed.
- **Partial file reads**: use change detection or a done-file convention so you don't pick up a file mid-upload. This causes intermittent, unreproducible parse errors.
- **Post-processing**: move to an archive directory rather than delete, at least initially. Being able to reprocess yesterday's file is worth the storage.
- **VAN mailboxes** typically behave as SFTP or AS2 endpoints; the VAN handles partner-side routing, which simplifies onboarding but adds a dependency and a cost per kilocharacter.
- **OFTP2** is standard in European automotive; check current adapter availability for the tenant rather than assuming.

## Reference inbound pattern

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

## Reference outbound pattern

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

## B2B-specific failure modes

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
