# src/editors/xrWeatherEditor/property_holder_boolean.cpp

> Registers boolean properties on a node, choosing between a plain checkbox and a two-label choice.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_boolean.hpp`](property_boolean.hpp.md) · [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md) · [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md) · [`property_boolean_values_value_reference.hpp`](property_boolean_values_value_reference.hpp.md) · [`property_converter_boolean_values.hpp`](property_converter_boolean_values.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: builds managed presentation records from native text and native callback pairs

## Purpose

Four of the property-registration overloads: a boolean, bound either by a callback pair or by an alias to an engine field, presented either as a raw truth value or as a choice between two authored labels ("clear"/"overcast" rather than false/true).

## State

Stateless — everything is written into the node's container.

## `add_property` (boolean)

**Contract** — Builds a presentation record and a value adapter and registers the pair with the node's container. Identifier, category and description are copied out of native text at this moment. Returns nothing usable. Allocates; does not block.

```text
FUNCTION add_property(identifier, category, description, default, binding, labels?)
  spec = PresentationSpec {
    name        = copy(identifier),
    declared    = bool,
    category    = copy(category),
    description = copy(description),
    default     = default,
    converter   = IF labels PRESENT THEN boolean_labels_converter ELSE none
  }
  adapter = CHOOSE by binding flavour and whether labels are present
  container.add_property(spec, adapter)
  RETURN none
```

| binding | labels | adapter |
|---|---|---|
| callback pair | absent | accessor-bound boolean |
| field alias | absent | reference-bound boolean |
| callback pair | two labels | accessor-bound labelled boolean |
| field alias | two labels | reference-bound labelled boolean |

**Notes** — Two decisions here run through every dispatch file in the directory and are stated once:

*The presentation flags are accepted and discarded.* The interface offers four per-property flags — read-only, notify-parent-on-change, mask-as-password, refresh-the-whole-grid-on-change — and every registration in this editor ignores all four. The grid's own defaults stand. A rebuild is free to honour them; nothing in the authored weather data depends on them, and nothing currently passes anything but the defaults.

*Registration returns nothing.* The interface promises a handle through which a caller could later change those same flags. This implementation never produces one, so the handle side of the interface is dead. A rebuild should either deliver the handle or delete it from the interface, but should not reproduce a promise no implementation keeps.

The label form exists because the authored data is a truth value while the vocabulary the artist works in is not. The labels are presentation only — nothing but the truth value is written back to the engine.
