# src/xrGame/ai/monsters/control_animation.h

> Declares the creature animation component: the one place that actually starts clips on a creature's skeleton, and the records the rest of the creature talks to it through.

**Needs** — [`control_combase.h`](control_combase.h.md) · [`control_animation.cpp`](control_animation.cpp.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`control_animation.cpp`](control_animation.cpp.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager.h`](control_manager.h.md) · [`control_sequencer.cpp`](control_sequencer.cpp.md) · [`control_threaten.cpp`](control_threaten.cpp.md) · [`controller_animation.cpp`](controller/controller_animation.cpp.md)
**Tier floor** — T1: it drives the renderer's animation interface directly, holding raw handles to active blends and writing their playback speeds

## Purpose

Declares the surface implemented in [`control_animation.cpp`](control_animation.cpp.md), plus the two data records that make this component's boundary. The records matter more than the class: they are the contract by which a *decision* ("play this clip on the legs") becomes a *blend* on the skeleton, and they are what a rebuild must reproduce.

## `AnimationPart`

One of the three independently animated slices of a creature — whole body, legs, torso. Each holds the clip it should be playing, a handle to the blend actually playing it, a flag saying whether the two agree, and when it started.

**Invariants** — `actual` false means "the requested clip has not been started yet"; the frame update is what reconciles them. Only the whole-body slice's speed is ever overridden.

## `ControlAnimationData`

The component's published data block: a speed override and the three animation parts. Whoever holds animation control writes into this; the component reads it each frame. A negative speed means "use the clip's own speed".

## `AnimationSignalEventData`

The payload of a signal raised when playback crosses a marked time within a clip: which clip, at what fraction, and which signal identity. This is how the frame a blow lands on reaches the game layer.

## `ControlAnimation`

The component. Its exported surface:

- `reinit`, `reset_data`, `update_frame` — the component lifecycle and the per-frame reconciliation.
- `add_anim_event` — register a signal at a fraction of a named clip.
- `current_blend` — the whole-body blend handle, for callers that need to read playback position.
- `restart` — re-issue every active clip against a replaced visual.
- `freeze` / `unfreeze` — stop and resume playback without losing position.
- `motion_time` — a free-standing query: how long a clip runs at its authored speed.
- Three public flags set from playback-end callbacks, read and cleared by the component itself.
- `AnimationEventType` — the signal identities; only "hit" has a meaning outside this component.
