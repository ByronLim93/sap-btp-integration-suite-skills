# CI/CD and Transport Reference

Contents: [Landscape](#landscape) · [Transport options](#transport-options) · [What doesn't transport](#what-does-not-transport) · [Configure-only pattern](#the-configure-only-pattern) · [Git](#git-and-version-control) · [Pipelines](#pipeline-automation) · [Practices](#practices)

## Landscape

A typical setup is three tenants — DEV, QA (or TEST), PRD — each a separate subaccount, each with its own runtime URL, security material and configuration values.

Some customers add a sandbox for prototyping and training. Some run a single tenant with package-level separation; this is cheaper and consistently regretted, because there's no way to test a change without risking production.

The unit of transport is the **integration package**, though individual artifacts can be moved.

## Transport options

**1. Cloud Transport Management (cTMS) + Content Agent Service** — the SAP-endorsed route. The source tenant hands the content to the Content Agent Service, which creates a transport request in cTMS; approvers release it into the next node. Gives an audit trail and approval gates, which is usually what governance actually wants.

**2. CTS+** — routes Integration Suite content through an existing ABAP-based transport landscape. Chosen when the customer already runs CTS+ and wants integration content in the same change process as ABAP transports.

**3. Manual export/import** — download the package as a `.zip` from the source tenant, upload to the target. Fine for a two-tenant proof of concept, unacceptable as a production process: no audit trail, no approval, easy to import the wrong version.

**4. API-based / pipeline** — use the design-time OData API to pull the artifact from source and deploy to target from a CI/CD pipeline, with Git as the source of truth. Most control, most setup effort. See [Pipeline automation](#pipeline-automation).

Which to recommend depends on the customer's existing change management. If they have cTMS or CTS+ running for other BTP or ABAP content, extend it. If they're building a modern pipeline-driven landscape and have the skills to maintain it, go API-based. Don't propose a Jenkins pipeline to a team with no pipeline experience and no one to own it.

## What does not transport

The recurring cause of "it worked in DEV" incidents. None of the following move with the package:

- **Security Material** — credentials, OAuth artifacts, keystore entries, known hosts, PGP keys. Must be created on each tenant with the *same alias names*.
- **Configured externalized parameter values** — the parameters transport; the values are per-tenant configuration and need setting after import.
- **JMS queues** — created on deployment, but existing queue contents obviously don't move.
- **Data Store / Variable / Number Range contents.**
- **Certificate-to-user mappings, role assignments, service keys.**
- **Cloud Connector configuration** — separate per subaccount.
- **API Management KVMs**, and API products/applications/developer registrations.

Maintain a per-tenant checklist of these. On a first transport to a new tenant, walking that list before deploying saves an afternoon of confusing 401s.

## The configure-only pattern

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

## Git and version control

Cloud Integration has built-in versioning per artifact, but it is not source control: no branching, no diff across artifacts, no link to a change request, no history once a package is deleted.

Practical approach:

- Export packages to Git as the system of record. The `.zip` unpacks to a readable tree — `.iflw` (BPMN2 XML), `.groovy`, `.xsl`, `.mmap`, `.wsdl`, `.xsd` — so scripts and stylesheets diff perfectly well.
- `.iflw` diffs are noisy (coordinates, generated IDs move when you drag a step), but still useful for spotting unintended changes.
- Store the per-tenant parameter values as versioned config files alongside, so a deployment is reproducible.
- Automate the export rather than relying on someone remembering.

For Groovy and XSLT specifically, keeping the canonical version in Git and pushing to the tenant is far better than treating the tenant as the master copy — it gives you real code review on the parts of the integration where bugs actually live.

## Pipeline automation

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

## Practices

**Version deliberately.** Cloud Integration artifacts carry a version; bump it for meaningful changes rather than editing in place, so you can identify what's running.

**Test data that isn't production data.** Copying production payloads into DEV for testing is common, convenient, and a data protection problem when the payload contains personal data. Mask it.

**Deploy during a window for stateful flows.** Redeploying a flow that's mid-processing can abandon in-flight messages. For high-volume async flows, drain the queue or accept the retry.

**Keep a deployment record.** Which artifact version, which tenant, when, by whom, for which change request. The API can generate this automatically; do that rather than relying on a spreadsheet.

**Document the rollback.** For most flows, redeploying the previous version is enough. For flows involving stateful stores or partially-consumed queues, it isn't — work out the answer before you need it.

**Review the transport list before a release.** The set of artifacts changed is rarely exactly what people remember. Diff the package against the target tenant.
