# src/xrGame/script_game_object_trader.cpp

> The five facade methods that drive a trader's idle presentation: its body animation, its head animation, and the speech played over them.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`ai/trader/ai_trader.h`](ai/trader/ai_trader.h.md) · [`ai/trader/trader_animation.h`](ai/trader/trader_animation.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation only

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across,
and the smallest. A **trader** is a creature that does not move, fight or path: it stands
at its counter and its entire visible behaviour is an animation pair plus speech. These
five methods are how a dialogue script drives that.

All five follow the
[guarded-delegation pattern](script_game_object.cpp.md#the-guarded-delegation-pattern),
with a downcast to the trader type and a *log-and-return* on failure.

## State

`Stateless.`

## `set_trader_global_anim(name)` and `set_trader_head_anim(name)`

**Contract** — each selects one of the trader's authored animations by name: the body
animation it loops, and the head animation layered over it.

**Invariants** — the two are **independent layers**, not one animation. A trader can shake
its head while leaning on the counter, which is the entire point of separating them, and a
rebuild that collapses them into one animation set cannot reproduce the shipped dialogue
presentation.

## `set_trader_sound(sound, anim)`

**Contract** — plays a sound *and* switches the head animation in one call, so that lip and
head movement start on the same frame as the speech. Passing them separately would let a
frame slip between them, which reads as a dubbing error.

## `external_sound_start(sound)` and `external_sound_stop`

**Contract** — start and stop a sound that the trader's own animation system does not
own — a radio, a gramophone, a recording — routed through the trader so it is positioned at
the trader and stops when the trader does.

**Notes**

The error message on a failed downcast is the same text for all five and names neither the
object nor the method, unlike the convention in the other eight files. That is a wart; a
rebuild should name both, since "cannot cast to trader" on a level full of traders is not
actionable.
