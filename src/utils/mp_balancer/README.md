# src/utils/mp_balancer — the multiplayer balance authoring tool

Part of chapter 28 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Deriving a multiplayer balance set from the previous game's, with a designer in the loop.
It has two modes and they are the two halves of one authoring session:

- **Merge.** Read the previous game's configuration and the patch that shipped on top of
  it, flatten each multiplayer item's inherited values into a concrete section, and — where
  the base game and its patch disagree about a value — ask a human which to keep,
  remembering the answer for every remaining occurrence of that key.
- **Export.** Transpose the same item set into spreadsheet grids, one per item family, so
  a designer can compare thirty weapons across forty numbers at once.

Neither mode is on any critical path. It reads shipped data and produces text a person
then edits by hand; nothing in the engine ever runs it, and **a rebuilder who already has
the shipped configuration files does not need to re-derive them.** This directory is
recipe'd because the rules it encodes are the only written record of how the multiplayer
balance set was constructed.

## Where it sits

At the end, beside the packer. It rests on the core layer only — the virtual filesystem,
the string helpers and a private fork of the configuration parser — and nothing rests on
it. It is a console program that must be run from inside a real game installation, because
it resolves its inputs through the game's own filesystem description rather than through
paths.

## Load-bearing ideas, named once

**The tool has its own configuration parser, and the fork exists for two fields.** The
engine's parser flattens section inheritance at load and discards both the parent names
and the per-item comments, because nothing at run time needs them. A tool that *rewrites*
configuration needs both — the comment says what a number means, the parent list says
where the rest of the values came from. Everything else about the two parsers is the same,
and a rebuild should have one parser with a retention option rather than two copies.

**Inheritance can only refer backwards.** A section's parents must already be loaded when
it is parsed, which makes the whole configuration set a single ordered pass with no
fixpoint. It is also why inheritance is legal only in read-only mode: a writable file
cannot tell, on save, which of its values were its own.

**The multiplayer item set is defined by the deathmatch price list.** Not by a weapon
list, not by a class registry — by what can be bought. Anything absent from that section
is not a multiplayer item however much it looks like one.

**Extraction is governed by prefix rules, and a rule's name is also its output file.** The
job description is one line per output file: the line's key is a section-name stem and the
output name at once, and its value is the set of inherited keys to pull down for matching
sections. Longest matching prefix wins. The conflation of "which keys" with "where it
goes" is arbitrary and a rebuild may separate them.

**A standing answer is remembered per key, not per section.** That is the entire reason
the tool is usable: a designer answers a few dozen questions instead of a few thousand.

**A flattened section contains the inherited keys the rule asked for, plus the section's
own keys that no parent defines.** A key the section *overrides* and the rule did not ask
for is lost. That is the sharpest hazard in the whole directory and the reason the job
description's key sets have to be written generously rather than minimally.

**The output is written by a hand-rolled writer, not the parser's own**, because the
parser's writer does not emit the inheritance header. The duplication is a gap, not a
decision.

**The console's text encoding is load-bearing.** The comments the tool prints and
preserves are written in the developers' own language in a single-byte encoding, and the
value of the interactive merge is that the operator can read the comment explaining the
number they are being asked about. The files carry no encoding declaration, so a rebuild
in a Unicode world has to decode those bytes explicitly.

## The files

| File | Role |
|---|---|
| [`entry_point.cpp`](entry_point.cpp.md) | Process entry: filesystem description, console encoding, the two modes |
| [`wpn_collection.cpp`](wpn_collection.cpp.md) | The merge: the item set, the prefix rules, section flattening, the operator conversation, the writer |
| [`wpn_collection.hpp`](wpn_collection.hpp.md) | Its surface |
| [`statistics_collector.cpp`](statistics_collector.cpp.md) | The export: table definitions, item-to-table assignment, the grid format |
| [`statistics_collector.hpp`](statistics_collector.hpp.md) | Its surface |
| [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md) | The forked configuration parser and writer: syntax, inheritance, multi-line values, the round trip |
| [`xr_ini_ex.h`](xr_ini_ex.h.md) | Its surface |
| [`tools.hpp`](tools.hpp.md) | The comma-separated list value, split and joined |
| [`iostreams_proxy.h`](iostreams_proxy.h.md) · [`iostreams_proxy.cpp`](iostreams_proxy.cpp.md) | A console-output shim for a build configuration that no longer exists; drop it |
| [`pch.h`](pch.h.md) · [`pch.cpp`](pch.cpp.md) | Build-time header aggregation; the one fact is that it substitutes the forked parser tool-wide |

Two further files in the directory are build description and carry no decisions.
