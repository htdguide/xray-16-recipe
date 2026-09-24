# src/utils/mp_configs_verifyer/mp_config_sections.cpp

> Enumerates, in a fixed and reproducible order, every configuration section whose contents a multiplayer client must not have altered.

**Needs** — [`mp_config_sections.h`](mp_config_sections.h.md) · [`pch.h`](pch.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T2: the enumeration order and the serialized form are a wire contract, because a digest is taken over the result.

## Purpose

An anti-cheat that compares configuration has to decide *which* configuration. Most of a
game's data is irrelevant to fairness — a texture path, a sound volume, a menu layout —
and hashing all of it would make every legitimate mod a cheat. This file draws the line:
the sections that decide how much damage a weapon does, how tough an actor is, what a rank
grants, what a game mode pays out, and what anything costs.

The line is drawn in two ways at once, and that is the design:

- **By reference.** Most of the set is not written here; it is read out of the shipped
  configuration itself, from a section that lists the item groups. Adding an item to the
  game therefore adds it to the anti-cheat set with no code change — the data names its
  own protected surface.
- **By name.** A short hard-coded list covers the things that are not items: the actor's
  own parameters, the rank ladder, the per-mode and per-team rules, and the two bonus
  tables.

Everything downstream depends on this set being **identical and identically ordered** on
both ends, because the digest is taken over the concatenation.

## State

```text
RECORD ProtectedSet
  sections : list<text>     # item-group members first, then the fixed names, in order
  cursor   : position into sections
```

**Invariants**

- **Order is the contract.** The list is built by reading the item-group section in file
  order, expanding each group's comma-separated member list in order, and then appending
  the fixed names in the order they are written in the source. Nothing sorts it. Two ends
  that build the same *set* in a different *order* produce different digests and every
  honest player is reported as a cheat.
- **Every named section must exist.** Serializing a section that is absent from the
  configuration fails hard rather than emitting nothing, because a silently missing section
  is a digest mismatch with no explanation.
- **The fixed names are a frozen list.** They are:
  the actor's base section, its damage table, its immunity table and its condition table;
  the rank ladder's base plus five ranks; the deathmatch rules and its single team; the
  team-deathmatch rules' two teams; the artefact-hunt rules and its two teams; the
  capture-the-artefact rules and its two teams; the deathmatch price list; and the money
  and experience bonus tables. Twenty-three names. Their *content* is game data; their
  *membership in this list* is the decision, and it is the shortest honest statement of
  what the designers considered exploitable.
- The stepper is single-pass and not reentrant: one cursor, rewound explicitly.

## `mp_config_sections` — building the set

**Contract** — reads the shipped configuration at construction and produces the ordered
protected set. Fails hard if the item-group section is missing. Allocates one shared name
per section. Does not read the sections' contents.

```text
FUNCTION build_protected_set() -> ProtectedSet
  sections <- empty list<text>
  FOR position IN 0 .. line_count(settings, "mp_item_groups") - 1
    name, members <- line_at(settings, "mp_item_groups", position)
    FOR EACH member IN split_value(members)          # comma-separated, in file order
      sections.append(member)
  FOR EACH name IN the twenty-three fixed names
    sections.append(name)
  cursor <- past the end                              # a stepper must be started explicitly
```

**Notes**

- The group's *key* is read and discarded — only the member list matters. The key names the
  group for a human; the anti-cheat does not care which group an item is in.
- Duplicates are not filtered. An item listed in two groups is serialized twice, which is
  harmless as long as both ends do it identically — and they do, because both run this same
  enumeration.

## `dump_one` — serializing one section

**Contract** — appends the next section's full text to a byte sink in the configuration
format's own syntax, advances the cursor, and reports whether a further section remains.
Fails hard on a section that does not exist. The sink is the caller's; this appends and
never rewinds.

```text
FUNCTION dump_one(sink) -> bool
  IF cursor IS past the end THEN RETURN false
  FAIL WITH missing_section IF NOT section_exists(settings, sections[cursor])

  # Borrow the live section into an otherwise-empty scratch file and write that file out,
  # so one section is emitted in exactly the syntax a whole file would use.
  scratch.sections <- [ section(settings, sections[cursor]) ]
  write_config(scratch, sink)
  scratch.sections <- empty

  cursor <- cursor + 1
  RETURN cursor IS NOT past the end
```

**Invariants**

- The bytes appended are the **serialized text form**, not the in-memory values: the
  digest is taken over a rendering, so the renderer's column padding, its separator
  spacing and its blank line between sections are all part of the protocol. Change the
  writer and every existing dump stops verifying. This is the single most fragile coupling
  in the anti-cheat and it is entirely implicit.
- The return value is "more remain", not "this one was written", so a caller driving it in
  a loop writes every section and stops after the last — the loop's test lags the work by
  one step. A rebuild is better served by an explicit iteration over the set.

**Notes**

- Borrowing a live section into a scratch file rather than copying it is an
  allocation-avoidance trick, and it means the scratch file must be emptied again before
  it goes out of scope or it would release a section it does not own. That whole hazard
  disappears in a language with value semantics; the decision it encodes is only "emit one
  section using the whole-file writer".

## `mp_active_params` — the dynamic half

**Contract** — `load_to` copies every line of one named section out of the authoritative
configuration into a scratch file under the same section name. Silently does nothing when
the section does not exist, because the name came from an untrusted dump and a bad name is
a verification failure rather than a crash.

```text
FUNCTION load_active_section(name, scratch)
  IF NOT section_exists(settings, name) THEN RETURN
  FOR position IN 0 .. line_count(settings, name) - 1
    key, value <- line_at(settings, name, position)
    set(scratch, name, key, value)
```

**Notes**

- This is the verifier's side of a two-sided idea. The client, holding a live game object,
  asks it for the name of the section describing its current state and writes that
  section's *actual* values into the dump. The verifier, holding no game objects, takes
  the **names** from the dump and looks up what those sections *should* say. The
  comparison then catches a client that edited the values, and cannot catch a client that
  lied about the names — which is why the name list is itself covered by the digest.
- The client-side half is present in the source as commented-out text and is not compiled
  here: this tool is the server, and the server has no game objects. The contract it
  implies is still load-bearing, so it is stated above.
- Dynamic section names are distinguished from static ones by a three-character prefix,
  which is how the comparison in
  [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) knows which of the two
  sources to check a section against.
