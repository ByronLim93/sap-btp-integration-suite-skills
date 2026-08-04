# Interface Specification — [Interface ID] [Interface Name]

| | |
|---|---|
| **Interface ID** | INT-XXX-001 |
| **Version / Date** | 1.0 / YYYY-MM-DD |
| **Author** | |
| **Status** | Draft / Review / Approved |
| **Business owner** | |
| **Technical owner** | |

## 1. Business context

What business process this supports and why it exists. Two or three sentences — enough that someone finding this in two years understands the purpose without asking around.

**Business trigger:** what causes a message to flow.

## 2. Interface overview

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

## 3. Technical design

**Integration package:** 
**iFlow name:** 
**Runtime tenant(s):** 

### Sender

| Attribute | Value |
|---|---|
| Adapter | |
| Endpoint / address | |
| Authentication | |
| Authorization (role) | |
| Message format | |

### Receiver

| Attribute | Value |
|---|---|
| Adapter | |
| Endpoint (per environment) | DEV: / QA: / PRD: |
| Authentication | |
| Credential alias | |
| Timeout | |
| Connectivity | Direct / Cloud Connector (Location ID: ) |
| Message format | |

### Processing steps

| # | Step | Type | Description |
|---|---|---|---|
| 1 | | | |
| 2 | | | |

Include a flow diagram if one exists.

## 4. Message structure and mapping

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

## 5. Error handling

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

## 6. Security and data protection

- Transport security:
- Message-level security (signing/encryption):
- Credential aliases required per tenant:
- Certificates used, with expiry dates and renewal owner:
- Personal data in the payload: yes/no — if yes, what fields, retention, and logging restrictions
- Payload logging policy per environment:

## 7. Configuration parameters

| Parameter | DEV | QA | PRD | Notes |
|---|---|---|---|---|
| Receiver_URL | | | | |
| Credential_Alias | | | | |
| LogLevel | DEBUG | DEBUG | NONE | |
| | | | | |

## 8. Dependencies and prerequisites

- Upstream/downstream interfaces:
- Required security material per tenant:
- Cloud Connector mappings:
- Backend configuration (roles, RFC destinations, partner profiles):
- Third-party services:

## 9. Testing

| # | Scenario | Test data | Expected result |
|---|---|---|---|
| 1 | Happy path | | |
| 2 | Missing mandatory field | | |
| 3 | Receiver unavailable | | |
| 4 | Duplicate message | | |
| 5 | Maximum volume / payload | | |
| 6 | Business rejection | | |

## 10. Operations

- **Monitoring:** where to look, what normal looks like
- **Key metrics:** expected daily volume, acceptable failure rate, latency SLA
- **Known issues and workarounds:**
- **Escalation path:** L1 → L2 → L3 with contacts
- **Maintenance windows:**

## 11. Change log

| Version | Date | Author | Change | Change request |
|---|---|---|---|---|
| 1.0 | | | Initial version | |
