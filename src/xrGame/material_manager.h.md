# src/xrGame/material_manager.h

> Declares the per-creature footstep system: which material it is made of, which it is standing on, and the sounds that pairing produces.

**Needs** — [`material_manager.cpp`](material_manager.cpp.md) · [`material_manager_inline.h`](material_manager_inline.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md)
**Used by** — [`bloodsucker.cpp`](ai/monsters/bloodsucker/bloodsucker.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · [`entity_alive.h`](entity_alive.h.md) · [`material_manager.cpp`](material_manager.cpp.md) · [`material_manager_inline.h`](material_manager_inline.h.md) · [`step_manager.cpp`](step_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CMaterialManager`. See [`material_manager.cpp`](material_manager.cpp.md).

Exported units:

- `CMaterialManager` — the owning object and its movement control, this creature's own
  material index, the material last stood on, the step timer, and up to four concurrent
  footstep sounds.
- `Load` — read the creature's own material from configuration.
- `reinit` — reset per-spawn state and wire the movement control's material reporting.
- `reload` — present and empty.
- `set_run_mode` — switch between walking and running footsteps.
- `update` — advance the step timer and keep playing sounds positioned.
- `last_material_idx` · `self_material_idx` · `get_current_pair` — in
  [`material_manager_inline.h`](material_manager_inline.h.md).
