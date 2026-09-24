# src/editors/xrWeatherEditor/property_vec3f_base.cpp

> A vector row is one value with three editable faces: the whole triple as text, and three component rows that each read-modify-write it.

**Needs** — [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_float.hpp`](property_float.hpp.md) · [`property_float_limited.hpp`](property_float_limited.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: builds native callback objects bound to a managed object through a native trampoline that must keep it reachable

## Purpose

Everything a vector property does except actually reaching the engine. The concrete adapters supply only whole-vector read and write; this file derives the component rows, the nesting, and the conversion to and from the presentation value from those two operations.

## State

```text
RECORD VectorPropertyBase
  container  : PropertyContainer    # holds exactly three rows: x, y, z
  components : ComponentBridge      # native trampoline; see below

RECORD ComponentBridge              # native
  owner : rooted reference to VectorPropertyBase
```

**Invariant** — The vector has no per-component storage. Every component read goes to the engine and every component write is a whole-vector write. There is exactly one copy of the value, and the engine holds it.

## `construct(initial)`

**Contract** — Builds the nested container, creates the trampoline, and registers three real rows — "x", "y", "z", all in a category named for components — each bound to the trampoline's matching accessor pair with a nudge step of `0.01`. The initial vector seeds the three rows' declared defaults. Allocates.

```text
FUNCTION construct(initial)
  container  = new PropertyContainer(owned_by: self)
  components = new ComponentBridge(self)      # native, roots self

  FOR EACH axis IN [x, y, z]
    getter = bind(components, read axis)
    setter = bind(components, write axis)
    container.add_property(
      PresentationSpec { name: axis, declared: real, category: "components",
                         description: axis + " component", default: initial[axis],
                         converter: real_converter },
      accessor_bound_real(getter, setter, step: 0.01))
```

**Notes** — Two things here are decisions, not mechanics.

*Why a trampoline at all.* The binding contract — the getter/setter pair the whole property layer speaks — is declared on the native side of the process. A managed object's method cannot be handed across it directly, so binding a component row to this adapter requires a native object whose methods forward to it, and that native object must hold the managed adapter in a way the collector honours, or the adapter can be reclaimed while three rows still point at it. The trampoline is that object. A rebuild in a single runtime deletes it; a rebuild that still spans two runtimes needs the same construct under some other name, and needs to answer the same question of who keeps whom alive.

*Why `0.01` and not the `0.05` used for a standalone real.* A vector component is almost always a direction or a per-axis factor near unity, where a `0.05` nudge is a visible jump. The finer step is a statement about what these quantities are, not about vectors in general.

## `release`

**Contract** — Frees the nested container, once, whether triggered by the grid or by the collector.

**Notes** — The trampoline is not freed. It is a native allocation reachable only from this adapter, so once the adapter goes it is unreachable and leaks — one small allocation per vector row, per document reopen. It is stated here rather than silently fixed because a rebuild must decide the ownership deliberately: the trampoline has to outlive every callback bound to it, which means it has to be released *after* the container that holds those callbacks, and that ordering is the actual requirement.

## `GetValue`

**Contract** — Returns the nested container, so the grid renders the row as expandable. The textual form the row displays is not produced here — the converter asks the adapter for the raw vector and formats it ([`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md)).

## `SetValue`

**Contract** — Takes a presentation vector, converts it field by field into the engine's vector record, and writes the whole thing through `set_value_raw`. Reached when the user types all three numbers into the parent row.

## `x` · `y` · `z`

**Contract** — Read the whole vector, replace one field, write the whole vector back. Each is one engine read and one engine write.

```text
FUNCTION set_component(axis, value)
  current       = get_value_raw()
  current[axis] = value
  set_value_raw(current)
```

**Notes** — Read-modify-write rather than a partial update is what lets the engine keep sole ownership of the value and run whatever it runs on a vector write. The cost is that a drag on one component issues two engine calls per frame of mouse movement, which the editor can afford.

## `get_value_raw` · `set_value_raw`

**Contract** — Demanded of the implementor: read the engine's vector, write the engine's vector. No clamping, no conversion, no notification — those belong to the layers above.
