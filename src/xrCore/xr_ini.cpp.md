# src/xrCore/xr_ini.cpp

> The LTX configuration parser and writer: the INI-like text format with section inheritance and include directives that configures every tunable number in the game.

**Needs** — [`xr_ini.h`](xr_ini.h.md) · [`xrstring.h`](xrstring.h.md) · [`xr_trims.h`](xr_trims.h.md) · [`xr_token.h`](xr_token.h.md) · [`clsid.h`](clsid.h.md) · [`FS.h`](FS.h.md) · [`FileSystem.h`](FileSystem.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`xrDebug.h`](xrDebug.h.md) · [`log.h`](log.h.md) · [`string_concatenations.h`](string_concatenations.h.md)
**Used by** — [`xr_ini.h`](xr_ini.h.md)
**Tier floor** — T2: it is pure text parsing over an already-mapped byte range; only the fixed-size line buffers and the in-place mutation of the read buffer are T1 habits, and both are conveniences rather than requirements.

## Purpose

`ltx` is the single configuration format of the engine. Every weapon damage number, every AI parameter, every UI element size, every graphics preset and every entity class table lives in it. It is the second acceptance criterion in the conformance list, and it is **frozen on the read side**: the parser must accept everything the three shipped games contain, including duplicated keys, values with embedded commas parsed as tuples, multi-line quoted values, and sections that inherit from several parents at once.

The file carries both halves of the format — the parser and the writer — because the engine writes settings back in the same format, and a value that round-trips must survive both. The two halves are asymmetric on purpose: parsing is permissive, writing is canonical.

## State

A parsed configuration is a sorted set of sections; each section is a sorted set of key/value pairs. **Both orderings are byte-wise ascending on the key** and both are maintained as an invariant at every insertion, because all lookup is binary search. Nothing about authored order survives; neither do comments.

```text
RECORD Item                       # one "name = value" line
  key    : optional<text>         # interned; normalized by the whitespace-stripping pass
  value  : optional<text>         # interned; none when the line had no "=" or an empty right side

RECORD Section
  name   : text                   # interned, lowercased, without the brackets
  items  : list<Item>             # invariant: sorted ascending by key, keys unique

RECORD Config
  file_name     : text            # the path to write back to; may be empty for a reader-sourced config
  sections      : list<Section>   # invariant: sorted ascending by name, names unique
  includes      : list<text>      # include lines to re-emit on save; NOT the includes that were read
  read_only     : bool            # a read-only config refuses every write and drops value-less lines
  save_at_end   : bool            # write the file back when the config is released
  override_names: bool            # a write to an existing key replaces it instead of failing
```

Three invariants are enforced by scattered code and are the ones a rebuild is most likely to miss:

- **A section name may appear only once across the whole load, includes included.** A second `[name]` is a fatal error, not a merge. This is what makes a shipped configuration set diagnosable at all.
- **Inheritance is resolved at parse time, not at lookup time.** When a section header names parents, every parent's items are copied into the new section *before* the section's own lines are read, so a later own-line silently overwrites the inherited one. A parent must therefore already be fully parsed — inheritance is strictly backwards-looking, and the include order in the root file is load-bearing.
- **The `includes` list is a write-side artifact.** Reading an `#include` does not add to it. Only an explicit request to record an include does. A config that was parsed and is then saved loses its include structure and is flattened.

## `load`

**Contract** — Parses one configuration stream into the section set, recursively pulling in whatever it includes, resolving inheritance as it goes. Takes the stream, the directory that include paths are resolved against, and an optional predicate that can veto an include by path. Fatally fails on a duplicate section, on a section header with no closing bracket, and on an include that cannot be opened. Appends to whatever the config already holds, which is what makes recursive include work.

**Invariants** — On return the section list is sorted and duplicate-free. A section is committed to the list only when the *next* section header is seen or the stream ends; until then it is the "current" section and is the target of every `name = value` line.

```text
FUNCTION load(stream, base_dir, allow_include) -> void
  current = none
  WHILE NOT stream.at_end
    line = stream.read_line()            # reader strips the line terminator
    trim(line)

    # --- comment stripping -------------------------------------------------
    # Two comment markers: ';' and '//'. Whichever comes first wins.
    cut = first_index_of(line, ";")
    slashes = first_index_of(line, "//")
    IF slashes exists AND (cut does not exist OR slashes < cut) THEN cut = slashes
    IF cut exists THEN
      # A marker inside a quoted run is not a comment: the shipped data has
      # values like "bla-bla;nah-nah". Detect by finding a quote before the
      # marker whose partner lies after it.
      IF NOT marker_is_inside_quotes(line, cut) THEN truncate line at cut

    IF line starts with "#" AND line contains "#include" THEN
      handle_include(line, base_dir, allow_include)
    ELSE IF line starts with "[" THEN
      commit_section(current)             # fails on duplicate name
      current = begin_section(line)       # resolves inheritance, see below
    ELSE
      add_pair(current, line)             # ignored entirely when current is none
  commit_section(current)
```

**Notes** — Lines before the first section header are silently discarded; there is no global section. A configuration whose first non-comment line is a key/value pair loses that pair without complaint, and the shipped data relies on nothing here.

## Include resolution

**Contract** — An include line is `#include "relative/path.ltx"`. The path is taken as the *second* comma-free item of the line delimited by double quotes. It is joined to the directory of the including file, and the joined path's own directory becomes the base directory for anything that file in turn includes — so include paths are relative to the includer, not to the configuration root. A veto predicate, when supplied, is consulted with the resolved path and may skip the file silently.

```text
FUNCTION handle_include(line, base_dir, allow_include) -> void
  name = quoted_item(line, index 1)
  full = base_dir + name
  nested_base = directory_of(full)

  IF name contains "*.ltx" THEN
    # A wildcard include pulls in every .ltx in the named directory.
    # Order is whatever the filesystem enumeration yields, so a shipped
    # configuration must not depend on it -- and none does, because
    # wildcard includes are only used for leaf, non-inheriting files.
    FOR EACH found IN list_files(nested_base, pattern = name)
      load_one(nested_base + found, nested_base)
  ELSE
    load_one(full, nested_base)

FUNCTION load_one(path, nested_base) -> void
  IF allow_include exists AND NOT allow_include(path) THEN RETURN
  inner = filesystem.open(path)
  IF inner is none AND the host filesystem is case-sensitive THEN
    # The game data was authored on a case-insensitive filesystem and its
    # include lines do not match the on-disk case. Retry lowercased.
    inner = filesystem.open(lowercase_path(path))
  IF inner is none THEN FAIL WITH "can't find include file"
  load(inner, nested_base, allow_include)      # recursion: nesting is unbounded
```

## Section header and inheritance

**Contract** — A header is `[name]` optionally followed by `:parent[, parent]…`. The name is lowercased; parent names are not case-normalized before lookup but the lookup itself lowercases. Parents are resolved against sections already parsed; an unknown parent is a fatal error through the normal section lookup path. Inheritance is *multiple* and *left-to-right*: later parents overwrite earlier ones key-by-key, and the section's own lines overwrite all of them.

**Invariants** — Inheritance is only legal in a read-only config. A writable config that encounters an inherited header is a programming error, because the writer cannot reproduce the inheritance it flattened.

```text
FUNCTION begin_section(header_line) -> Section
  IF header_line has no "]" THEN FAIL WITH "bad ini section"
  section = new Section
  parents = text after "]:" in header_line, if any

  IF parents exists THEN
    # Two passes over the parent list: the first only to size the item list,
    # because a section inheriting from a dozen parents is common in the
    # weapon and creature tables and the reallocation cost showed.
    total = sum over p IN items(parents) OF size(lookup_section(p).items)
    reserve(section.items, total)
    FOR EACH p IN items(parents)          # items() splits on commas and trims
      FOR EACH item IN lookup_section(p).items
        insert_or_replace(section.items, item)

  section.name = lowercase(text between "[" and "]")
  RETURN section
```

**Notes** — `insert_or_replace` keeps the sorted invariant: binary-search for the key, replace the value in place on an exact hit, otherwise insert at the found position. The *replace* behaviour is what makes later parents win.

## Key/value lines and multi-line quoted values

**Contract** — A line is split at the first `=`. The left side is trimmed and becomes the key; the right side goes through the whitespace-stripping pass and becomes the value. A line with no `=` yields a key with no value. A key that is empty after trimming yields an item with no key, which a read-only config drops and a writable config keeps (so that a hand-edited file's blank-ish lines survive a round trip).

The whitespace pass is the subtle part and is the reason values may contain spaces at all:

```text
FUNCTION strip_whitespace(src) -> (dest, inside_quotes)
  # Removes every run of whitespace that is NOT inside a double-quoted run.
  # Toggling on each '"' means an odd number of quotes leaves us "inside",
  # which is the signal that the value continues on the next line.
  inside = false
  FOR EACH ch IN src
    IF ch is whitespace AND NOT inside THEN skip the whole run; CONTINUE
    IF ch is '"' THEN inside = NOT inside
    emit ch
  RETURN (emitted, inside)
```

When the pass reports that the value ended inside a quoted run, the value is a **multi-line string**: subsequent lines are appended, separated by carriage-return/line-feed, until the quotes balance.

```text
FUNCTION read_multiline_value(stream, first_raw) -> text
  raw = first_raw
  saved = current parse of raw            # the value as it stands after one line
  mark = stream.tell()                    # to rewind if the file is malformed

  WHILE the parse is still inside quotes
    raw = raw + CRLF
    more = stream.read_line()

    # A line that looks like a section header means the quotes were never
    # going to balance -- the file has an odd quote count, which several
    # shipped files do. Report it, keep only the first line's value with
    # its dangling quote trimmed, and rewind so the header parses normally.
    malformed = more contains "[" followed later by "]"
    IF malformed OR length(raw) + length(more) would overflow the line buffer THEN
      warn("incorrect inifile format ... odd number of quotes")
      value = trim(saved, trimming '"')
      stream.seek(mark)
      BREAK

    raw = raw + more
    reparse raw
    IF still inside quotes AND the reader is sitting on a blank line THEN
      raw = raw + CRLF                    # preserve the authored blank line
  RETURN value
```

**Notes** — The rewind-and-warn branch is a compatibility decision, not a bug guard: the shipped data contains files with unbalanced quotes, and the original engine's behaviour there — truncate at the first newline and carry on — is observable in the values those keys end up with. A stricter parser fails to load the game.

The line buffer is four kilobytes. That is a real limit of the format as shipped: a value longer than that is truncated with a warning rather than growing the buffer.

## `read_section` / `section_exists` / `line_exists` / `line_count` / `section_count`

**Contract** — Lookup by section name lowercases a stack copy of the name (capped at 256 bytes) and binary-searches the section list. A miss is **fatal**, not an error return: the engine treats a missing section as unrecoverable data corruption and dies with the section name in the message. `section_exists` and `line_exists` are the non-fatal probes callers must use first, and the whole `read_if_exists` family in the header exists to pair the probe with the read.

`line_count` counts only items that have a key, so value-less filler lines do not shift indices for callers that iterate by position.

**Notes** — The fatal-on-miss decision propagates outward: almost every configuration read in the engine is unguarded, which is why the acceptance criterion is that the *whole shipped set* parses, not that individual reads are robust.

## Typed reads

**Contract** — Every typed read is "fetch the string, then convert". The string itself is returned with quotes intact; only the explicitly quote-stripping variant removes one leading and one trailing quote. Conversions are deliberately forgiving — they parse a prefix and yield zero on garbage — because the shipped data contains fields with trailing text.

| Read | Conversion |
|---|---|
| integer of any width | decimal parse, then truncate to the requested width; no range check |
| 64-bit unsigned | decimal parse over the full 64-bit range |
| real | decimal float parse |
| bool | lowercase a copy, true for exactly `on`, `yes`, `true`, `1`; everything else false |
| float colour | four comma-separated reals into red, green, blue, alpha |
| integer colour | four comma-separated unsigned values into red, green, blue, alpha, **alpha defaulting to 255** when absent, then packed |
| 2/3/4-element integer or real vector | that many comma-separated values; components not present keep their zero initializer |
| class identifier | the eight-character packed identifier — see [`clsid.cpp`](clsid.cpp.md) |
| token | linear, case-insensitive scan of a caller-supplied name/number table; **0 when nothing matches**, which collides with a legitimate token numbered 0 |

**Invariants** — The vector and colour reads are the reason the format is "values with embedded commas that are parsed as tuples". They do not fail on a short list; they leave the missing components at their initialized value (zero, or 1 for a 4-vector's last component in some callers). A rebuild that errors on a short tuple will reject shipped data.

The separate "try" variants exist for exactly the cases where a short tuple must be detected — they report how many components actually converted.

## `write_string` and the typed writes

**Contract** — Writes are refused outright on a read-only config. The section name and both halves of the line go through the same whitespace-stripping pass as the parser, so a written value is already in canonical form. A missing section is created; an existing key is a fatal error *unless* the config was told to allow overrides, in which case it is replaced.

Every typed write formats to text and delegates: integers as decimal, reals with the default six-decimal fixed notation, vectors and colours as comma-separated components in field order, bool as `on`/`off` (note the asymmetry — the reader accepts four spellings of true, the writer emits one).

**Notes** — Because a written value is stripped of whitespace before storage, a value containing a space can only be written by quoting it. This is the same rule the parser enforces, which is what makes settings round-trip.

## `save`

**Contract** — Emits the recorded include lines first, then a blank line, then every section in sorted order. A section is `[name]`, then one line per item indented eight spaces with the key padded to 32 columns, ` = `, and the decorated value padded to 32 columns; trailing whitespace is stripped from each emitted line, and a blank line separates sections. An item with no key emits nothing at all. Saving to a path renames the target path's separators to the platform's and fails softly (returns failure) when the file cannot be opened.

Value *decoration* is the inverse of the parser's whitespace pass, applied only to commas:

```text
FUNCTION decorate(value) -> text
  # Put a space after every comma that is not inside quotes, so that a
  # written tuple reads like the hand-authored ones. The parser strips it
  # again on the next load, so this is purely cosmetic and need not be
  # reproduced -- but the shipped files look like this and a rebuild that
  # writes user settings differently will produce a noisy diff.
  inside = false
  FOR EACH ch IN value
    emit ch
    IF ch is '"' THEN inside = NOT inside
    ELSE IF ch is ',' AND NOT inside THEN emit ' '
```

An optional check mode additionally emits, as a comment under each section header, the interned name's checksum, reference count and length — a debugging aid for the string interner, not part of the format.

**Notes** — A config asked to save at end does so when it is released, and logs a failure rather than propagating it. That is the only place in the engine where a configuration write failure is non-fatal.

## `remove_line` / `remove_include`

**Contract** — Both refuse on a read-only config. Removing a line that does not exist is a no-op. Removing an include compares by exact text against the recorded include list.

## Global configurations

Three configurations are global and are the ones almost every caller reaches for:

- the **main settings** — the flattened result of loading the game's root `system.ltx` and everything it includes;
- the **authentication settings** — a small separate file, kept apart so it is not written back with the rest;
- the **OpenXRay settings** — this engine's own additions, keyed under a section named `openxray`, kept separate so that a shipped configuration set is never modified to hold them.

**Notes** — The separation of the third is the compatibility decision worth keeping: engine-specific configuration lives in its own file and its own section name, so a retail installation is never written into.

## Platform string conversions

**Notes** — On non-Windows targets the file carries hand-written 64-bit integer to/from text conversions that reproduce a specific C runtime's behaviour (including its error codes and its truncation-on-overflow semantics). This is incidental: it exists only because the rest of the file calls those functions by their Windows names. A rebuild uses its own integer formatting and deletes this entirely. The one observable behaviour worth preserving is that an out-of-range unsigned parse **saturates** rather than wrapping.
