# src/editors/xrWeatherEditor/property_holder_integer.cpp

> Registers whole-number properties in five shapes, including the one where the stored number is an index into a list the engine rebuilds on demand.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_integer.hpp`](property_integer.hpp.md) · [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_integer_limited.hpp`](property_integer_limited.hpp.md) · [`property_integer_limited_reference.hpp`](property_integer_limited_reference.hpp.md) · [`property_integer_enum_value.hpp`](property_integer_enum_value.hpp.md) · [`property_integer_enum_value_reference.hpp`](property_integer_enum_value_reference.hpp.md) · [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md) · [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md) · [`property_integer_values_value_reference.hpp`](property_integer_values_value_reference.hpp.md) · [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md) · [`property_converter_integer_enum.hpp`](property_converter_integer_enum.hpp.md) · [`property_converter_integer_values.hpp`](property_converter_integer_values.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: builds managed presentation records over native callback pairs and native field aliases

## Purpose

Ten registration overloads — five shapes across the two binding flavours. Whole numbers in the weather document are mostly not quantities but selections: which sky texture, which sound set, which of an authored list of things.

## State

Stateless.

## `add_property` (whole number)

**Contract** — As in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md): presentation record plus adapter, registered with the container, nothing returned.

| shape | what the stored number means | adapter family | converter |
|---|---|---|---|
| plain | a quantity | whole number | none |
| range-limited | a quantity clamped to `[min, max]` | clamped whole number | none |
| named set | one of an authored `(number, label)` list | whole-number choice | enumerated-choice |
| fixed label list | an **index** into a label list fixed at registration | whole-number selection | indexed-selection |
| live label list | an **index** into a label list the engine regenerates per query | whole-number selection, list re-read | indexed-selection |

**Notes** — The two list shapes look alike and decide different things.

In the *named set*, the number the engine stores is meaningful on its own — a mode, a flag value — and the labels only name the values it may take. Choosing "linear" stores whatever number was authored against that label.

In the *label list*, the number the engine stores has no meaning except as a position: it is an index, and the labels are the list. Choosing the third label stores `2`. This makes the list's ordering part of the document's meaning — reordering the list silently repoints every value stored against it. That is a hazard a rebuild inherits with the file format and cannot fix by choosing better types.

The *live* variant exists because some of those lists are not known when the property is registered. The set of weather names, for instance, depends on what is on disk and changes while the editor runs. So instead of a snapshot the engine supplies a pair of callbacks — produce the list, and say how long it is — and the adapter asks again every time the grid needs the list. The cost is that the index the user picked can be invalidated by a list that changed underneath; the adapters clamp rather than fail.
