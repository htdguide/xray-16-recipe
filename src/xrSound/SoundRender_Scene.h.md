# src/xrSound/SoundRender_Scene.h

> Declares one sound world and the geometry, emitters and event queue it owns.

**Needs** — [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`Sound.h`](Sound.h.md)
**Used by** — [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) · [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) · [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md)
**Tier floor** — T2: a declaration surface; the file-image reads it implies live in the `.cpp`.

## Purpose

Declares the scene, implemented in [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md). It fills the
abstract scene interface from [`Sound.h`](Sound.h.md), which is what the game and the editor
actually hold.

## Exported units

- **`Scene`** — the world. Final: there is one implementation and the seam is at the interface
  above it, not here.
- **`play` / `play_at_pos` / `play_no_feedback`** — the three ways to start a sound; the third
  gives the caller no handle back.
- **`i_play`** — the shared emitter-creation step behind all three.
- **`stop_emitters` / `pause_emitters`** — broadcast; pause is depth-matched.
- **`set_geometry_env` / `set_geometry_som` / `set_geometry_occ`** — install the three geometry
  databases. The occlusion model is *borrowed* from the level (the same tree physics queries); the
  other two are built and owned here.
- **`set_user_env` / `get_environment`** — the reverb-region query and its script override.
- **`set_environment` / `set_environment_size`** — inert vendor-extension remnants.
- **`get_occlusion` / `get_occlusion_to`** — the listener-relative and point-to-point occlusion
  queries.
- **`set_handler`** — install the AI's ear.
- **`update`** — drain the hearing queue.
- **`object_relcase`** — detach every emitter from a dying game object.
- **`get_emitters` / `get_events`** — direct access used by the processor's update walk and by the
  emitters themselves. A rebuild should narrow these to the two operations that are actually
  needed: reap-during-walk, and withdraw-my-events.
