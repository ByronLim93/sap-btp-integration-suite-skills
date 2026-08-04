# Security and Connectivity Reference

Contents: [Security material](#security-material) · [Inbound auth](#inbound-authentication) · [Outbound auth](#outbound-authentication) · [OAuth flows](#oauth-flows) · [Certificates](#certificates-and-tls) · [Cloud Connector](#cloud-connector) · [Principal propagation](#principal-propagation) · [Message-level security](#message-level-security) · [Common errors](#common-connectivity-errors)

## Security material

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

## Inbound authentication

How callers authenticate *to* your iFlow's sender endpoint:

**Client certificate** — mutual TLS. The client cert's subject/issuer DN pair is mapped to a role in the tenant's Certificate-to-User Mappings. Strongest option and common in SAP-to-SAP scenarios.

**OAuth 2.0 client credentials** — the caller obtains a token from the tenant's token endpoint using a service key, then presents the bearer token. This is the standard approach for modern API consumers and what you should default to.

**Basic authentication** — user in the BTP identity provider with the required role. Acceptable for internal or legacy callers; increasingly restricted by SAP for new tenants.

Authorization is role-based. The default role for message sending is `ESBMessaging.send`. In production, create dedicated roles per consumer group rather than granting every caller the default role — otherwise any authenticated client can invoke any endpoint on the tenant. This is a genuinely common finding in security reviews.

The endpoint address configured on the HTTPS sender is a path suffix; the full URL is `https://<tenant-runtime-host>/http/<your-path>`.

## Outbound authentication

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

## OAuth flows

**Client Credentials** — machine-to-machine, no user context. The default for system integration.

**SAML Bearer Assertion** — the flow asserts a user identity to the target, letting the target apply that user's authorizations. Requires trust configuration on both sides and a signing key in the tenant keystore. Used for principal propagation to SAP backends.

**JWT Bearer** — similar, with a JWT instead of a SAML assertion.

**Authorization Code** — needs a browser and user consent. Not workable for unattended integration; if a requirement seems to need it, the actual answer is usually a long-lived refresh token obtained once out-of-band, or a different grant.

## Certificates and TLS

The tenant keystore holds:
- **Own certificates** (with private key) — used to identify the tenant to receivers, and to sign
- **Trusted certificates / root CAs** — used to validate receivers' certificates

Practical points:

- Server certificate validation requires the receiver's **full chain** — root and any intermediates — in the keystore. Missing intermediate is the most common cause of `SSLHandshakeException: unable to find valid certification path`.
- Certificate expiry is a scheduled outage waiting to happen. The tenant surfaces expiring certificates in monitoring; put a calendar reminder in independently. A production interface failing at 03:00 on renewal day is a classic.
- SAP rotates its own load balancer root certificates periodically, and consumers calling your tenant need the current root. Watch the relevant SAP Notes.
- Key pairs used for signing (SAML, PGP, message-level) should be documented with their purpose — keystores accumulate mystery entries nobody dares delete.

## Cloud Connector

The tunnel from BTP to on-premise systems. The iFlow addresses a **virtual host:port**, which Cloud Connector maps to the internal host:port.

Configuration checklist when a connection fails:

1. Is the Cloud Connector connected to the correct subaccount? (Green in the CC admin UI)
2. Is the system mapping present, with the right virtual host and protocol?
3. Are the **resources** exposed? Access control is path-based and defaults to nothing. A mapping with no accessible resource returns a failure that looks like a routing problem.
4. Does the path match — including the trailing-slash and sub-path settings?
5. For RFC: is the resource an exposed function module or prefix?
6. Is the on-premise system reachable *from the Cloud Connector host*? Test there, not from your laptop.

In the iFlow, set Proxy Type = `On-Premise` and Location ID if multiple Cloud Connectors serve the subaccount. Forgetting the Location ID with multiple connectors produces intermittent failures as requests hit the wrong one.

## Principal propagation

Carrying the calling user's identity through to the backend so the backend applies that user's authorizations, rather than everything running as a technical user.

Chain: caller authenticates to the iFlow → the flow obtains an assertion for that user → the receiver accepts it and establishes the user session. Requires trust configured at each hop (BTP subaccount trust, Cloud Connector principal propagation settings, backend certificate rules in `STRUST`/`SNC`), and the sender adapter must be configured to preserve the user context.

It is a genuinely fiddly setup and each hop has its own failure mode. When a customer asks for it, confirm it's actually required — often the real requirement is auditability of *which* user triggered a change, which can be satisfied by passing the user ID as a data field, at a fraction of the effort.

## Message-level security

Beyond transport security, for cases where the payload must be protected end-to-end or non-repudiation is needed:

- **PGP** — encrypt/decrypt and sign/verify steps, keys in the PGP keyrings. Standard in B2B file exchange.
- **XML Digital Signature / XML Encryption** — for SOAP and XML payloads.
- **Simple Signer / Verifier** — PKCS#7/CMS signatures.
- **AS2 / AS4** — protocol-level security with MDN receipts, for EDI partners.

Sign-then-encrypt is the conventional order. Whatever you choose, document the key exchange process — the operational cost of message-level security is almost entirely in key lifecycle, not in the flow configuration.

## Common connectivity errors

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
