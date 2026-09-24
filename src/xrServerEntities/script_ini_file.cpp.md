# src/xrServerEntities/script_ini_file.cpp

> Wraps every configuration read in a check that names the missing section or key, because a script that mistypes one would otherwise get a default and a mystery.

**Needs** — [`script_ini_file.h`](script_ini_file.h.md) · [`object_factory.h`](object_factory.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — reached through its declarations in [`script_ini_file.h`](script_ini_file.h.md); callers name that, not this file.
**Tier floor** — T3.

## Purpose

The engine's own configuration reader aborts the process on a missing key, which is right
for engine code — a missing key in shipped data is a broken installation. It is wrong for
scripts, where a missing key is usually a mod's typo and should surface as a script error
the script layer can report with a stack trace. This file is that change of failure mode,
applied uniformly, plus two genuine conversions.

## `resolve`

**Contract** — turns a logical root and a file name into a filesystem path through the
virtual filesystem, and returns it.

**Notes** — the returned text is interned and then returned by pointer, and the source marks
it as a dangling reference. It survives in practice because the constructor consumes it
immediately and the intern table holds the string, but it is a real defect and a rebuild
should return an owned value.

## the checked readers

**Contract** — each one verifies that the section exists and that the key exists, raising a
script-visible error naming the missing one, and only then delegates to the underlying
parser. `line_count` is the exception: a missing section yields zero rather than an error,
because counting the lines of an absent section is a reasonable question with a reasonable
answer.

```text
FUNCTION read(section, key) -> value
  IF NOT section_exists(section) THEN FAIL WITH "cannot find section " + section
  IF NOT key_exists(section, key) THEN FAIL WITH "cannot find line " + key
  RETURN underlying_read(section, key)
```

## `read_class_identifier`

**Contract** — reads the section's class tag and returns the **script-visible class
number**, not the 64-bit tag. It goes through the registry's index lookup (see
[`object_factory_inline.h`](object_factory_inline.h.md)) so that the value a script gets
from configuration is directly comparable to the `clsid` enumeration it already has.

**Invariants** — this is the one place the two class identities meet. A script never sees
the eight-character tag; it sees the table index at both ends of every comparison.

## `read_token`

**Contract** — reads a key whose value is one of a named set, using a token list supplied by
the script. Converts the script's list into the shape the parser wants by taking its first
element's address, which assumes the list is contiguous and terminated — both true of the
script token list type, and both invisible at this call site.

## the writers

**Contract** — one per scalar type, each a direct forward to the underlying parser's writer.
`save_as` refuses a missing file name and otherwise forwards.

**Notes** — the engine itself never writes configuration through this class; the whole write
surface exists for mods, which use it to persist their own settings. It is the reason the
configuration format's write side exists at all, since the engine's own settings go through
the console.
