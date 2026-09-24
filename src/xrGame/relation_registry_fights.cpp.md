# src/xrGame/relation_registry_fights.cpp

> Keeps a short-lived list of who is currently fighting whom, so that "helping" can be recognized.

**Needs** — [`relation_registry.h`](relation_registry.h.md) · [`game_type.h`](game_type.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a short linear-scanned list with time-based expiry

## Purpose

Shooting someone is an act with no social meaning on its own; shooting someone *who is
currently attacking a third party* is help, and help earns goodwill. To recognize that, the
game needs to know which fights are in progress at the moment a hit lands. This file is that
memory: every hit between two entities registers or refreshes a fight, and fights expire
after a configured window of silence.

It is a separate file from the reaction rules in
[`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) because the two change
independently: this is a cache with an expiry policy, that is a table of consequences.

## State

```text
GLOBAL fights : list<FightData>     # unordered; scanned linearly
```

Fields of `FightData` are listed in [`relation_registry.h`](relation_registry.h.md), where
the three write-only ones are also called out.

**Invariants**

- The list is keyed by the **ordered pair** (attacker, defender). A mutual fight therefore
  occupies two entries, one per direction, and neither knows about the other. Deliberate:
  the question being answered is "is X attacking anyone", not "are X and Y fighting".
- Entries are expired by *inactivity*, measured from the last hit, not by fight duration. A
  running battle never expires; a pause longer than the configured remember-time drops the
  entry and a subsequent hit starts a fresh one.
- The list is small — one entry per attacker-defender pair with recent damage between them —
  so linear search is appropriate. A rebuild indexing it should key by attacker, since that
  is what every lookup asks for.

## `fight_register`

**Contract** — called from the damage path every time one entity hits another. Expires stale
entries first, then either refreshes the matching entry or appends a new one. Never blocks;
allocates only when appending.

```text
FUNCTION fight_register(attacker, defender, defender_opinion_of_attacker, damage)
  update_fight_register()                       # expire first, so a resumed fight after a
                                                #   long pause starts fresh rather than
                                                #   reviving a stale opinion snapshot
  existing = find entry with this (attacker, defender)
  IF existing EXISTS THEN
    existing.time_old  = existing.time
    existing.time      = now
    existing.total_hit = existing.total_hit + damage
  ELSE
    append new entry with this pair, this damage, timestamp now,
      and the defender's CURRENT opinion of the attacker
```

**Invariants** — the defender's opinion is captured **only on the first hit** of a fight and
never refreshed. That is the field's whole purpose: to remember how the victim felt *before*
the fight changed his mind, so a kill can be judged against the provocation rather than
against the hostility the provocation created. Expiring before the lookup is what makes that
snapshot honest — a fight that lapsed and resumed re-snapshots.

**Notes** — the snapshot is not actually consulted by the shipped kill handler; see the
`FightData` notes in [`relation_registry.h`](relation_registry.h.md). It is maintained
correctly here regardless, so a rebuild that wants the intended behaviour only has to change
the consumer.

## `find_fight`

**Contract** — the first entry whose attacker, or whose defender, matches an identifier;
absent if none. The caller chooses which end to match on. First match wins, and with a
character in several fights at once that is an arbitrary one.

**Notes** — the arbitrary choice matters for the help rule, which asks "who is my target
currently attacking" and then credits the player for defending *that* victim. A character
attacking two people credits the player for only one of them, chosen by list order. A
rebuild could credit all of them; nothing depends on the single answer except the size of
the goodwill award.

## `update_fight_register`

**Contract** — drops every entry whose last hit is older than the configured remember-time.
Called at the head of every registration, and separately from the game's periodic update, so
the list cannot grow unboundedly during a lull. Reads the remember-time once, in seconds,
and holds it in milliseconds for the process.

**Notes** — the remember-time is the window in which a third party's intervention still
counts as help. Making it longer makes late arrivals count; making it shorter demands the
help be simultaneous. It is the only tuning knob in the file and it is in configuration,
which is the right place for it.
