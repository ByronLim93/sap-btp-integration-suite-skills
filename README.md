# SAP BTP Integration Suite Skills

A [Claude Code / Claude Agent Skill](https://code.claude.com/docs/en/skills) providing
expert guidance for **SAP Integration Suite** on SAP BTP — Cloud Integration (CPI),
API Management, B2B/EDI with Trading Partner Management, and the surrounding
delivery lifecycle.

## What's in here

```
sap-integration-suite/
├── SKILL.md                    # Entry point: routing table, design principles, review checklist
├── references/                 # Loaded on demand, one topic each
│   ├── iflow-design.md         # iFlow structure, steps, adapters, Simple expressions
│   ├── groovy.md               # Groovy scripting: Message API, MPL logging, recipes, pitfalls
│   ├── mapping.md              # Message Mapping, XSLT, contexts, node functions, format conversion
│   ├── b2b-edi.md              # EDIFACT/X12, EDI steps, AS2/AS4, acknowledgements, control numbers
│   ├── tpm-integration-advisor.md  # Trading Partner Management, MIG/MAG, Partner Directory, onboarding
│   ├── error-handling.md       # Exception subprocess, retry, idempotency, QoS, alerting
│   ├── security-connectivity.md    # Security material, OAuth, certs, Cloud Connector, principal propagation
│   ├── api-management.md       # API proxies, policies, rate limiting, fault handling
│   ├── monitoring-troubleshooting.md  # MPL statuses, trace, error decoder, monitoring APIs
│   └── cicd-transport.md       # Tenant landscape, transport options, Git, pipelines
└── assets/                     # Starter artifacts to copy into your own work
    ├── iflow-spec-template.md          # Technical interface specification template
    └── b2b-partner-onboarding-checklist.md  # End-to-end trading partner onboarding checklist
```

The skill uses **progressive disclosure**: `SKILL.md` carries a routing table, and
each reference file is self-contained and loaded only when the task calls for it —
so context stays lean while deep material is a click away.

## What it helps with

Ask Claude anything about SAP Integration Suite and this skill sharpens the answer:

- **Cloud Integration (CPI)** — iFlow design, step selection, adapters (OData, SOAP,
  HTTP, SFTP, IDoc, SuccessFactors, ProcessDirect, JMS), headers vs. properties,
  externalization, decoupling patterns.
- **Code** — deployable Groovy scripts, XSLT, message-mapping logic, Camel Simple
  expressions, API Management policy XML.
- **B2B / EDI** — EDIFACT, ANSI X12, EANCOM, AS2/AS4, MDN and CONTRL/997
  acknowledgements, control numbers, EDI splitting, Trading Partner Management,
  Integration Advisor (MIG/MAG), Partner Directory, partner onboarding.
- **Resilience** — exception subprocesses, retry and idempotency, EO/EOIO,
  dead-letter and reprocessing, alerting.
- **Security & connectivity** — security material, OAuth flows, keystore and
  certificates, Cloud Connector, principal propagation.
- **API Management** — proxy design, policy catalogue, rate limiting, fault handling.
- **Operations** — reading an MPL trace, decoding runtime errors, monitoring APIs.
- **Lifecycle** — transports between DEV/QA/PRD, Git, MTAR, CI/CD pipelines.
- **Documentation** — interface specs, onboarding checklists, runbooks.

### Scope, honestly

The skill won't fabricate a deployable `.iflw` artifact from scratch — the underlying
BPMN2 XML is verbose, version-specific, and easily corrupted. Instead it gives
**step-by-step Web UI build instructions** (palette element → tab → field → value),
real deployable code, design and review against a checklist, and troubleshooting.
It can also **read and review** exported artifacts you have on disk (`.zip` iFlow
exports, `.iflw`, `.mmap`, `.xsl`, `.groovy`, `.wsdl`, `.edmx`).

## Installing the skill

Copy the `sap-integration-suite/` directory into a skills location Claude reads:

**Personal (all projects):**

```bash
mkdir -p ~/.claude/skills
cp -r sap-integration-suite ~/.claude/skills/
```

**Project-scoped (this repo / team, checked into version control):**

```bash
mkdir -p .claude/skills
cp -r /path/to/sap-integration-suite .claude/skills/
```

The skill triggers automatically whenever a request touches SAP CPI, Cloud
Integration, iFlows, Integration Suite, API Management, message mapping, Groovy for
SAP, or B2B/EDI topics — including EDI work where SAP isn't named explicitly.

## License

Provided as-is for use with SAP Integration Suite delivery work.
