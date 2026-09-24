# src/xrServerEntities/alife_human_brain.cpp

> The human-specific part of an offline creature's record: randomized equipment tastes, and the version-gated read that lets a stalker saved by any of the three games load here.

**Needs** — [`alife_human_brain.h`](alife_human_brain.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Data: save games](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — reached through its declarations in [`alife_human_brain.h`](alife_human_brain.h.md); callers name that, not this file.
**Tier floor** — T1: the read path is a byte-exact, version-gated walk over a saved record.

## Purpose

A human offline is a monster offline plus *taste*: two small arrays that say how much this
stalker wants each kind of equipment and each kind of main weapon, which is what the offline
trading and equipping logic consults when it decides whether a stalker picks up the rifle it
walked past. This file owns those arrays, their randomized initialization, and — the
load-bearing part — the version-gated serialization that keeps saves from three different
games readable.

## State

```text
RECORD HumanBrain EXTENDS MonsterBrain
  object_handler        : OfflineInventoryHandler   # owned here
  equipment_preferences : list<int (8-bit)>         # exactly 5 entries; see below
  weapon_preferences    : list<int (8-bit)>         # exactly 4 entries
  money                 : int (32-bit)
```

**Invariants**

- The two arrays' lengths are **not free**: they must equal the number of distinct values the
  data-driven equipment-kind and weapon-kind classifiers can produce, which the engine reads
  out of the loaded evaluation functions at construction. If they disagree the engine aborts
  with an instruction to rebuild the spawn file — because a spawn file records these arrays
  at their length, and a length change silently mis-parses every stalker in the world.
  Shipped data gives 5 and 4.
- Each entry is a small value in `0..2`. The range is the classifier's, not an arbitrary
  cap.

## `construct`

**Contract** — binds to the record, creates the offline inventory handler, zeroes the money,
sizes both preference arrays from the loaded classifiers, checks those sizes against the
expected 5 and 4, and fills every entry with a fresh random value in `0..2`.

**Notes** — the randomization is the point: two stalkers spawned from the same configuration
section differ in what they will bother to carry, which is most of what makes the offline
population feel like individuals rather than clones. The seed is the global one, so a rebuild
that wants reproducible worlds must decide where it comes from — the original does not.

The size check is written as a fatal assertion whose message tells the user to rebuild
`game.spawn`. That is the honest diagnosis: the arrays are *in* the spawn file, so a tools
build and a game build that disagree about their length cannot interoperate at all.

## `on_state_write`

**Contract** — appends both preference arrays, each as a length followed by its bytes.
Writes nothing at all when the destination is the configuration-backed stream rather than a
packet — see the note below.

## `on_state_read`

**Contract** — reads the brain's slice back, and this is where the three games' formats are
reconciled. The record's own version number decides which fields are present.

```text
FUNCTION on_state_read(source)
  version = owner.record_version

  IF version <= 19  THEN RETURN                    # nothing of the brain was stored yet

  IF version < 110
    read and discard a list of 32-bit values       # an obsolete per-kind counter
    read and discard a list of booleans            # an obsolete per-kind flag

  IF version <= 35  THEN RETURN                    # the rest was added at 36

  IF version < 110
    read and discard one text field                # an obsolete "current task" name

  IF version < 118
    read and discard a list of entity identifiers  # an obsolete "known objects" list

  IF source is a packet
    read equipment_preferences
    read weapon_preferences
```

**Invariants** — the obsolete fields must still be *read*, not skipped by byte count: the
stream has no framing, so the only way past them is to parse them and throw them away. A
rebuild that wants to support old saves has no shortcut here.

**Notes**

**The version numbers are the format's history, and each one is a fact.** 19 and 35 are
pre-release revisions from before the brain had a record at all; 110 is where three obsolete
fields were dropped together; 118 is where the "known objects" list went. The numbers have no
internal structure — they are a single counter bumped whenever any entity record changed
shape — so a reader must compare against exactly these values.

**The configuration-backed stream is a different medium.** These records can also be
constructed from an `ltx` configuration section instead of a binary packet (the editor and
some script spawns do this), and a configuration section cannot express a length-prefixed
byte array. Both directions therefore skip the preference arrays when the stream is
configuration-backed, and the arrays keep their randomized values. This is the one place in
the chapter where "the same record, three serializations" becomes *four*, and it is worth
keeping in mind when reading the record twins.
