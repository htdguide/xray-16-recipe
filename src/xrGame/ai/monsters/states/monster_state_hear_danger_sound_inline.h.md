# src/xrGame/ai/monsters/states/monster_state_hear_danger_sound_inline.h

> Heard something frightening: run forty units away from it, turn to face the open ground, and then
> cower there indefinitely — unless the creature has a home region, in which case go home instead.

**Needs** — [`monster_state_hear_danger_sound.h`](monster_state_hear_danger_sound.h.md) · [`state_hide_from_point.h`](state_hide_from_point.h.md) · [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md) · [`state_data.h`](state_data.h.md)
**Used by** — [`monster_state_hear_danger_sound.h`](monster_state_hear_danger_sound.h.md)
**Tier floor** — T3: a transition table plus one vector construction

## Purpose

One of the two top-level sound reactions (the other is
[`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md), for sounds
that are merely interesting). It is what makes a level feel reactive from a distance: a shot fired
anywhere within earshot sends every unengaged animal in the area running, and the player sees the
wildlife move before they see anything else.

## State

`Stateless.` Everything is recomputed from the sound memory each time a leaf is entered.

## `reselect_state`

**Contract** — the go-home leaf wins whenever its own start condition holds, checked *first and on
every reselection*. Otherwise the sequence is: bolt, then face the open ground, then cower — and
cower absorbs.

```text
FUNCTION next_leaf(previous) -> leaf
  IF go_home.can_start()      RETURN go_home        # re-tested every time
  IF previous is none         RETURN bolt
  IF previous == bolt         RETURN face_open_ground
  RETURN cower
```

**Notes** — the go-home test being first and unconditional is the important structural choice.
A creature with an authored home region that is currently outside it, and whose danger is also
outside it, abandons the flee-and-cower sequence at any point and heads home instead. Territorial
animals therefore do not scatter across the level when startled; they converge. That single
re-test is what separates a pack that holds a lair from wildlife that simply runs.

`cower` is absorbing: the creature stays frightened in place until the brain's motivation weighting
selects some other top-level behaviour. As with the lost-contact search, this behaviour does not
conclude, it is concluded.

## `setup_substates`

**Contract** — fill the parameters for whichever leaf was selected. The go-home leaf configures
itself and appears here not at all.

```text
bolt:
  # flee not from the sound, but from a point one unit "behind" the creature
  away      = normalize(self.position - sound.position)
  point     = self.position + away * 1
  distance  = 40                      # how far counts as escaped
  gait      = run, accelerating, no braking, aggressive profile
  cover     = default band (10..30, searched within 20)
  voice     = silent                  # explicitly the dummy voice
  voice delay = section key "attack_sound_delay"

face_open_ground:
  action    = stand idle, scared posture
  time_out  = 2000 ms
  voice     = silent

cower:
  action    = stand idle, scared posture
  time_out  = none                    # runs until pre-empted
  voice     = silent
```

**Notes** — three things here are easy to get wrong in a rebuild.

*The flee anchor is displaced by one unit, not taken as the sound's position.* The flee leaf
retreats from a point; giving it the sound's own position would be the obvious reading. Instead the
anchor is placed one unit from the creature along the away-direction. The effect is that the flee
direction is pinned to the creature's *current* offset from the sound at the moment of entry and
does not swing as the creature moves — the retreat stays straight instead of curving. It also
behaves sanely when the sound arrives from the creature's own position, where an un-displaced
anchor would give no direction at all.

*The voice is explicitly the dummy channel in all three leaves.* A frightened animal makes no
noise, and the silence is authored as a real choice rather than as an omission — the leaves still
carry an authored repeat delay for a voice they never play. That delay (`attack_sound_delay`) is
the only per-creature number anywhere in this behaviour.

*Forty units is the flee distance, and it is much larger than the cover band the flee leaf searches
in.* The creature will pass several usable cover spots on its way; the distance, not the cover, is
what ends the leaf. Both numbers are hard-coded.

The second leaf's name says it faces an *unprotected* area — the direction with the least cover.
That is deliberate: a frightened animal turns to watch the open ground it might be charged across,
not the concealment it is hiding in. The direction comes from the cover component, and the leaf
that computes it is [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md).
