# src/editors/xrWeatherEditor/property_string_shared_str.cpp

> A text row aliased onto an interned-text slot, with every read and write routed through the engine so the store's accounting stays correct.

**Needs** — [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`engine_include.hpp`](engine_include.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — reached through its declarations in [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md); callers name that, not this file.
**Tier floor** — T2: aliases an engine-owned handle whose representation the editor must not touch directly

## Purpose

The reference-bound text adapter. It exists as its own type, rather than reusing the scalar alias, because the engine's text fields are not plain character buffers: they are handles into a shared, reference-counted store. Assigning one is not a memory write, so the alias cannot be poked the way a real or a whole number can.

## State

```text
RECORD InternedTextProperty
  engine : Engine                # the facade that can read and write interned handles
  slot   : alias to InternedText # the engine field being edited; NOT owned,
                                 # and its representation is opaque to this layer
```

## `construct(engine, field)` · `release`

**Contract** — Records the facade and the aliased slot. Allocates nothing. Release does nothing — there is no foreign resource here to free, only an alias — but the two release paths still converge on the same empty body, because the type participates in the same lifetime protocol as its siblings.

## `GetValue`

**Contract** — Asks the engine for the characters behind the handle and returns a copy as presentation text. Never reads the handle's representation directly.

## `SetValue`

**Contract** — Converts the grid's string to a native buffer, asks the engine to install it into the slot, frees the buffer. The engine performs the interning: it finds or creates the shared entry, adjusts the counts on the old and new entries, and stores the resulting handle.

```text
FUNCTION SetValue(text)
  buffer = native_copy_of(text)
  engine.assign_interned(buffer, slot)   # engine owns interning and refcounting
  free(buffer)
```

**Notes** — Routing through the facade rather than manipulating the handle is the load-bearing decision, and it is why the editor's abstract engine interface carries two otherwise inexplicable text methods. The alternative — letting the editor half link against the store's implementation — would put reference-count updates on both sides of the runtime boundary, where a missed release leaks every texture name the artist ever typed.

The consequence for a rebuild: wherever one side owns a shared, counted resource, the other side gets *operations* on it, never its representation.
