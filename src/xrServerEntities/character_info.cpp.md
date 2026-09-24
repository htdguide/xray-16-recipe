# src/xrServerEntities/character_info.cpp

> Turns "this creature is profile *X*" into a named individual with a faction, a rank, a reputation and an opening line, filling in from the authored character only what the record did not already decide.

**Needs** — [`character_info.h`](character_info.h.md) · [`specific_character.h`](specific_character.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [Data: UI layout and text](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Two decisions live here. First, **a profile is a template and an entity record overrides
it**: whatever the record already carries (its own rank, reputation, community) wins, and
the authored character supplies only the gaps. Second, **almost none of this is saved** —
the profile identifier and the specific-character identifier are on the record, and
everything else is re-derived from XML on load. Only the opening dialogue is serialized
here, because scripts change it at run time.

## State

```text
RECORD CharacterProfile              # shared; one per profile identifier, loaded from XML
  specific_character : optional<text>   # if present, this profile IS that individual
  class              : optional<text>   # otherwise, the requirement to match against
  rank               : int              # unset means "any"
  reputation         : int              # unset means "any"

RECORD CharacterInfo                 # per entity
  profile_id           : text
  specific_character_id: text
  specific_character   : SpecificCharacter   # the authored individual, loaded by id
  start_dialog         : text                # the only serialized field
  rank                 : Rank
  reputation           : Reputation
  community            : Community
  sympathy             : real                # derived from the community
```

**Invariants**

- A profile either names a specific character *or* states a (class, rank, reputation)
  requirement; never both. The XML read enforces this by checking for the specific-character
  element first and ignoring the requirement fields when it is present.
- The class name is lowercased on load; profile matching is case-insensitive because the
  shipped XML is not consistent.
- Sympathy is never set independently — it is always the community's sympathy value for the
  community the character is in, recomputed whenever the community changes.

## `load_profile`

**Contract** — loads the shared profile template for an identifier. Profiles are shared
across every entity using them, so the template is loaded once and referenced.

## `read_profile_from_xml`

**Contract** — reads one profile element out of the XML file the identifier index points at.
If it has a `specific_character` child, the profile *is* that individual and nothing else is
read. Otherwise the profile is a requirement: `class` (lowercased, or unset), `rank` and
`reputation` (each defaulting to unset, not to zero — see
[`character_info_defs.h`](character_info_defs.h.md)).

**Notes** — the file and element are located through an identifier-to-position index built
once over every profile XML file named in configuration; the index is what makes "find
profile `sim_default_stalker_2`" cheap across several thousand entries. See
[`xml_str_id_loader.h`](xml_str_id_loader.h.md).

## `bind_to`

**Contract** — fills this from a trader record: takes the record's community index, rank and
reputation as the *starting* values, loads the record's profile, then resolves the record's
specific character. Order matters — the record's values must be in place before the
resolution step, because that step only fills gaps.

```text
FUNCTION bind_to(record)
  community  = record.community_index
  rank       = record.rank
  reputation = record.reputation
  load_profile(record.profile_id)
  resolve_specific_character(record.specific_character_id)
```

## `resolve_specific_character`

**Contract** — loads the authored individual and copies across only what is still unset:
rank, reputation, community, and the opening dialogue. Requires a non-empty identifier.

```text
FUNCTION resolve_specific_character(id : text)
  specific_character.load(id)
  IF rank is unset        THEN rank       = specific_character.rank
  IF reputation is unset  THEN reputation = specific_character.reputation
  IF community is unset   THEN community  = specific_character.community
  IF start_dialog is empty THEN start_dialog = specific_character.start_dialog
```

**Invariants** — this is the *only* inheritance rule between a record and its authored
character, and it runs exactly once per entity. A script that later changes the rank is
changing the entity, not the character.

## `save` / `load`

**Contract** — writes and reads exactly one field: the opening dialogue identifier, as a
null-terminated string. Everything else is re-derived.

**Notes** — this is the whole of this class's contribution to the entity record, and it is
worth stating plainly because the class looks much heavier than it serializes. The reason
only the dialogue is stored is that it is the only field scripts mutate: scripts routinely
reassign an NPC's opening conversation, and that change must survive a save. Rank,
reputation and community *are* saved — but by the trader record itself, not here.

## the read surface

**Contract** — name, biography, portrait and the actor-dialogue list all delegate to the
resolved authored character and assert that one has been resolved. Community, rank,
reputation and sympathy are the entity's current values. `set_community` also recomputes
sympathy, so the two can never disagree.

**Notes** — the name may be a string-table key rather than a literal; the UI translates it.
A character whose authored name is the marker `GENERATE_NAME` is meant to be given a random
name at spawn time, which the spawn tooling honours — see
[`object_factory_spawner.cpp`](object_factory_spawner.cpp.md).

## `configure_identifier_index`

**Contract** — tells the identifier index that profiles live under the element name
`character` and that the list of files to scan comes from the `profiles / files`
configuration key. Called once before any profile is loaded. It is a one-time binding of
data locations, and a rebuild can hard-wire it or keep it configurable as here.
