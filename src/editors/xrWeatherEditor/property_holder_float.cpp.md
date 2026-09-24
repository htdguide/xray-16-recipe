# src/editors/xrWeatherEditor/property_holder_float.cpp

> Registers real-valued properties, and fixes the step size a drag or a spin applies.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_float.hpp`](property_float.hpp.md) · [`property_float_reference.hpp`](property_float_reference.hpp.md) · [`property_float_limited.hpp`](property_float_limited.hpp.md) · [`property_float_limited_reference.hpp`](property_float_limited_reference.hpp.md) · [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) · [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md) · [`property_converter_float_enum.hpp`](property_converter_float_enum.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: builds managed presentation records over native callback pairs and native field aliases

## Purpose

Six registration overloads for real-valued properties — the bulk of a weather keyframe, since nearly every authored quantity in a time-of-day frame is a real number. Three shapes: unconstrained, range-limited, and restricted to an authored set of named magnitudes.

## State

Stateless.

## `add_property` (real)

**Contract** — As in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md): builds a presentation record plus an adapter, registers both, returns nothing. Every real property is given the real-number converter, which is what makes the grid display and parse the value in the editor's chosen precision rather than the runtime's default.

| shape | binding | adapter | step |
|---|---|---|---|
| unconstrained | callback pair | accessor-bound real | `0.05` |
| unconstrained | field alias | reference-bound real | `0.05` |
| range-limited | callback pair | accessor-bound clamped real | `(max − min) × 0.0025` |
| range-limited | field alias | reference-bound clamped real | `(max − min) × 0.0025` |
| named set | callback pair | accessor-bound real choice | step suppressed |
| named set | field alias | reference-bound real choice | step suppressed |

**Notes** — The step size is the load-bearing number here, and the two rules differ for a reason.

An unconstrained real has no scale to normalise against, so it gets a fixed absolute step of `0.05`. That is a compromise: it is a sensible nudge for the quantities that dominate the weather document — factors and intensities that live near unity — and a uselessly small one for a quantity measured in hundreds. A rebuild that can ask the engine for a quantity's natural scale should.

A range-limited real *does* have a scale, so its step is a fraction of the range: `0.0025` of it, which is four hundred steps from minimum to maximum. Four hundred is the interesting choice — it is fine enough that dragging a slider-like control feels continuous and coarse enough that a keyboard repeat crosses the range in a few seconds.

A value restricted to a named set has no step at all; incrementing a choice is meaningless, so the adapters for that shape refuse it.
