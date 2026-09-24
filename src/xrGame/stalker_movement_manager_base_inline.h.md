# src/xrGame/stalker_movement_manager_base_inline.h

> Field access over the current and target movement records, plus two small predicates.

**Needs** — [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md)
**Used by** — [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md)
**Tier floor** — T3: field access.

## Purpose

Holds the accessors for the movement manager base. Most are one line and carry no decision;
what they do carry is a consistent rule about *which* of the two movement records they
touch, and that rule is the content of this page.

## The current/target split

**Contract** — every **setter** writes the *target* record; every **reader** returns the
*current* one, except the four readers explicitly named `target_*` and `target_params`.

**Invariants** — this asymmetry is the layer's contract with the AI above it. An action
says what it wants (target) and asks what is (current), and the gap between the two is what
the per-frame update closes. An action that read its own wish back would never notice that
the creature has not turned yet, has not stood up yet, has not reached walking speed yet.

A rebuild that collapses the two records loses every behaviour that depends on transitions
taking time — which, in this engine, is most of them.

## `set_body_state` and `set_mental_state`

**Contract** — write the target posture and target mental state, and **fail** if the result
would be a creature that is both at ease and crouched.

**Invariants** — the check is written into both setters so that neither order of assignment
can produce the forbidden pair. It is a hard failure rather than a clamp: the combination
has no animation, so silently correcting it would produce a creature in a pose nobody
authored, discovered much later and much further away.

**Notes** — a line that would also have invalidated the current path whenever the target
mental state changed is commented out, with a note that it is correct and was disabled
before a presentation for lack of time to fix the consequences. The consequence is real: a
creature that becomes alarmed keeps walking the path it built while relaxed, at the new
speeds, until that path completes. The combat planner works around it by clearing the path
at combat entry (see
[`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md)). A rebuild should do the
right thing here and delete the workaround.

## `turn_in_place`

**Contract** — is the creature rotating on the spot. True when a path is still pending, the
linear speed is zero, and the body's current heading differs from its target.

**Notes** — the three conditions together are the definition of the state: something to walk
to, not walking, not yet facing the right way. The animation layer reads this to select the
turning motions.

## `set_desired_direction` and the head speed

**Contract** — plain writes into the target record and into the head rotation state.
`danger_head_speed` sets how fast the head tracks while alarmed; it is a field rather than a
constant so that an action can slow a creature's head deliberately — a device the kill-wounded
sequence once used and no longer does.
