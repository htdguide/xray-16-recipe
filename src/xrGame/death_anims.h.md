# src/xrGame/death_anims.h

> Declares the three-level table that picks a scripted death animation from the shot that caused it, implemented in [`death_anims.cpp`](death_anims.cpp.md) and [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md).

**Needs** — [`Include/xrRender/animation_motion.h`](../Include/xrRender/animation_motion.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`death_anims.cpp`](death_anims.cpp.md) · [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the three types that make up the death-animation selector. Substance is split:
loading and selection in [`death_anims.cpp`](death_anims.cpp.md), the seven kill-type
predicates in [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md).

Exported units:

- `rnd_motion` — a bag of interchangeable motions and a uniform pick from it. `setup` parses
  a space-separated motion-name list against a skeleton; `motion` draws one.
- `type_motion` — one *kind* of kill (shotgun, headshot, grenade …) holding four
  `rnd_motion` bags, one per incoming direction. `setup` parses a slash-separated list of
  four motion lists; `motion(direction)` draws from one bag; `dir` classifies a hit into a
  direction and a residual angle; `predicate` is the abstract test each kill type
  implements.
- `death_anims` — the whole table: seven kill types plus one fallback bag. `setup` loads it
  all from one configuration section; `motion` runs the predicates in order and returns the
  first match, or a fallback draw.
- `death_anim_debug` — a debug-build switch that logs every load and every selection. The
  selector has no other diagnostic, and it is the only way to find out why a death played
  the wrong animation.

**Notes** — `type_motion::not_definite` is a fifth direction value used only as the
"unclassified" sentinel and as the exclusive bound of the four real ones, so the direction
count and the enumeration cannot drift apart.
