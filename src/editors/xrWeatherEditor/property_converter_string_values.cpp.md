# src/editors/xrWeatherEditor/property_converter_string_values.cpp

> Hands the grid the list of values a text row admits, and declares the list closed.

**Needs** — [`property_converter_string_values.hpp`](property_converter_string_values.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure presentation; it never touches engine memory

## Purpose

The bridge between a text row's published admissible set and the grid's notion of standard values. It holds no state and belongs to no particular row — the grid instantiates one per property *type*, not per property, and tells it which row it is acting for on every call.

## State

Stateless. This is forced, not incidental: one converter instance serves every row of its type, so anything it remembered would leak between rows.

## `GetStandardValues(context)`

**Contract** — Answers with the admissible set of the row the context names. Recovers the container from the context's subject and the row identity from the context's descriptor, asks the container for that row's adapter, and asks the adapter for its values. Fails if the row's adapter does not publish an admissible set — a configuration error at registration, not a runtime condition.

```text
FUNCTION GetStandardValues(context) -> list<text>
  container = context.subject        AS PropertyContainer
  row       = context.descriptor     AS row identity
  adapter   = container.property(row)
  RETURN (adapter AS publishes_admissible_set).values()
```

**Notes** — The two-step lookup — subject to container, descriptor to row, container plus row to adapter — is the whole reason this file exists rather than the converter simply holding the list. The grid's conversion protocol is per-type, and the only thing it passes that identifies the individual row is the context. Every converter and value editor in this directory opens with the same two steps; a rebuild whose grid passes the row's own object instead reduces each of them to one line.

Because the adapter is asked on every call, a live admissible set ([`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md)) is picked up without the converter knowing such a thing exists.

## `GetStandardValuesSupported`

**Contract** — Always true. Every row given this converter has a list.

## `GetStandardValuesExclusive`

**Contract** — Always true: the list is the whole admissible set, so the grid may present it as a closed choice rather than as suggestions.

## `CanConvertFrom`

**Contract** — Always false. Even the typing-permitted variant refuses conversion *from other value shapes*; the free-typed string arrives by the grid's own text path, which this answer does not close.
