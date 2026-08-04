# Trading Partner Onboarding Checklist — [Partner Name]

| | |
|---|---|
| **Partner name** | |
| **Partner ID (internal)** | |
| **Onboarding owner** | |
| **Target go-live** | |
| **Status** | Discovery / Build / Test / Parallel / Live |

## 1. Discovery — collect before building

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

## 2. Identifiers

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

## 3. Transport

**Protocol:** AS2 / AS4 / SFTP / VAN / HTTPS

### If AS2
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

### If SFTP
- [ ] Host, port, directory structure (in / out / archive / error)
- [ ] Authentication: key or password; keys exchanged
- [ ] Host key added to known hosts
- [ ] File naming convention documented
- [ ] Partial-write protection: done-file, temp extension, or change detection
- [ ] Post-processing behaviour agreed (move to archive, not delete)
- [ ] Retention on both sides

### Certificates register

| Purpose | Subject | Issuer | Expiry | Owner |
|---|---|---|---|---|
| Our signing | | | | |
| Their encryption | | | | |
| TLS server | | | | |

- [ ] All expiry dates added to the renewal calendar with owners

## 4. Content design

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

## 5. Configuration

- [ ] Trading partner profile created with identifiers, systems, channels
- [ ] Agreement instantiated from template (template name: )
- [ ] Partner-specific values populated
- [ ] Number range defined for outbound control numbers (per partner, per direction)
- [ ] Agreement activated in test
- [ ] Backend configuration complete (partner profiles, output determination, EDI partner setup)

## 6. Testing

### Transport test (do this first, alone)
- [ ] Connectivity established
- [ ] Empty or trivial payload delivered end to end
- [ ] MDN / receipt returned successfully
- [ ] Reverse direction confirmed

### Content test
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

## 7. Operations readiness

- [ ] Test/production indicator (ISA15 or equivalent) switched for production
- [ ] Alerting configured; recipients confirmed
- [ ] Acknowledgement timeout monitor in place
- [ ] Negative acknowledgement routing to a named business owner
- [ ] MPL custom properties set: partner ID, document number, control number
- [ ] Reprocessing procedure documented and tested
- [ ] Support team briefed; runbook handed over
- [ ] Interface specification completed and stored
- [ ] Entry added to the partner register with certificates and contacts

## 8. Cutover

- [ ] Production agreement activated
- [ ] Parallel run period (if replacing an existing connection): from ___ to ___
- [ ] Output compared against incumbent — differences resolved
- [ ] Partner notified of go-live date and time
- [ ] Heightened monitoring for first ___ days, owner: ___
- [ ] Old connection decommissioned, date: ___
- [ ] Post-go-live review held

## Notes and deviations
