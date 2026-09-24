# src/xrServerEntities/xml_str_id_loader.h

> The index that turns "find the entry named *X*" into a lookup across a set of authored XML files, and gives every entry a stable small number.

**Needs** — [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`InfoPortion.cpp`](../xrGame/InfoPortion.cpp.md) · [`InfoPortion.h`](../xrGame/InfoPortion.h.md) · [`PhraseDialog.cpp`](../xrGame/PhraseDialog.cpp.md) · [`PhraseDialog.h`](../xrGame/PhraseDialog.h.md) · [`encyclopedia_article.h`](../xrGame/encyclopedia_article.h.md) · [`character_info.h`](character_info.h.md) · [`specific_character.cpp`](specific_character.cpp.md) · [`specific_character.h`](specific_character.h.md)
**Tier floor** — T2: it keeps parsed documents alive and hands out borrowed positions into them.

## Purpose

Several kinds of authored content are defined in XML, spread over a list of files named by
the configuration, and referred to elsewhere **by string identifier**: character profiles,
dialogue phrases, information pieces. A record holds `"sim_default_stalker_1"`; something
must turn that into the document node that defines it.

This is that index, generic over the kind of content. It is built once per kind, on first
demand, by opening every file in the kind's list and recording — for each element bearing an
identifier — which document it is in and which position within that document. Lookup by
identifier then finds the entry, and the caller navigates straight to that position.

It also assigns every entry a **dense index**, which is the second half of its job: a
16-bit-or-smaller number that can go in a record where the string cannot.

## State

```text
RECORD IndexEntry
  id          : text     # the authored identifier; unique across the whole kind
  index       : int      # dense, assigned in discovery order, 0..n-1
  position    : int      # which occurrence of the tag within its document
  document    : parsed XML document (borrowed; the index owns it)

# one of these per content kind, process-wide:
RECORD Index OF Kind
  entries   : list<IndexEntry>
  files     : text       # comma-separated file names, from the configuration
  tag       : text       # the element name that marks an entry
```

**Invariants** — the dense index is **the position in the entry list**, assigned in the
order files are listed and elements are met. It is therefore stable only as long as the file
list and the files' contents are: adding an entry in the middle of a file renumbers every
entry after it. Anything that stores a dense index across a save is relying on the game data
not changing, which is true for a shipped game and false for a mod under development — see
[`character_info.cpp`](character_info.cpp.md) for the consequence.

Identifiers are unique across the whole kind, enforced at build time by rejecting the
duplicate. The document is kept open for the process's lifetime, because entries point into
it.

## `InitInternal`

**Contract** — builds the index. Reads the file list from the kind's configuration, opens
each file, and records every element bearing the kind's tag and a non-empty identifier.
Blocks; allocates; runs once per kind. Two behaviours are caller-selected: whether a file
that fails to parse kills the process or is skipped, and whether a missing closing tag is
tolerated.

```text
FUNCTION build_index(kind)
  REQUIRE the index is not already built
  kind.establish_file_list_and_tag()      # supplied by the kind
  REQUIRE the file list is present
  IF the tag is absent
    LOG and RETURN                         # an unnamed kind indexes nothing, but does not crash
  next_index = 0
  FOR EACH name IN split(kind.files)
    doc = OPEN name + ".xml" FROM the gameplay configuration directory
    IF doc failed to parse
      CONTINUE                             # or die, per the caller's choice
    FOR EACH i, element IN occurrences of kind.tag IN doc
      id = element.attribute("id")
      IF id is absent or empty
        LOG and CONTINUE                   # a nameless entry is unreachable, not fatal
      IF id already in entries
        LOG and CONTINUE                   # first definition wins
      APPEND (id, next_index, i, doc) TO entries
      next_index = next_index + 1
    IF no occurrence was found IN doc
      CLOSE doc                            # nothing points into it, so do not keep it
```

**Invariants** — a document is kept alive if and only if at least one entry points into it.
That is the only lifetime rule here and it is easy to get wrong in a rebuild: closing a
document that an entry references leaves every lookup for that entry reading freed memory.

**Notes** — **the duplicate check is a linear scan of everything indexed so far**, inside a
loop over everything being indexed. Character profiles number in the thousands, so this is
quadratic and measurably slow at startup; the source carries a disabled timing probe that
suggests somebody measured it. A rebuild should use a set. The *behaviour* to preserve is
only "first definition wins, and say so".

**The first-wins rule matters for mods.** A mod that adds a file to the end of the list
cannot override an entry an earlier file defines — it must replace that file. This is the
opposite of how the configuration format resolves conflicts, and the asymmetry is not
documented anywhere in the game data.

## `GetById` / `GetByIndex`

**Contract** — find an entry by identifier or by dense index. Both **build the index on
first use** — so a lookup is the trigger, and nothing has to be initialized explicitly.
Both take a flag saying whether a miss is fatal: by default it is, because a record naming a
profile that does not exist is a data error the game cannot paper over.

**Notes** — the by-identifier lookup is again a **linear scan**, on every call. Profile
resolution happens once per character at spawn, so the cost is bounded but not small. A
rebuild indexes by identifier and keeps the dense-index ordering separately.

## `IdToIndex` / `IndexToId`

**Contract** — the two conversions, each with a caller-supplied value for "not found". These
are the operations a record actually uses: it stores the dense index and resolves it to an
identifier when it needs the authored data.

## `GetMaxIndex` / `DeleteIdToIndexData`

**Contract** — the highest dense index in use, which is how a random choice over the kind is
made; and teardown, which closes every document the index holds.

## Notes

**The index is per content kind and process-wide.** The kind supplies two things — a
function that establishes the file list and the tag name, and nothing else — and gets an
index in return. The generic-over-kind arrangement is a language mechanism; what matters is
that each kind has exactly one index and that it is built lazily.

**The file list comes from the configuration, not from a directory scan.** That is
deliberate: order determines the dense indices, and a directory scan's order is not stable
across platforms.
