# src/xrGame/stalker_animation_names.cpp

> The fragment tables whose cartesian product is a stalker's entire animation set.

**Needs** — [`stalker_animation_names.h`](stalker_animation_names.h.md)
**Used by** — reached through its declarations in [`stalker_animation_names.h`](stalker_animation_names.h.md); callers name that, not this file.
**Tier floor** — T3: literal data

## Purpose

A stalker model ships several thousand motions whose names follow a rigid composition
rule. Rather than list them, the engine lists the *fragments* and builds names by
concatenation at load time: a visual's base name, then one fragment per nesting level.
This file is that list of fragments. It is a separate file purely so the tables have one
home; a rebuild may inline them into the loader.

The load-bearing content is not the strings — it is **the order**, because every other
file in the stalker animation and AI layer addresses these animations by numeric index
into these tables, never by name. Reordering a table silently rebinds gameplay behaviour
to a different animation.

## State

```text
RECORD FragmentTables
  state_names            : list<text>  # 3: crouched, normal, damaged-normal
  weapon_names           : list<text>  # 11: animation slot 0 .. 10
  weapon_action_names    : list<text>  # 15, see below
  movement_names         : list<text>  # 2: walk, run
  movement_action_names  : list<text>  # 4: forward, backward, left-strafe, right-strafe
  in_place_names         : list<text>  # 10 complete names, not fragments
  global_names           : list<text>  # 23, see below
  head_names             : list<text>  # 2 complete names: idle, talk
  food_names             : list<text>  # declared, never defined
  food_action_names      : list<text>  # declared, never defined
```

**Invariants**

- Every table is terminated by a sentinel empty entry, because the loader counts entries
  by scanning for it rather than carrying a length. A rebuild with sized lists deletes
  the sentinel and nothing else changes.
- A fragment that is a *prefix* ends with a separator character; a table whose entries are
  *complete* names (the in-place and head tables) does not. Which of the two a table is
  decides whether it may nest another table beneath it.
- The weapon table's eleven entries are the eleven **animation slots** an inventory item
  can declare. Slot 0 is "no item"; slot 2 is the one the special-danger and look-back
  variants are authored for, which is why several call sites test `slot == 2` explicitly.

## `weapon_action_names`

**Contract** — the fifteen per-weapon actions, in the order the rest of the codebase
indexes them:

| Index | Action | Who indexes it |
|---|---|---|
| 0 | draw | torso animation for weapon "showing" |
| 1 | attack (fire) | torso animation for firing |
| 2 | drop | unused by the torso selector |
| 3 | holster | torso animation for weapon "hiding" |
| 4 | reload | three sub-entries: begin, in-process, end |
| 5 | pick | unused by the torso selector |
| 6 | aim | the idle/aim family; sub-entries 0–6, see below |
| 7 | walk | free-state walking |
| 8 | run | free-state running |
| 9 | idle | free-state standing |
| 10 | prepare | |
| 11 | strap | two sub-entries: to-strapped, strapped-to-idle |
| 12 | unstrap | two sub-entries: to-unstrapped, unstrapped-to-idle |
| 13 | look-back left-strafe | walking backwards while glancing over a shoulder |
| 14 | look-back right-strafe | the mirrored variant |

The two movement indices 7 and 8 are addressed as `7 + movement_type`, which requires
walk and run to be adjacent *and* in the same order as the movement-type enumeration. That
coupling is invisible from either side and is the kind of thing a rebuild should make
explicit.

Within the aim family (index 6) the sub-entries mean: 0 standing, 2 walking, 3 running,
and 4–6 the *special danger move* variants of those three, used only when the creature is
tuned for it and carries a slot-2 weapon. A model without the special-danger variants has
a shorter sub-list, and the selector falls back to the ordinary aim rather than failing.

**Notes** — the two look-back fragments are misspelled in the shipped data ("beack"). The
misspelling is frozen: the strings must match the authored motion names exactly.

## `global_names`

**Contract** — animations that are not tied to a weapon: damage reaction, escape, the
stop-on-death pose, greeting, then eighteen critical-hit animations, then the panic stand.

**Invariants** — the eighteen critical-hit entries are **three severity tiers of six body
regions**, region-major within each tier and in the exact order of the critical-wound
enumeration. Entry 4 is therefore the first tier's head hit, which is what makes that
enumeration start at 4. A rebuild may keep the flat table or split it into
`tier × region`, but if it keeps it flat the arithmetic `4 + 6*tier + region` must hold.

**Notes** — the body-region fragments in the shipped names are misspelled too ("hend" for
hand); frozen for the same reason.

## `in_place_names`

**Contract** — ten complete leg animations for a stationary or jumping stalker: two idle
variants, four turn variants (right/left at two speeds), and four jump phases (begin,
airborne, land, land-variant). These are complete names because nothing nests below them.

## Dead entries

`food_names` and `food_action_names` are declared in the header and defined nowhere; no
code references them. They are the residue of an eating behaviour that was cut. A rebuild
should omit them.
