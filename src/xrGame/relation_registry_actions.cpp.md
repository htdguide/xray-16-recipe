# src/xrGame/relation_registry_actions.cpp

> The consequence table: what killing, attacking or helping someone does to the player's standing with that character's group and faction, and to his own rank and reputation.

**Needs** — [`relation_registry.h`](relation_registry.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`Actor.h`](Actor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a handful of table lookups and a walk over one squad group per event

## Purpose

The rules that make the world react to the player. Every one of the series' famous social
consequences — a faction turning on you for one stray shot, strangers warming to you for
saving them, rank rising with each kill — is decided here, from numbers in configuration.

The file is deliberately a *rules table with a dispatcher*, not an algorithm: the structure
that matters is which parties are affected, not how the arithmetic is done.

## State

```text
RECORD AttackPoints            # one set of attack awards, loaded under a name prefix
  goodwill_vs_friend      : int
  goodwill_vs_neutral     : int
  goodwill_vs_enemy       : int
  goodwill_to_community   : int    # optional in data; zero when absent
  reputation_vs_friend    : int
  reputation_vs_neutral   : int
  reputation_vs_enemy     : int

GLOBAL points_when_provoked : AttackPoints   # loaded under the "danger" prefix
GLOBAL points_when_free     : AttackPoints   # loaded under the "free" prefix
```

Kill and help awards are read as individual configured values rather than as a record,
because they have no provoked/unprovoked split.

**Invariants**

- Both attack-point sets are loaded once, when the relation registry is first created, so
  they exist before any hit can land. The community award is the only optional key; every
  other key must be present or the load fails, which is the right failure — a missing award
  would silently be zero.
- Awards are **deltas**, not absolutes, and every one of them lands in the clamped goodwill
  setter, so the social model saturates rather than running away.

## `action`

**Contract** — applies the consequences of one social event. Takes the actor of the event,
its subject, and which kind of event it was. Reads configuration, the fight register, the
squad hierarchy and the relation registry; writes goodwill, faction goodwill, reputation and
rank. Does nothing at all unless the actor of the event is an inventory-carrying character
that is not a monster — **the entire consequence system exists for the player and for
stalkers, and monsters have no standing in it.**

### The common prologue

```text
FUNCTION action(from, to, kind)
  IF from is not an inventory owner OR from is a monster THEN RETURN

  IF to is a stalker THEN
    to.remember_actor_did(kind)              # accumulated as a flag set, not overwritten
    relation = relation_type(FROM to, TOWARD from)  # the SUBJECT's opinion of the ACTOR,
                                             #   which is what decides the award band
```

**Invariants** — the relation used to choose an award band is the *subject's opinion of the
actor*, not the reverse and not the mutual one. Attacking someone who hates you costs less
than attacking someone who likes you, regardless of how you feel about him. That direction
is the whole design.

The subject also records the event kind in a per-stalker flag set, which persists: a stalker
remembers the accumulated set of things the player has done to it, and the dialogue and
behaviour layers read that set.

### `ATTACK`

Two independent halves.

**Recognizing help.** Only when the actor is the player. Attacks are rate-limited per
attacker by a configured minimum interval — the first thing the branch does is compare the
player's last scored attack against that interval and bail if it is too soon. This is what
stops a burst of automatic fire from scoring twenty separate social events. Then, if the
*subject* is itself currently attacking somebody (asked of the fight register), the player is
credited with helping that somebody, recursively, as human help or monster help depending on
whether the subject's victim's attacker is a stalker.

```text
IF actor is the player THEN
  my_fight = fight_register.find_by_attacker(from)
  IF now - my_fight.last_scored_attack < minimum_interval THEN BREAK
  my_fight.last_scored_attack = now

  their_fight = fight_register.find_by_attacker(to)
  IF their_fight EXISTS THEN
    victim = object(their_fight.defender)
    IF victim is a stalker THEN
      action(player, victim, their_fight.attacker is a stalker ? HELP_HUMAN : HELP_MONSTER)
```

**Notes** — the rate-limit lookup assumes the player already has an entry in the fight
register, because the hit that triggered this call registered one. That assumption is
load-bearing and unguarded: reaching this branch without a registered fight reads a missing
entry. A rebuild must keep the register-then-score ordering, or guard.

**Scoring the attack.** Only when the subject is a stalker. Which award set applies is
decided by one question: **was the player provoked?** If the subject's currently selected
enemy is a human who counts the player as *his* enemy, the player is fighting on the
subject's side of an existing quarrel and the "danger" set applies; otherwise the "free" set
does. The award is then chosen from that set by the relation band, and applied:

- to **every member of the subject's squad group**, not only the subject — the unit of
  social memory is the group, which is why shooting one man makes a whole patrol hostile;
- to the subject's **faction**, scaled by the subject's personal sympathy value, so how much
  a faction cares is a per-character trait;
- to the actor's own **reputation**.

**Invariants** — a stalker attacking another stalker scores nothing. The guard is explicit
and the reason given is that stalker-on-stalker hits are assumed accidental, which is really
a statement that the social model is only defined for player actions. A rebuild that wants
non-player factions to react to each other must build that separately; this table will not
carry it.

### `KILL`

Same shape without the provocation split. The band comes from the subject's relation to the
killer at the moment of death, and the award goes to every group member **except the victim
himself** — a corpse is not updated — plus the faction, scaled by sympathy, plus the killer's
reputation. Then the killer's **rank** rises by the kill-points value for the *victim's* rank
band: killing a veteran advances you more than killing a novice.

**Invariants** — the team-mate guard is different here than for attack. A kill is excused
only when killer and victim share a faction; an attack is excused whenever both parties are
stalkers. So a stalker killing a stalker of another faction *does* score, while a stalker
merely shooting one does not. Whether that asymmetry is intentional is not recoverable; it
reads more like drift than design.

**Notes** — the commented-out lines show the intended version: judge the kill by the
defender's opinion snapshot taken when the fight started
([`relation_registry_fights.cpp`](relation_registry_fights.cpp.md)) rather than by his
opinion at death. The shipped behaviour means provoking a neutral into attacking you and then
killing him is scored as killing an enemy — the provocation launders itself. The intended
behaviour is the more interesting game. A rebuild should choose knowingly.

### `FIGHT_HELP_HUMAN` / `FIGHT_HELP_MONSTER`

One branch for both. Requires the subject to be a *live* stalker — you cannot earn credit for
avenging the dead. The award is chosen by relation band from the help values, applied to
every member of the subject's group, to the faction scaled by sympathy, and to the actor's
reputation. The two kinds are distinguished only by which configured values are read, which
is what lets helping against a mutant be worth a different amount than helping against a man.

## `load_attack_goodwill`

**Contract** — loads both attack-point sets, under the "danger" and "free" name prefixes,
from the action-points configuration section. Called exactly once, from the relation
registry's creation. A prefix-plus-key naming scheme is what lets one record type serve two
tuning sets; a rebuild may prefer two sections.

## Unrecovered

- **Sympathy** scales every faction-level award and is a per-character configured value, but
  nothing in this file or its neighbours states the intended range. If it exceeds one, a
  single sympathetic character can move a faction's standing more than the configured award.
- The interaction between the group-wide award and squad membership changing mid-fight is
  undefined: a stalker who joins the group after the event carries none of it, and one who
  leaves keeps it.
