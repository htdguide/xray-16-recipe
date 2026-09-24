# src/xrGame/ai/stalker/ai_stalker_space.h

> The stalker's vocabulary: thirty vocalisations, the bit algebra that decides which may interrupt which, and the weapon-class table that reconciles three games' worth of data.

**Needs** — _(none: a vocabulary header)_
**Used by** — [`ai_stalker.cpp`](ai_stalker.cpp.md) · [`ai_stalker_cover.cpp`](ai_stalker_cover.cpp.md) · [`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md) · [`ai_stalker_misc.cpp`](ai_stalker_misc.cpp.md) · [`ai_stalker_script.cpp`](ai_stalker_script.cpp.md) · [`sight_manager_target.cpp`](../../sight_manager_target.cpp.md) · [`stalker_animation_global.cpp`](../../stalker_animation_global.cpp.md)
**Tier floor** — T3: enumerations and a lookup, except that the bit values are frozen by the sound algebra

## Purpose

Three unrelated vocabularies share this file because all three are "what a stalker is made
of" and none is big enough for its own.

The interesting one is the **sound mask algebra**, which is the engine's answer to a problem
every talking-NPC game has: thirty lines, several of which want to play at once, and a
character with one voice. The usual answer is a priority number. This engine instead gives
each line a *bit mask*, and the sound player will only start a line whose mask overlaps the
currently-permitted set. That expresses "these lines are mutually exclusive but any of them
may cut this other group off" directly, which a scalar priority cannot.

## The vocalisations

```text
ENUM StalkerSound =
  die · die_in_anomaly · injuring · injuring_by_friend
  humming · tolls · wounded
  alarm · panic_human · panic_monster
  attack_no_allies · attack_allies_single_enemy · attack_allies_several_enemies
  backup · need_backup · detour
  search_no_allies · search_with_allies
  enemy_lost_no_allies · enemy_lost_with_allies
  grenade_alarm · friendly_grenade_alarm · throw_grenade
  running_in_danger
  kill_wounded · enemy_critically_wounded · enemy_killed_or_wounded
  script
```

A twenty-ninth value, *walking in danger*, is present only as a comment: its registration
and the branch that would play it are both disabled. A stalker moving carefully in danger is
silent; only running in danger is voiced.

## The mask algebra

```text
# three category bits, high in the word, are the actual algebra:
non_triggered = bit 31 | bit 30     # lines that are not a reaction to an event
no_humming    = bit 29              # "may cut off the idle hum"
no_danger     = bit 28              # "may cut off peacetime chatter"

free   = no_humming | non_triggered            # the peacetime set
danger = no_danger  | non_triggered            # the combat set

# each combat line takes one low bit for itself, plus the whole danger set:
alarm                = bit 0  | danger
attack_no_allies     = bit 1  | danger
attack_single_enemy  = bit 2  | danger
attack_several       = bit 3  | danger
backup               = bit 4  | danger
detour               = bit 5  | danger
search_no_allies     = bit 6  | danger
search_with_allies   = bit 7  | danger
enemy_lost_no_allies = bit 8  | danger
enemy_lost_with_allies = bit 9 | danger
need_backup          = bit 10 | danger
moving_in_danger     = bit 11 | danger
kill_wounded         = bit 12 | danger
enemy_critically_wounded = bit 13 | danger
enemy_killed_or_wounded  = bit 14 | danger

humming = bit 0 | free              # the only peacetime line

# four masks are all bits set, meaning "may interrupt anything":
die · die_in_anomaly · injuring · injuring_by_friend

# six lines carry the bare danger set with no bit of their own, so they are
# interchangeable with each other and with any combat line:
panic_human · panic_monster · tolls · wounded
grenade_alarm · friendly_grenade_alarm

any_sound = 0                       # the mask a caller passes to mean "no restriction"
```

**Invariants**

- Death and injury masks are *all bits set*, which is the encoding of "unconditional". A
  stalker always cries out when shot and always when killed, whatever it was saying.
- The `humming` mask shares bit 0 with `alarm`, but they are in different category groups
  (`free` versus `danger`), so they never contend. Bit reuse across groups is deliberate and
  a rebuild must not assume the low bits are globally unique.
- The six lines with no private bit cannot be distinguished by the filter, which is why a
  panicking stalker and one tolling for a dead comrade will cut each other off freely.
- **These values are stored nowhere on disk**, so a rebuild may renumber them — unlike the
  sound *identifiers*, which are exported to Lua and are frozen by conformance criterion 10.

## The body-action enumeration

```text
ENUM BodyAction = { none, hello }
```

One gesture. It exists so that dialogue can make a stalker wave.

## The weapon-class table

**Contract** — converts the *weapon type number stored in the game data* into the engine's
weapon class. This is a compatibility shim and it is the reason one executable runs three
games.

```text
ENUM WeaponClass =
  unknown · item · melee · mutant_1 · mutant_2 · mutant_3
  pistol · submachine_gun · shotgun · machine_gun · sniper_rifle
  grenade_launcher · grenade
  anomaly_mine · anomaly_field · psy_strike · throwing_items
  gravi · mincer · burning_fuzz · rusty_hair
```

The same stored number means different things in the third game than in the first two, so
there are two tables and the live one is chosen by which game's data is mounted:

| Stored | First two games | Third game |
|---|---|---|
| 4 | mutant type 3 | grenade |
| 5 | pistol | pistol |
| 6, 7, 8 | submachine gun, shotgun, sniper rifle | all three are submachine gun |
| 9 | grenade launcher | shotgun |
| 10 | grenade | machine gun |
| 11 | psychic strike | sniper rifle |
| 12 | thrown items | grenade launcher |
| 13–15, 17–19 | identical in both | identical in both |
| 16 | not used by any shipped data | not used by any shipped data |

**Invariants** — everything downstream that branches on weapon kind — cover distances, fire
queues, whether burst fire makes sense — branches on the *converted* class, never on the
stored number. That single indirection is what keeps the rest of the stalker free of
per-game conditionals, and a rebuild must preserve it or the conditionals will spread.

**Notes** — the slot for stored value 16 is skipped in both tables with a note that no
shipped data uses it. What it meant is not recoverable.
