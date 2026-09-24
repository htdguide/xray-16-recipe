# src/xrCore/ChooseTypes.H

> The vocabulary a tool uses to ask the engine "let the user pick one of these", for twenty
> kinds of engine resource.

**Needs** — [`xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`fastdelegate.h`](fastdelegate.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a callback record and two enumerations. Nothing here touches memory
layout, timing or a device. The one T1-shaped detail — a platform drawing handle passed to
the thumbnail callback — is an artifact of the original's tool host, not a requirement.

## Purpose

An editor needs to ask the user to choose a resource — a texture, a sound, a skeleton, a
material, an animation — and the chooser that does it belongs to the tool, not the engine.
This file is the contract between them: it names the *kinds* of thing that can be chosen,
the flags that vary how a choice behaves, and the four callbacks through which a chooser
asks the engine to fill a list, reports what was picked, requests a thumbnail and announces
that it closed.

It lives in the engine's core rather than in a tool because the *lists* live here — only
the engine knows what materials or skeletons exist — while the presentation does not.
Nothing in the shipped engine consumes it; it is the engine's half of an editor interface
whose other half is not in this repository (the weather editor in chapter 29 uses the
separate property interface instead). A rebuild that ships no tools may delete it.

## State

```text
ENUM ChooseMode        # what kind of resource is being chosen
  custom               # the caller supplies the list itself
  sound_source, sound_environment
  object, group
  engine_shader, compiler_shader
  particle_effect, particle_system
  texture, texture_raw
  entity_type, spawn_item
  light_animation
  visual
  skeleton_animations, skeleton_bones, skeleton_bones_in_object
  game_material, game_animation, game_motions
# The order is an authored contract with the tool, not an internal detail: a tool and an
# engine built from different revisions must agree on it, and it has only ever been
# appended to.

ENUM ChooseFlags       # a bit set, combined freely
  multi_select         # more than one item may be returned
  allow_none           # an explicit "nothing" answer is offered, spelled "<none>"
  full_expand          # a hierarchical list opens fully expanded

RECORD ChooseItem
  name : text          # the identifier returned to the caller
  hint : text          # a longer description shown beside it
# Both are interned names, so a list of thousands costs no copying.

RECORD ChooseEvents
  caption   : text                  # defaults to "Select Item"
  on_fill   : (out list<ChooseItem>, context) -> nothing
  on_select : (ChooseItem, out list<Property>) -> nothing
  on_draw_thumbnail : (text, drawing_target, Rect) -> nothing
  on_close  : () -> nothing
  flags     : set of { animated }   # the preview animates rather than being a still
# Invariant: on_fill must be set before the chooser opens; the other three are optional and
# a chooser must tolerate any of them being absent.
```

## The choosing contract

**Contract** — a tool opens a chooser for one `ChooseMode`. The chooser calls `on_fill` to
obtain the candidate list, displays it, and on each selection calls `on_select` so the
engine can offer the item's properties for a preview pane. If a thumbnail is wanted the
chooser calls `on_draw_thumbnail` with the item's name and a region to draw into. When the
chooser is dismissed it calls `on_close` exactly once, whether or not anything was chosen.

**Invariants** — `on_fill` is called at least once per opening and may be called again if
the underlying set changes; the list it produces is owned by the caller and is valid only
until `on_close`; a name returned to the tool is meaningful only within the mode it was
chosen under, since two modes may use the same string for different things.

**Notes** — the "nothing chosen" answer is an in-band string rather than an absent value,
which means a resource genuinely named that string cannot be distinguished from no choice.
A rebuild should return an optional and delete the sentinel.

The four callbacks are the load-bearing content; that the original binds them with a
delegate type, and hands the thumbnail callback a platform drawing handle, is incidental —
the decision is *the engine supplies data and the tool supplies presentation*, and any
language expresses that with whatever callable it has.
