# src/xrGame/ai/monsters/rats/rat_state_initialize.cpp

> The three rat states that need something set up at the moment they are entered: where to flee to, where the startle came from, and whether to re-roll a wander goal.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`../../../movement_manager.h`](../../../movement_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: three short entry routines

## Purpose

The rat's brain holds states as objects with an enter/run/leave contract; these are three of the
enters. They are split into their own file from the runs in
[`rat_state_activation.cpp`](rat_state_activation.cpp.md) purely by lifecycle phase, which is a
reasonable split and one a rebuild may keep or discard.

What they share is the *flee anchor*: two of the three compute the same thing — a point some
authored distance directly behind the rat's current facing — and then let the per-tick logic
steer at it. A rat does not flee *from* anything; it flees *ahead*, having first turned away.

## `init_state_under_fire`

**Contract** — entered when morale has dropped below normal. If the rat has no enemy, has heard
something recently, and that sound is current, place the flee anchor one authored flee-distance
straight ahead of where the rat is presently facing. Then point the goal at the anchor,
unconditionally.

```text
FUNCTION init_state_under_fire()
  IF no enemy AND a sound is recorded AND that sound is not stale
    anchor = my position + (unit vector along my current facing) * flee_distance
  goal = anchor                 # even if the condition above did not fire
```

**Invariants** — the anchor is **not** recomputed when the condition fails, so the goal is then
whatever the anchor was left at by a previous state. That is usually the nest, which means a
frightened rat with an enemy in view runs home rather than blindly forward — probably the
intent, but it is achieved by omission rather than by a branch, and a rebuild should make it
explicit.

The flee direction is the rat's *current facing*, not away from the threat. The turning away is
left to whatever startled it; this routine only supplies the distance.

## `init_free_recoil`

**Contract** — entered when a startling sound is heard. Stamps the startle clock, records where
the sound came from, and — if the rat has no enemy and the startle has not already expired —
places the same style of flee anchor ahead of its facing.

**Invariants** — the recorded sound position is stored and then **never read**. The rat does not
run away from the sound; it runs forward. Storing the source looks like the beginning of an
unfinished "flee away from the noise" and one of the commented-out blocks in
[`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) confirms that reading. As shipped it is dead state.

Note that the startle clock is stamped here from the *global* clock, while the expiry test in
[`rat_state_switch.cpp`](rat_state_switch.cpp.md) compares it against the rat's *last update*
time. The two are the same clock, but the comparison is therefore against when this rat last
ran rather than against now — so a settled rat, updated rarely, stays startled longer. Whether
that was intended is not recoverable; it does make a slow-updating rat panic for longer, which
reads correctly.

## `init_free_active`

**Contract** — entered when the rat starts or resumes wandering. If the goal countdown has run
out (which also re-arms it and picks a fresh goal), force maximum speed when the rat is beyond
its home radius, and in every case re-roll the speed.

```text
FUNCTION init_free_active()
  IF goal_countdown_expired()                    # re-arms the countdown and re-rolls the goal
    IF distance(me, home) > home_radius
      speed = safe_speed = max_speed             # ... and then immediately re-rolled below
    choose_new_speed()
```

**Notes** — the forced maximum speed is **overwritten on the next line** by the speed lottery,
which may pick the minimum. The intent — a rat that has strayed too far hurries home — is
plainly there and plainly does not happen. The equivalent code in
[`rat_state_activation.cpp`](rat_state_activation.cpp.md) has the same two lines in the correct
order (an if/else rather than a fall-through), so this one is a transcription slip during the
port from the old brain. A rebuild should make it an alternative, not a prelude.
