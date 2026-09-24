# src/xrServerEntities/object_factory_register.cpp

> The registration list itself: every class identifier the engine knows, paired with the record type that holds its state and the live object that plays it.

**Needs** — [`object_factory_impl.h`](object_factory_impl.h.md) · [`clsid_game.h`](clsid_game.h.md) · [`xrServer_Objects_ALife_All.h`](xrServer_Objects_ALife_All.h.md) · [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a literal table. Nothing here is an algorithm.

## Purpose

This file is a data table written as code. It answers one question for every tag in
[`clsid_game.h`](clsid_game.h.md): *which record type deserializes this spawn record, and
which live object does the game create from it?* It is the join between the frozen data
identity (the tag), the persisted record (chapter 22's types) and the gameplay code
(chapter 23's). It belongs in this directory rather than in the game because the tools build
needs the record half of the same table.

A rebuild transcribes this table. There is no reasoning to recover: the pairings are
authored facts, and getting one wrong means a shipped level spawns the wrong thing.

## State

`Stateless.` The table is written straight into the registry described in
[`object_factory_inline.h`](object_factory_inline.h.md).

## `register_classes`

**Contract** — runs once when the registry is constructed, before any spawn record is read.
Registers, in this order: the multiplayer rules objects, the game screens, the actor, the
record-only types, then every creature, vehicle, artefact, weapon, item, outfit, grenade,
zone, device and physics object. Allocates one entry per line. Aborts on a duplicate tag.

```text
FUNCTION register_classes()
  # 1. the game-rules objects: one per shipped mode, three flavours each
  FOR EACH mode IN [single, deathmatch, team deathmatch, artefact hunt, capture the artefact]
    add server_rules_for(mode), client_rules_for(mode), screen_for(mode)

  # 2. the actor: shape depends on which game's data is mounted
  IF running Shadow of Chernobyl data
    add pair (actor, actor_record)
  ELSE
    add single-or-multiplayer pair (actor / mp_actor, actor_record / mp_actor_record)

  # 3. record-only types: no live object exists for these
  add record_only(monster_group_template), record_only(game_graph_point),
      record_only(online_offline_group)

  # 4. everything else: one line per tag, (live object, record type, tag, script name)
  FOR EACH entry IN the authored list
    add pair (entry.client, entry.server, entry.tag, entry.script_name)

  # 5. the script-only shadow tags, and only when scripts exist
  IF this process is a dedicated server WITHOUT scripts
    RETURN
  FOR EACH entry IN the shadow list
    add pair (entry.client, entry.server, entry.shadow_tag, entry.script_name)
```

**Invariants**

- Every tag registered here is unique, and every script name is unique. The registry asserts
  both.
- A tag that appears in a shipped spawn file must appear here, or loading that level aborts
  with "class identifier is missing".

## What the table actually says

Four facts are worth naming, because they are the ones a transcription can get wrong.

**Many tags share one record type.** Twenty-odd artefact tags all deserialize with the same
artefact record; the live objects differ (each has its own effect) but the persisted state is
identical. Likewise almost every magazine-fed weapon shares one weapon record and differs
only in its live behaviour and its configuration section. The *record* hierarchy is
deliberately much shallower than the *behaviour* hierarchy, because every extra record type
is another on-disk shape to keep frozen.

**A few tags share one live object too.** Two different anomaly tags both instantiate the
mincer; two zone tags both instantiate the hairs zone. These are cosmetic variants
distinguished by their configuration section, not by code.

**The actor is registered differently for the three games.** Under *Shadow of Chernobyl*
data the actor is a plain pair. Otherwise it gets a **single-player/multiplayer switchable**
entry: the same tag builds a different record and a different live object depending on
whether the process is running a single-player game, because the multiplayer actor carries
extra network state that the single-player save format must not contain. The choice is made
when the object is constructed, not when it is registered — see
[`object_item_client_server_inline.h`](object_item_client_server_inline.h.md).

**The level changer is registered under one of two tags, never both.** *Shadow of Chernobyl*
scripts call the level-changer class by the name that the later games use for a *different*
tag, so registering both would make the script name ambiguous. Which tag is live is decided
by which game's data is mounted. This is the clearest example in the file of the cost of
supporting three games' data from one binary.

**The shadow tags are the script-spawn names.** After the main list, a second block
registers roughly thirty more tags — `WP_AK74 `, `SM_FLESH`, `ZS_MINCE`, `SO_HLAMP` and
their siblings — pointing at the *same* class pairs as entries already in the table. These
never appear in authored level data; they are the identifiers the shipped Lua scripts use
when they spawn something at run time, and they exist because the three games' script sets
disagree about what a class is called. The block is skipped entirely on a dedicated server,
which has no scripts and therefore no use for them.

## Notes

**Why the includes are the whole game.** This one file pulls in nearly every gameplay header
in the project, which makes it the slowest translation unit in the build and the reason
chapter 22 and chapter 23 form a cycle. The cycle is not a design: the table needs both
halves of every pairing, and the two halves live in different modules. A rebuild breaks it by
making registration *declarative data* that both modules contribute to at startup, rather
than one file that names everything.

**The table is the enumeration scripts see.** Sorting it by tag and publishing
position-as-number is done in [`object_factory_script.cpp`](object_factory_script.cpp.md);
the script names in the fourth column of every line are the keys of that enumeration, and
they are frozen by conformance criterion 10 — a shipped script writes `clsid.wpn_ak74`, so
that exact spelling must exist.
