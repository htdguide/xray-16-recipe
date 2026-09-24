# src/xrGame/death_anims.cpp

> Loads the per-creature table of death animations from configuration and, given a fatal hit, runs the kill-type predicates in order to pick one.

**Needs** — [`death_anims.h`](death_anims.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: table lookup and a weighted draw

## Purpose

When a creature dies the engine can either drop it as a ragdoll or play an authored death
animation and blend into the ragdoll afterwards. Which animation depends on *how* it died:
a shotgun at close range, a sniper round to the head, a grenade and a running headshot all
have their own authored sets. This file owns the lookup from "the hit that killed it" to
"the motion to play"; [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md) owns the
conditions.

The table is three levels deep, and the nesting is the design:

```text
death_anims  →  seven kill types, tried in a fixed order
  type_motion  →  four incoming directions (front, back, left, right)
    rnd_motion   →  a bag of interchangeable motions, drawn from uniformly
```

Each level answers a different question: *what killed it*, *from where*, and *which of the
several takes the animators authored*.

## State

```text
RECORD rnd_motion
  motions : list<MotionID>          # may be empty, which means "no animation"

RECORD type_motion                  # abstract; one per kill type
  anims : list<optional<rnd_motion>>  # exactly four entries, indexed by direction;
                                      #   an entry may be absent

RECORD death_anims
  anims     : list<type_motion>     # exactly seven, and the ORDER IS THE PRIORITY
  rnd_anims : rnd_motion            # the fallback bag, used when nothing matches
```

**Invariant** — the direction list is sized to four before anything is written into it, and
a direction the configuration did not supply stays absent rather than becoming an empty bag.
The distinction survives to the lookup: an absent direction yields an invalid motion, which
the caller treats as "this kill type does not apply after all" and keeps searching.

**Invariant** — every motion name in the configuration must resolve against the creature's
own skeleton. A name that does not is a hard failure at load, not a skipped entry. Death
animations are authored per model and a missing one means the section was attached to the
wrong visual.

## `death_anims::setup`

**Contract** — builds the whole table from one configuration section against one animated
skeleton. Clears first, so it is re-runnable. Requires the skeleton, the section name and
the configuration file to be present. Allocates the seven kill types and the fallback bag.

The seven types are installed under fixed keys, and **the slot each one occupies is not the
order it is written in the source**:

| slot (= priority) | configuration key | kill type |
|---|---|---|
| 0 | `kill_enertion` | running headshot: momentum carries the body forward |
| 1 | `kill_burst` | riddled with a burst |
| 2 | `kill_shortgun` | shotgun at close range |
| 3 | `kill_grenade` | explosion or fragment |
| 4 | `kill_sniper_headshot` | scoped rifle, head |
| 5 | `kill_sniper_body` | scoped rifle, body |
| 6 | `kill_headshot` | any other headshot |

The shuffle is deliberate and is the only place the priority is recorded. The general
headshot sits **last** so the two sniper cases and the running case get first refusal on
the same hit; the grenade case sits ahead of both sniper cases because an explosion is
unambiguous. A rebuild that lists them in source order gets a different game.

Finally, if the section names a `random_death_animations` list, it fills the fallback bag.

**Notes** — the configuration-key spelling `kill_shortgun` is a typo frozen by the shipped
data; it cannot be corrected.

## `type_motion::setup`

**Contract** — parses one kill type's configuration line. The line is a **slash-separated
list of up to four fields**, one per direction in enumeration order, and each field is
itself a **space-separated list of motion names** forming that direction's bag. A missing
line, an empty line or a line with fewer fields leaves the remaining directions absent.
Line length is capped at one kilobyte, checked rather than truncated.

```text
FUNCTION setup(skeleton, config, section, type_key)
  directions = four absent slots
  IF config has no line (section, type_key) THEN RETURN      # this kill type is unused
  line = that line
  FOR i IN 0 .. (number of '/'-separated fields in line) - 1
    directions[i] = new bag from field i, resolved against skeleton
```

**Invariants** — fields are assigned by *position*, so a configuration that wants to supply
only a "left" bag must still emit two empty fields before it. Nothing validates the field
count against four; a fifth field would write past the direction list.

**Notes** — an abandoned earlier design is visible in the file: directions keyed by name
(`front`, `back`, …) as separate configuration lines, and a direction-name token table that
now survives only to label debug messages. The shipped data uses the positional form.

## `rnd_motion::setup` · `rnd_motion::motion`

**Contract** — `setup` splits a space-separated list of motion names and resolves each one
against the skeleton into a motion handle, failing hard on any name the skeleton lacks.
`motion` draws one handle uniformly at random, or returns an invalid handle when the bag is
empty. The draw happens at *selection* time, not load time, so the same creature dying
twice in a save-scum loop plays different takes.

## `type_motion::motion`

**Contract** — draws from the bag for one direction. The direction must be one of the four
real ones; the unclassified sentinel is not accepted. An absent bag yields an invalid
handle.

## `death_anims::motion`

**Contract** — the selector. Given the dying creature and the fatal hit, returns the motion
to play and writes out a residual **angle**: how far the hit direction sits from the axis of
the chosen direction bucket, so the caller can rotate the body to line the authored
animation up with the real shot. Returns an invalid handle when there is nothing to play.

```text
FUNCTION motion(entity, hit) -> (MotionID, angle)
  angle = 0
  IF the table is empty THEN RETURN (fallback bag draw, 0)
  FOR EACH kill_type IN anims          # slot order = priority
    IF kill_type.predicate(entity, hit) yields a VALID motion THEN
      RETURN (that motion, the angle the predicate wrote)
  RETURN (fallback bag draw, 0)        # angle reset: the fallback is direction-agnostic
```

**Invariants** — a predicate that matches but whose direction bag is absent does **not**
stop the search; both the match and a valid handle are required. This is what lets the
configuration enable a kill type for some directions only and fall through to a lower
priority for the rest.

**Invariants** — the angle is reset to zero on the fallback path. A fallback animation is
authored neutral and must not be rotated.
