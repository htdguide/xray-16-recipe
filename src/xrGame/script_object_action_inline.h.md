# src/xrGame/script_object_action_inline.h

> The object channel's constructors and setters.

**Needs** — [`script_object_action.h`](script_object_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for [`script_object_action.h`](script_object_action.h.md). Each constructor is a run
of setter calls; each setter writes its field and clears the channel's completion flag.
The contracts are in that header's twin.

## Exported units

- construct from (object, action, burst size) — three setters in order.
- construct from (bone name, action) — two setters.
- construct from (action) — one setter.
- `set_object(bone_name)` — stores the bone name.
- `set_object_action(action)`, `set_queue_size(count)`.
- `initialize` — empty.

**Notes**

The completion flag is cleared by every setter, including one that writes the value already
present. That is deliberate: re-setting a field is the idiom a script uses to restart a
finished order without rebuilding it.

`set_object` taking an object rather than a bone name is not here — it needs to unwrap a
script facade and lives in
[`script_object_action.cpp`](script_object_action.cpp.md).
