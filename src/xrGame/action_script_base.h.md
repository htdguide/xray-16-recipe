# src/xrGame/action_script_base.h

> Declares the bridge type that lets an engine-written action be installed in a planner that speaks to scripts. Behaviour is in [`action_script_base_inline.h`](action_script_base_inline.h.md).

**Needs** — [`action_base.h`](action_base.h.md) · [`action_script_base_inline.h`](action_script_base_inline.h.md)
**Used by** — [`action_script_base_inline.h`](action_script_base_inline.h.md) · [`stalker_base_action.h`](stalker_base_action.h.md) · [`stalker_planner.h`](stalker_planner.h.md)
**Tier floor** — T3: a declaration

## Purpose

A creature whose brain is scripted has a planner typed against the *game object facade* —
the handle scripts hold. An action written in the engine wants the concrete client object,
with its full interface, not the facade. This type sits between them: it is an action the
script-facing planner accepts, and it exposes the concrete object to its own subclass.

Substance is in [`action_script_base_inline.h`](action_script_base_inline.h.md).

Exported units:

- Two constructors — with and without an initial precondition and effect set — each taking
  the concrete object and deriving the facade from it.
- Two `setup` overloads: one taking the facade, which the planner calls, and one taking
  the concrete object, which the first resolves to.

**Notes** — the concrete object is held in a member that *shadows* the base's differently
typed one. That is deliberate — the base's member is the facade and this one is the real
object — but it means the same name denotes two different things depending on which type
you are looking through. A rebuild should name them apart.
