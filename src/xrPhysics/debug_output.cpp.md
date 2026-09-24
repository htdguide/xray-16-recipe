# src/xrPhysics/debug_output.cpp

> The debug port's do-nothing filling, installed so that physics never has to ask whether
> anyone is listening.

**Needs** — [`debug_output.h`](debug_output.h.md)
**Used by** — [`debug_output.h`](debug_output.h.md)
**Tier floor** — T3: an interface implemented with empty bodies.

## Purpose

[`debug_output.h`](debug_output.h.md) declares a wide interface and a module-level slot
pointing at the current implementation. This file provides the implementation that slot starts
out holding: every drawing call does nothing, every counter is a private field handed out by
reference, every query returns an empty answer, and the tracked object's name is the literal
text `none`.

The decision is the **null object**, and it is worth stating plainly because it is the only
thing here: physics has several hundred debug call sites, and a null implementation means none
of them needs a null check. The engine replaces the slot at startup when debug tooling is
compiled in; until then, and forever in a build without it, the no-op stands.

## State

```text
# the counters exist only so that the by-reference accessors have something to
# return; nothing reads them
RECORD NullDebugOutput
  mask_1, mask_2 : bit set            # both permanently empty, so every
                                      # "should I draw this" test is false
  tries_num, saved_tries_for_active_objects, total_saved_tries,
  reused_queries_per_step, new_queries_per_step,
  bodies_num, joints_num, islands_num, contacts_num : int
  collision_damage_to_display : real
```

**Invariants** — both masks are empty, so physics short-circuits before building any of the
data a draw call would need. That is what makes the no-op genuinely free rather than merely
cheap: the expensive part of debug drawing is assembling the geometry, and the mask test
happens first.

**Notes** — the counter fields are uninitialised in the original. Nothing reads them, so it
does not matter, but it is the kind of detail a rebuild should simply not reproduce.

The whole file, like its header, is compiled out entirely when debug tooling is absent. In
that configuration the slot, the interface and all the call sites vanish together.
