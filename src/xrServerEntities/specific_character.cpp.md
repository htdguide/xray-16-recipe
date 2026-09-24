# src/xrServerEntities/specific_character.cpp

> Parses one authored character profile out of the gameplay XML, and defines what an authored individual consists of.

**Needs** — [`specific_character.h`](specific_character.h.md) · [`xml_str_id_loader.h`](xml_str_id_loader.h.md) · [`shared_data.h`](shared_data.h.md) · [`character_info_defs.h`](character_info_defs.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`specific_character.h`](specific_character.h.md)
**Tier floor** — T2.

## Purpose

A creature record holds a profile identifier. This turns that identifier into a named
individual: a display name, a faction, a rank, a face, a voice, a starting inventory and a
handful of behavioural tunings. It is the boundary between the simulation — which deals in
records and numbers — and the authored content that gives those records a personality.

Every field here is authored. Nothing in it changes at run time, which is why one parsed
copy serves every entity that names the profile.

## State

```text
RECORD SpecificCharacter
  display_name        : text       # shown in the UI; not the profile identifier
  biography           : text       # a localization key, resolved at load
  starting_inventory  : text       # a section list; see the escape note below
  behaviour_section   : text       # the configuration section supplying this character's numbers
  voice_prefix        : text       # selects the character's sound set
  visual              : text       # the model
  icon                : text       # inventory portrait; defaults to the barman's
  faction             : community  # required
  rank                : int        # required
  reputation          : int        # required
  classes             : list<text> # the groups this character belongs to, lowercased
  start_dialogue      : optional<text>
  actor_dialogues     : list<text> # dialogue options unlocked only by meeting this character
  terrain_section     : text       # which terrain preferences apply
  panic_threshold     : real       # default 0
  hit_probability     : real       # default 1
  crouch_style        : int        # default 0
  is_repair_mechanic  : bool
  critical_wound_weights : text    # default "1"
  money               : (min: int, max: int, unlimited: bool)
  excluded_from_random : bool
  default_for_faction  : bool
```

**Invariants** — **`excluded_from_random` and `default_for_faction` are mutually exclusive**
and the loader refuses a profile that sets both, naming it. The two mean opposite things:
the first says this individual is only ever produced by explicit reference, the second says
they are what you get when a faction is asked for a representative and nothing more specific
applies.

Faction, rank and reputation are **required** and their absence is fatal, each with its own
message naming the profile. An unrecognized faction name is fatal too. These are the three
fields the simulation cannot invent a default for: a creature with no faction has no
relations, and a creature with no rank cannot be ordered against others.

Money's maximum is **clamped up to its minimum** if the author wrote them the wrong way
round, silently. Absent entirely, the character has no money and is not unlimited.

## `Load`

**Contract** — binds this handle to a profile identifier and triggers the shared load.
Requires a non-empty identifier. Cheap on every call after the first for a given identifier;
the parse happens once per profile per process.

## `InitXmlIdToIndex`

**Contract** — tells the index what to look for: the element name is `specific_character`,
and the file list comes from the configuration key naming the profile files. Both are set
only if not already set, so the first kind to ask wins — which is fine because there is only
one profile kind.

## the parse

**Contract** — navigates to this profile's element in its document and reads every field.
Blocks; runs once per profile.

```text
FUNCTION parse_profile(id)
  entry = index.find(id)                      # document + position within it
  node  = entry.document.node_at(entry.tag, entry.position)
  REQUIRE node exists, naming id              # the index said it was there

  excluded_from_random = node.attribute("no_random") == 1
  default_for_faction  = node.attribute("team_default") == 1
  REQUIRE NOT (excluded_from_random AND default_for_faction)

  start_dialogue  = node.child("start_dialog")           # optional
  actor_dialogues = every child "actor_dialog", in order
  icon            = node.child("icon") ELSE "ui_npc_u_barman"
  display_name    = node.child("name")
  biography       = localize(node.child("bio"))          # resolved now, not at display
  ... the tunings, each with its default ...
  starting_inventory = unescape_newlines(node.child("supplies"))
  classes         = every child "class", lowercased
  faction         = lowercase(node.child("community"))   # required, must resolve
  rank            = node.child("rank")                   # required
  reputation      = node.child("reputation")             # required
  money           = node.child("money") attributes min/max/infinitive, ELSE zero
```

**Notes** — **the starting-inventory string carries escaped newlines that are unescaped
here.** The field is a multi-line list of sections to spawn, and the authoring format stores
it on one XML line with a two-character escape. Undoing it at load is what lets the consumer
split on real newlines. A rebuild whose authoring format allows real newlines skips this
entirely, but must still accept the escape, because the shipped files use it.

**The biography is localized at load, not at display.** That freezes the language at the
moment the profile is first demanded — which is once per process — so a language change
mid-session leaves biographies in the old language. Nothing in the game changes language
mid-session, so the bug is latent.

**Faction and class names are lowercased before use**, because the authored files are
inconsistent about case and the lookup is exact. This is a data-hygiene rule that a rebuild
must reproduce or it will fail to resolve a handful of shipped profiles.

**The class list reads the same element repeatedly.** The loop counts the `class` elements
and then reads index zero every time, so a profile with several classes gets the first one
repeated. This is a defect in the original and it means the multi-class feature has never
actually worked; no shipped profile has more than one class, which is why nobody noticed. A
rebuild should read the *i*-th, and should expect nothing in the game data to change as a
result.

**The actor-dialogue list, by contrast, reads the *i*-th correctly** — the two loops sit
next to each other and differ exactly in that.

**A disabled timing probe brackets the whole parse**, which is a fair hint that profile
loading was once a visible cost. It is; see
[`xml_str_id_loader.h`](xml_str_id_loader.h.md) for the quadratic index build that dominates
it.
