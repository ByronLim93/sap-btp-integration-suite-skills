# Mapping Reference

Contents: [Choosing an approach](#choosing-an-approach) · [Contexts](#contexts-the-core-concept) · [Node functions](#node-functions) · [Standard functions](#standard-functions) · [Custom functions](#custom-functions) · [XSLT](#xslt) · [Format conversion](#format-conversion) · [Value mappings](#value-mappings) · [Common problems](#common-problems)

## Choosing an approach

| Situation | Use |
|---|---|
| Structural XML→XML transformation, business-readable | Message Mapping |
| Complex recursion, grouping, sorting, or template reuse | XSLT |
| Simple field extraction or a single derived value | Content Modifier with XPath |
| Logic that isn't really mapping (calls, validation, state) | Groovy |
| JSON in, JSON out, no structural change | Groovy or JSON-native handling |

Message Mapping is the default because a functional consultant can read it. Reach for XSLT when the mapping starts needing more than three or four chained node functions to express one relationship — at that point the graphical version has stopped being readable and XSLT is genuinely clearer.

Message Mapping operates on XML. JSON payloads must be converted to XML first (JSON to XML Converter) and usually back afterwards.

## Contexts — the core concept

Almost every mapping bug traces back to contexts. A source field's values arrive not as a flat list but as a list grouped by the context node — the ancestor element the field is currently associated with.

Consider:

```xml
<Order>
  <Item><Material>A</Material></Item>
  <Item><Material>B</Material></Item>
</Order>
<Order>
  <Item><Material>C</Material></Item>
</Order>
```

With context `Order`, `Material` produces `[A, B]`, `[C]` — two contexts. With context set to the root, it produces `[A, B, C]` — one context. The same mapping produces a completely different target structure depending on which you pick, and the editor gives no warning.

Right-click a source field → **Context** to change it. When a target node repeats the wrong number of times, this is almost always why.

## Node functions

| Function | Effect | Typical use |
|---|---|---|
| `removeContexts` | Flattens all contexts into one | "I want all values regardless of parent" |
| `splitByValue` | Inserts context changes (each value / on value change) | Creating one target node per source value |
| `collapseContexts` | Keeps first value of each context | Deduplicating per group |
| `useOneAsMany` | Replicates a single value across many | Copying a header field onto every item |
| `sort` / `sortByKey` | Orders values (optionally by a second field) | Grouping items before mapping |
| `mapWithDefault` | Supplies a value when the context is empty | Guaranteeing a target node exists |
| `formatByExample` | Reshapes one list's contexts to match another's | Rebuilding grouping after flattening |
| `exists` | Boolean per context | Conditional target node creation |

The two that unlock most real scenarios: `useOneAsMany` (header-to-item replication) and `sortByKey` + `splitByValue` (grouping items by a key into parent nodes).

**Header onto items:** map `OrderNumber` → `useOneAsMany` with the item-level field as the count driver → target item's `OrderRef`.

**Group items by material:** `sortByKey(Material, Item)` then `splitByValue(value change)` on `Material` to create a group node per distinct material.

## Standard functions

Grouped as in the editor:

- **Arithmetic** — add, subtract, multiply, divide, round, formatNumber
- **Boolean** — and, or, not, equals, greater, if, ifWithoutElse, ifS
- **Constants** — constant, copyValue, sum, count, index
- **Conversions** — fixValues (code list), value mapping
- **Date** — dateTrans (format conversion), dateBefore, dateAfter, currentDate
- **Text** — substring, concat, replaceString, trim, toUpperCase, length, indexOf
- **Node functions** — as above

`ifWithoutElse` matters more than it looks: with plain `if`, an unmatched condition still produces an empty target node. With `ifWithoutElse`, the node is suppressed entirely. Optional target fields that must be absent rather than empty need this.

`dateTrans` uses Java date patterns. SAP dates arriving as `yyyyMMdd` and targets wanting `yyyy-MM-dd` is the most common instance.

## Custom functions

Message Mapping supports Groovy custom functions when the standard set falls short:

```groovy
def String buildKey(String company, String doc, MappingContext context) {
    return company + "-" + doc.padLeft(10, "0")
}
```

Signature options:
- **Simple** — scalar in, scalar out, called per value
- **Advanced (queue)** — `String[] input, Output output, MappingContext context` — gives access to the whole context queue, so you can implement custom node functions

`MappingContext` provides `getHeader(name)` and `getProperty(name)`, which is how a mapping reads values set upstream in the flow. Handy for injecting a run timestamp or a parameter without threading it through the payload.

Keep custom functions small and pure. A custom function making HTTP calls is a design smell — that belongs in a Content Enricher or a lookup step.

## XSLT

Full XSLT 1.0 and 2.0/3.0 support depending on the processor configured. Use it for:

- Recursive structures (BOMs, nested hierarchies)
- Grouping — `xsl:for-each-group` in XSLT 2.0 is dramatically simpler than sortByKey/splitByValue chains
- Reusable named templates across similar mappings
- Wholesale restructuring where field-by-field mapping would be hundreds of lines of clicking

Accessing exchange properties and headers requires declaring parameters:

```xml
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:param name="orderNumber"/>

  <xsl:template match="/">
    <Result>
      <Ref><xsl:value-of select="$orderNumber"/></Ref>
      <xsl:for-each-group select="//Item" group-by="Material">
        <Group material="{current-grouping-key()}">
          <xsl:for-each select="current-group()">
            <Qty><xsl:value-of select="Quantity"/></Qty>
          </xsl:for-each>
        </Group>
      </xsl:for-each-group>
    </Result>
  </xsl:template>
</xsl:stylesheet>
```

Parameters are populated from exchange headers/properties of the same name, configured on the XSLT Mapping step.

Always declare and use namespaces explicitly. Silently empty output from an XSLT is namespace mismatch in the overwhelming majority of cases — the source has a default namespace that the XPath doesn't account for.

## Format conversion

**XML to JSON** — watch for: single-element arrays collapsing to objects (the converter can't know an element is a collection from one instance), attributes getting `@` prefixes, and numeric/boolean values becoming strings. Configure "Suppress JSON Root Element" and the array handling options deliberately.

**JSON to XML** — JSON allows names XML doesn't (leading digits, special characters). The converter has a "JSON to XML Name Conversion" option; without it, invalid element names cause a parse failure downstream.

**CSV to XML / XML to CSV** — the converter handles delimiters, headers and field names. Quoted fields containing the delimiter are handled, but embedded newlines within quoted fields are a known trouble spot; if the source has them, pre-process.

**EDI** — the EDI to XML / XML to EDI converters plus Integration Advisor for the message implementation guidelines. B2B scenarios also involve Trading Partner Management for partner-specific configuration.

**Base64 / encoding** — encoder/decoder steps handle Base64 and GZip. Character encoding mismatches (payload declared UTF-8 but actually ISO-8859-1) produce mojibake or parse errors; the Content Modifier can set `CamelCharsetName`.

## Value mappings

For code list translations — plant codes, unit of measure, country codes, partner IDs. Maintained as a separate artifact and changeable without touching the mapping.

Structure: source agency + source identifier + value → target agency + target identifier + value.

Use them when the list changes independently of the interface logic, or is maintained by business users. Don't use them for two-entry conditions — a `fixValues` function is simpler and stays visible in the mapping.

Access from Groovy via `ValueMappingApi` (see `groovy.md`) when the lookup is needed outside a mapping step.

## Common problems

**Target node repeats the wrong number of times** — context setting on a source field. Check the context of every field feeding that node.

**Target field is empty but source has data** — namespace mismatch, or the field is in a different context than the target expects.

**"Cannot produce target element" / mapping fails on optional data** — a mandatory target field has no source value. Use `mapWithDefault` or make the source path optional-safe.

**Empty target nodes appearing** — `if` where you wanted `ifWithoutElse`.

**Mapping works in simulation but fails at runtime** — the runtime payload differs from the test file. Usually namespace declarations, encoding, or an envelope the test file lacked. Pull a real payload from the MPL and simulate with that.

**Performance degradation at volume** — mapping a very large document in one pass. Split before mapping.
