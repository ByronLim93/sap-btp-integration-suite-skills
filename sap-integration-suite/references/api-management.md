# API Management Reference

Contents: [Model](#object-model) · [Flow structure](#flow-structure) · [Policies](#policy-catalogue) · [Policy examples](#policy-examples) · [Variables](#variables) · [Fault handling](#fault-handling) · [Design guidance](#design-guidance)

SAP API Management is Apigee Edge under the hood, so Apigee documentation and policy semantics largely apply.

## Object model

**API Provider** — the backend system definition (host, port, auth, or a Cloud Integration/on-premise target).

**API Proxy** — the managed façade. Holds the endpoints, flows and policies. This is where the work happens.

**API Product** — a bundle of proxies (or specific resources) exposed as a purchasable/subscribable unit. Rate plans attach here.

**Application** — a consumer's registration against products; issues the API key.

**Developer / Developer Portal** — consumer-facing catalogue, self-service key management, documentation.

Consumers subscribe to Products, not Proxies. If a consumer can't call an API despite a valid key, check that the resource path is actually included in the Product — a Product with the wrong resource path returns 403 with a valid key, which is a confusing combination.

## Flow structure

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

## Policy catalogue

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

## Policy examples

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

## Variables

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

## Fault handling

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

## Design guidance

**Don't put business logic in the proxy.** API Management is for mediation, security and traffic control. Transformation of any complexity belongs in Cloud Integration behind it.

**Version in the base path** — `/v1/orders`. Header-based versioning is defensible but harder for consumers and for you.

**Decide the proxy-per-API granularity early.** One proxy per business API, not one per backend endpoint, and not one giant proxy for everything.

**Secure by default.** Every proxy gets, at minimum: key or token verification, threat protection, spike arrest. Make it a template rather than a per-proxy decision.

**Use KVMs for environment config**, not hardcoded target URLs — same transport argument as externalized parameters in Cloud Integration.

**Fronting Cloud Integration with API Management** is a common and sound pattern: the proxy handles consumer-facing auth, rate limiting, and the public contract; the iFlow handles orchestration and transformation. Configure the proxy's target as the CPI endpoint with its own credentials, and don't expose the CPI runtime URL to consumers directly.
