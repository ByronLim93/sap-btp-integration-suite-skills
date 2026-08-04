# Groovy Scripting Reference

Contents: [Signature](#script-signature) · [Message API](#message-api) · [Logging](#logging-to-the-mpl) · [Platform services](#platform-services) · [Recipes](#recipes) · [Pitfalls](#pitfalls) · [Performance](#performance) · [Testing](#testing)

## Script signature

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

## Message API

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

## Logging to the MPL

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

## Platform services

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

## Recipes

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

## Pitfalls

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

## Performance

Scripts run per message, and inefficiency compounds at volume:

- Don't parse the full payload if you only need one value — XPath in a Content Modifier is cheaper.
- Avoid regex on multi-megabyte strings.
- Don't do lookups (HTTP, Data Store) inside a per-item loop when a single batched call would do.
- Compile-once patterns: define `Pattern` as a static field rather than compiling per invocation in a hot path.
- Prefer streaming over string materialisation for large files.

## Testing

There's no built-in unit test runner in the Web UI. Practical options:

- **Simulation** in the iFlow editor — run a section of the flow with a sample payload without deploying. Good first check.
- **Local testing** — the script API classes can be stubbed. Keep business logic in plain Groovy classes with no `Message` dependency, and have `processData` be a thin adapter. That logic is then testable in any Groovy environment with Spock or JUnit.
- **Trace level** on a deployed flow shows the exchange before and after each step, which is the fastest way to confirm what the script actually received.

Structuring scripts so the testable part is separable is worth the small extra effort on anything with real logic in it.
