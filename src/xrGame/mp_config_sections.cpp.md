# src/xrGame/mp_config_sections.cpp

> Serializes the configuration sections that decide a multiplayer match, so a server can compare a client's tuning against its own.

**Needs** — [`mp_config_sections.h`](mp_config_sections.h.md) · [`Weapon.h`](Weapon.h.md) · [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: configuration traversal and text serialization

## Purpose

A client that edits its own configuration files can give itself a faster actor or a
more accurate weapon, because every gameplay number lives in text. This file produces the
evidence a server needs to detect that: an incremental dump of the sections that matter,
plus a dump of the values actually in memory, which catches tampering that happens after
load.

Two separate classes because they answer two different questions — *what does your
configuration say* and *what are you actually running with* — and a cheat can make those
disagree.

## State

```text
RECORD ConfigSectionReporter
  sections      : list<text>      # the sections to report, fixed at construction
  cursor        : iterator into sections
  scratch_file  : a configuration document used only as a serializer
```

**Invariants** — the cursor starts at the end, so a reporter that has not been started
reports nothing rather than starting from the beginning by accident.

The scratch document is constructed with no backing file and with reading, writing and
overriding all disabled. It exists purely to borrow the configuration format's serializer:
a section is lent to it, written out, and taken back. A rebuild with a standalone
section-serializer does not need this object at all.

## The reported section list

**Contract** — assembled once at construction from two sources:

1. Every item named on every line of the item-group section — that is, every weapon,
   armour and artefact obtainable in multiplayer, expanded from the group lists the shop
   screens are built from.
2. A fixed list of twenty-three sections naming the actor's own tuning, its damage and
   immunity tables, its condition model, the five rank tiers, the per-mode and per-team
   game data for the four shipped modes, the base cost table, and the two bonus tables.

**Invariants** — the list is *the* definition of what counts as a competitively significant
number in this game. Anything absent from it can be edited by a client without detection.
A rebuild adding tunable gameplay data to multiplayer must add its section here or silently
lose the protection.

The item list is derived from data rather than hardcoded, so a mod that adds a multiplayer
item gets it covered automatically; the twenty-three engine sections are hardcoded because
they have no enumerable source.

## `start_dump` / `dump_one`

**Contract** — `start_dump` rewinds the cursor. `dump_one` writes exactly one section into
the given byte sink and advances; it returns whether *more* sections remain, so a caller
loops while it returns true. It fails hard if a listed section is absent from the
configuration — a missing section is a data error, not a cheat.

```text
FUNCTION dump_one(sink) -> bool         # "more remain"
  IF cursor is at the end THEN RETURN false
  REQUIRE the section named by the cursor exists
  lend that section to the scratch document
  serialize the scratch document into the sink
  take the section back
  advance cursor
  RETURN cursor is not at the end
```

**Invariants** — one section per call, not the whole set, because the set is large and the
dump is produced on the frame thread while a match is running. Spreading it over frames is
the point of the class's existence.

The section is *lent*, not copied: the reporter appends a reference to the real
configuration's section into its scratch document, serializes, and removes it. A rebuild
must be careful that the scratch document does not own what it was lent.

**Notes** — the return value is "more remain", not "this one succeeded", so the last
section is written on the call that returns false. A loop written as *while (dump_one)*
therefore emits every section and stops correctly, but a caller reading the result as
success will discard the final section.

## `mp_active_params::dump`

**Contract** — records an object's live parameter values into a report. Derives a section
name by prefixing the object's declared anti-cheat name, writes a key-to-section mapping
line into the report's index section, and then — only if that derived section is not
already present — asks the object to write its own parameters into it. Handles an absent
object by recording an empty name.

```text
FUNCTION dump(object, key, report)
  name := ""
  IF object exists AND its anticheat name is non-empty
    name := "ap_" + its anticheat name
  report.write(index_section, key, name)
  IF report already has a section called name THEN RETURN     # already dumped
  IF object exists THEN object.write_active_params_into(report, name)
```

**Invariants** — the presence check makes the dump idempotent across many objects sharing a
configuration section: ten identical weapons produce one parameter section and ten index
lines. Without it the report would grow with the player's inventory.

**Notes** — the object writes its own parameters rather than being read from outside. That
is the anti-cheat's only leverage: the values live in whatever shape the class chose, and
only the class knows which of them are competitively significant. See
[`mpactor_dump_impl.cpp`](mpactor_dump_impl.cpp.md) for the actor's answer.

## `mp_active_params::load_to`

**Contract** — the verifier's side. Copies every key and value of a named section out of the
real configuration into a report document, verbatim as text. Silently does nothing when the
section does not exist.

**Notes** — copying line by line as strings rather than by typed read is deliberate: the
comparison downstream is textual, so a value must survive the round trip unchanged,
including formatting that a typed read-and-write would normalize away.
