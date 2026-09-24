# src/xrGame/ai/monsters/monster_sound_defs.h

> The creature vocal vocabulary: which utterances exist, how they preempt one another, and how a species extends the set.

**Needs** — _(none)_
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`poltergeist_state_attack_hidden_inline.h`](poltergeist/poltergeist_state_attack_hidden_inline.h.md) · [`pseudodog.cpp`](pseudodog/pseudodog.cpp.md) · [`script_monster_hit_info_script.cpp`](../../script_monster_hit_info_script.cpp.md) · [`script_sound_action.h`](../../script_sound_action.h.md) · [`script_sound_action_script.cpp`](../../script_sound_action_script.cpp.md)
**Tier floor** — T3: three enumerations whose values are chosen so they can be combined by bit arithmetic

## Purpose

Every creature sound the game plays is identified by one of these tags, chosen by the brain
("threaten now") rather than by naming a file. The mapping from tag to a bank of samples is per
species, loaded from configuration; this file fixes the tag set, the priority ladder, and the
channel scheme that decides what may play over what.

The three enumerations are not independent — a request to the sound player combines a type, a
priority and a channel, and the numeric layout below is what makes that combination cheap.

## `SoundType`

```text
ENUM SoundType
  idle              # ambient vocalisation
  eat
  aggressive        # the attack call
  attack_hit        # the grunt at the moment of contact
  take_damage
  strike            # the swing itself
  die
  die_in_anomaly    # a distinct death cry, because anomaly deaths read differently
  threaten          # the warning before an attack
  steal             # the sound of sneaking
  panic
  idle_distant      # a second idle bank, chosen by distance to the listener

  script_custom     # base for per-species additions (bit 7)
  species_custom    # base for further additions (bit 14)
  none              # the absent tag
```

**Invariants** — the twelve base tags are consecutive small integers starting at one, so a
species that wants additional sounds — a dog's psi howl, say — takes `species_custom` and adds
a small index to it, and the result cannot collide with a base tag. Two extension bases exist
at bit 7 and bit 14 precisely so a species may extend and a script may extend independently.

**Notes** — `none` is all-bits-set, not zero, because zero is the base of the valid range.

The two idle tags are the only pair distinguished by listener distance rather than by
situation; the sound player chooses between them. That is why `idle_distant` is a tag and not a
parameter.

## `Priority`

```text
ENUM Priority
  critical = 1
  high     = 1 << 3     # 8
  normal   = 1 << 7     # 128
  low      = 1 << 15
```

**Invariants** — lower numbers win. The gaps are the point: a caller requests `high + 3` to sit
below plain `high` but above everything at `normal`, and there are seven such slots between any
two named levels and 120 between `high` and `normal`. So the four names are bands, not values,
and a species places its own sounds inside a band by offset.

**Notes** — the ladder is inverted relative to the intuitive reading ("high priority" is the
small number), which is a live trap for a rebuilder.

## `Channel`

```text
ENUM Channel
  base            = 1 << 7    # the ordinary channel; one sound at a time
  independent     = 1 << 15   # may play at any moment alongside anything
  capture_all     = all bits  # plays alone, silencing everything
```

**Invariants** — a sound on the base channel preempts or is preempted by other base-channel
sounds according to priority. An independent sound is exempt from that contest entirely, and
the comment in the source notes that each independent sound needs its own bit — so the number
of simultaneously independent sound kinds is bounded by the width of the field, not unlimited.
`capture_all` is the death cry's channel: nothing else may be heard.

## `DEFAULT_SAMPLE_COUNT`

**Contract** — sixteen. The number of samples a species' sound bank is loaded with when the
caller does not say otherwise. It bounds the variety within one tag; a bank with fewer files on
disk simply loads fewer.
