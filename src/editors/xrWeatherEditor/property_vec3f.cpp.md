# src/editors/xrWeatherEditor/property_vec3f.cpp

> The vector row bound to the engine by a whole-vector getter and setter.

**Needs** — [`property_vec3f.hpp`](property_vec3f.hpp.md) · [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: owns native callback objects that must be released on a schedule the collector does not choose

## Purpose

The concrete half of the vector adapter: two callbacks, and the two operations [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md) derives everything else from.

## State

```text
RECORD VectorProperty EXTENDS VectorPropertyBase
  getter : callback() -> vector    # native; owned — a private copy
  setter : callback(vector)        # native; owned
```

## `construct(getter, setter)`

**Contract** — Takes private copies of the callbacks and seeds the base with the vector's **current** value, read through the getter during construction. Allocates.

**Notes** — Calling the engine before the row exists is a real ordering requirement, not an artefact: the base needs an initial vector to declare the three component rows' defaults, and the only source of one is the engine. A rebuild must therefore have the bound quantity readable at registration time — which the engine satisfies because it registers properties on records it has already populated.

## `release`

**Contract** — Frees the two callback copies, once. Release rule as in [`property_float.cpp`](property_float.cpp.md).

## `get_value_raw` · `set_value_raw`

**Contract** — Call the getter; call the setter. The vector record crosses by value, so both halves of the process must agree on its layout — see [`property_holder_include.hpp`](property_holder_include.hpp.md).
