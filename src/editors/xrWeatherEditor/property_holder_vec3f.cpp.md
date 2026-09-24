# src/editors/xrWeatherEditor/property_holder_vec3f.cpp

> Registers three-component vector properties, which the grid shows as one row that expands into three.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_vec3f.hpp`](property_vec3f.hpp.md) · [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md) · [`property_converter_vec3f.hpp`](property_converter_vec3f.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: converts between the engine's three-real vector record and the presentation layer's value type

## Purpose

Two registration overloads — vector by callback pair, vector by field alias. Vectors appear in a weather document where a quantity has a direction or a per-axis magnitude, and the authoring need is the same in both cases: type all three numbers at once, or nudge one.

## State

Stateless.

## `add_property` (vector)

**Contract** — As in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md). The declared type is the presentation layer's own three-real value, seeded from the engine's default, and the vector converter is always attached — it is what makes the row both expandable into components and editable as a single space-separated triple.

**Notes** — The component rows are not built here. They are built by the adapter's own base, which constructs a nested container of three real properties bound back to the whole vector — see [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md). The consequence worth stating: the vector has no per-component storage anywhere. Editing one component reads the whole vector, replaces one field, and writes the whole vector back. That keeps the engine as the single owner of the value, at the cost of a read-modify-write per keystroke.
