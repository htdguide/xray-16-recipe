# src/utils/mp_balancer/xr_ini_ex.cpp

> Parses and rewrites the configuration format while preserving the two things the engine's own parser discards — per-item comments and the section-inheritance list.

**Needs** — [`xr_ini_ex.h`](xr_ini_ex.h.md) · [`pch.h`](pch.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [`xrCore/FS_internal.h`](../../xrCore/FS_internal.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — reached through its declarations in [`xr_ini_ex.h`](xr_ini_ex.h.md); callers name that, not this file.

**Tier floor** — T3: text parsing against a frozen format. Nothing here is device- or layout-facing.

## Purpose

The engine reads configuration to *answer questions*: it flattens section inheritance at
load time, throws away comments, and never writes a file back. A balancing tool reads
configuration to *rewrite it*, and the two discarded things are precisely what a human
maintainer needs in the output — the comment that says what a number means, and the
`section : parent` line that says where the rest of the values came from.

So this is the engine's parser with two fields added and a writer attached. It is a fork,
and the duplication is a real cost: the shipped configuration files are parsed by two
implementations that must agree. A rebuild should have **one** parser whose retention of
comments and parent names is a load option, not a second copy. The decisions below are
what that one parser must implement; the ones the engine's copy shares are stated here
rather than cross-referenced, because this page is where the write side lives.

## State

```text
RECORD Item
  key     : optional<text>     # absent on a comment-only line
  value   : optional<text>     # absent when the line is "key ="
  comment : optional<text>     # the text after the comment marker, marker excluded

RECORD Sect
  name    : text               # lowercased at parse time
  items   : list<Item>         # sorted by key, ascending, case-sensitive
  parents : list<text>         # in the order written on the section header

RECORD ConfigFile
  path          : text
  sections      : list<Sect>   # sorted by name, ascending
  read_only     : bool
  save_at_end   : bool
  override_names: bool
```

**Invariants**

- **Both lists are kept sorted and searched by bisection, never scanned.** A shipped
  configuration set is tens of thousands of items and the tool looks values up per weapon
  per key; a linear scan is the difference between seconds and minutes. Sorting also fixes
  the *output* order, which is why a rewritten file is alphabetical rather than in the
  order the author wrote it — a visible, accepted consequence.
- **A section name is lowercased; a key is not.** Lookups by section are therefore
  case-insensitive and lookups by key are not. This is the format's rule, not this file's,
  and shipped data depends on it.
- **Inheritance is resolved at parse time, in header order, before the section's own
  items are read.** Each parent's items are inserted in turn, a later parent overwriting an
  earlier one on a key collision, and the section's own lines overwriting all of them. A
  parent must already be loaded — inheritance can only refer *backwards*, which makes the
  whole set a single ordered pass with no fixpoint.
- **Inheritance is legal only in read-only mode.** A writable file would have to decide,
  on save, which of its items were its own and which came from a parent; it cannot, because
  after the merge they are indistinguishable. Loading an inheriting file for writing
  therefore silently produces a file whose parents' values are written out as its own.
- **A duplicate section name is fatal, not a merge and not a last-wins.** Two sections of
  the same name in one file (or across an include) means the author does not know what the
  values are, so the load refuses rather than picking.
- **`parents` and `comment` are retained only in a diagnostic build.** In the shipped
  build both fields are compiled away, which makes the whole reason this fork exists
  disappear with them. This is the sharpest thing on the page: the tool is only useful
  when built with diagnostics on. A rebuild should retain them unconditionally and gate on
  a load option instead.
- **A value is parsed with whitespace collapsed outside quotes and preserved inside
  them.** An unbalanced quote does not end the value: it continues onto the following
  lines until the quotes balance, and blank lines inside such a value are preserved. This
  is how multi-line strings are expressed in a format that has no other way to say so.

## `Load`

**Contract** — parses a stream into sections, resolving include directives relative to a
caller-supplied folder, and merges the result into whatever is already loaded. Recursive:
an included file is loaded into the *same* object, so includes are textual and a section
defined in an include is visible to an inheriting section that follows it. Fails hard on a
duplicate section, a missing include, a section header with no closing bracket, or a value
whose quotes never balance before the stream ends. Allocates one section object per
section, which the object owns until it is destroyed.

```text
FUNCTION load(stream, folder)
  current <- none
  WHILE NOT stream.at_end
    line <- trim(stream.read_line())

    # A comment runs to end of line, unless the marker is inside a quoted run.
    marker <- earliest_of(first(line, ";"), first(line, "//"))
    IF marker EXISTS AND NOT inside_quotes(line, marker)
      comment <- line after marker
      line    <- line before marker

    IF line STARTS WITH "#include"
      name <- first quoted token of line
      # Resolved against the folder of the *including* file, not the working directory.
      load(open(folder + name), folder_of(folder + name))
      CONTINUE

    IF line STARTS WITH "["
      IF current EXISTS THEN commit(current)          # fatal if its name already exists
      current <- new Sect
      IF line CONTAINS "]:"                           # inheritance
        FAIL WITH not_read_only IF NOT read_only
        FOR EACH parent IN split_items(text after "]:")
          current.parents.append(parent)
          FOR EACH item IN section(parent).items
            upsert(current, item)                     # later parent wins
      current.name <- lowercase(text between "[" and "]")
      CONTINUE

    IF current IS none THEN CONTINUE                  # text before the first section: ignored

    key, raw <- split_once(line, "=")
    value <- collapse_whitespace_outside_quotes(raw)
    WHILE quotes_unbalanced(value)                    # multi-line string value
      value <- value + line_break + stream.read_line()
      IF still_unbalanced AND stream_just_passed_a_blank_line
        value <- value + line_break                   # a blank line inside the value is data
      FAIL WITH odd_quote_count IF value exceeds the line limit

    item <- Item(key or none, value or none, comment or none)
    IF read_only
      IF item.key EXISTS THEN upsert(current, item)   # comment-only lines are dropped
    ELSE
      IF item.key EXISTS OR item.value EXISTS OR item.comment EXISTS
        upsert(current, item)                         # comment-only lines are kept, to round-trip

  IF current EXISTS THEN commit(current)
```

**Notes**

- `upsert` replaces an existing item with the same key rather than appending a second one,
  which is what makes "own line overrides inherited line" work without a separate pass.
- The read-only and writable branches differ in exactly one way — whether a line with no
  key survives — and that difference exists so a file opened for rewriting keeps its
  free-standing comment lines. It is the only place the two modes change *parsing*.
- Detecting the blank line inside a multi-line value is done by inspecting the bytes just
  consumed from the stream for a pair of line terminators. A rebuild that tracks the
  parse position explicitly gets the same answer without reaching backwards into a buffer.

## `save_as`

**Contract** — writes every section and item to a sink in the format's own syntax. Two
forms: to an open sink, and to a named path which it opens and closes. The named form
returns whether the file could be opened at all; the sink form cannot fail. Does not
preserve the author's original ordering, spacing or blank lines — only the content, the
comments and, in a diagnostic build, the inheritance list.

```text
FUNCTION save_as(sink)
  FOR EACH section IN sections                # already in sorted order
    write_line(sink, "[" + section.name + "]")
    FOR EACH item IN section.items
      line <- CASE
        key AND value   -> pad(key) + " = " + pad(decorate(value)) + optional " ;" + comment
        key ONLY        -> pad(key) + " = "                        + optional " ;" + comment
        comment ONLY    -> ";" + comment
        nothing         -> ""
      write_line(sink, trim_right(line)) IF line NOT empty
    write_line(sink, " ")                     # one blank line between sections
```

**Invariants**

- Keys and values are written into fixed-width columns so a human can read a diff of the
  output. The widths are cosmetic; the trailing-whitespace trim that follows them is not,
  because the parser would otherwise read the padding back as part of the value.
- `decorate` reinserts a space after every comma that is **outside** quotes, undoing the
  whitespace collapse the parser applied. Inside quotes it changes nothing, because there
  the whitespace was data.
- A section is written even when it has no items, because its existence is itself a fact
  other sections may inherit from.

**Notes**

- The writer does **not** emit the inheritance header, even in a diagnostic build where
  the parent list survives. A file saved by this path therefore loses its inheritance and
  gains every inherited value as its own. The one place in this tool that needs the
  parents preserved writes the section header itself rather than going through here — see
  [`wpn_collection.cpp`](wpn_collection.cpp.md).

## `w_string`

**Contract** — sets one key in one section, creating the section if it does not exist.
Refuses to run at all on a read-only file. On a key that already exists it fails hard
unless the file was configured to allow overrides, in which case the old item is replaced
wholesale, comment included. Section name and key are whitespace-collapsed the same way a
parsed line is, so a value written and then read back compares equal.

```text
FUNCTION set(section_name, key, value, comment)
  FAIL WITH read_only IF read_only
  name <- lowercase(collapse(section_name))
  IF NOT section_exists(name) THEN insert new Sect(name) in sorted position

  item <- Item(collapse(key) or none, collapse(value) or none, comment)
  existing <- find(section(name).items, item.key)
  IF existing EXISTS
    FAIL WITH duplicate_key IF NOT override_names
    existing <- item
  ELSE
    insert item in sorted position
```

**Notes**

- Refusing a duplicate key by default is the opposite of the *parser's* rule, which
  silently overwrites. The asymmetry is deliberate: a duplicate in a file is the author's
  business, a duplicate written by a program is a bug in the program.
- The typed setters — number, flag, colour, vector — all render their argument to text and
  come back through here. Only the rendering differs, and it must match what the typed
  readers parse: integers in decimal, reals in a fixed-point form, flags as the words the
  format's truth set accepts, vectors and colours as comma-separated components. There is
  no separate storage for a typed value; the file holds text and the type lives in the
  caller's expectation.

## `r_string` and `r_string_wb`

**Contract** — return one value. The first returns it exactly as stored, quotes included;
the second returns it with one leading and one trailing quote removed if present. Both
**fail hard** when the section or the key is absent: a missing value is a broken
configuration set, and returning a default would hide it.

**Notes**

- Two readers exist because the format uses quotes for two different jobs: to protect
  embedded separators and whitespace (where the quotes are syntax and must go), and as
  literal characters inside a display string (where they must stay). The caller knows
  which; the file does not.
- The unquoting removes at most one quote from each end and does not check that they
  pair, so a value that starts quoted and ends unquoted loses its opening quote only.

## `r_line`

**Contract** — returns the *n*-th item of a section by position, as a key and a value,
reporting whether the position was in range. This is the only way to walk a section whose
keys are not known in advance, and it is how every job description in this tool is read.

**Notes**

- The position is into the **sorted** item list, not the file's line order, and
  comment-only lines occupy positions in it. A caller that pairs an index from
  `line_count` with an index used here can therefore run off the end, because the count
  excludes keyless items and this does not. Every caller in this tool gets away with it
  because its own job files carry no bare comments.

## `line_count`, `section_exist`, `line_exist`, `sections`

**Contract** — structure queries. The count reports how many items in a section have a
key, excluding comment-only lines. Existence checks bisect and never fail. `sections`
hands out the section list directly, which is how a caller iterates a whole file and how
[`mp_config_sections.cpp`](../mp_configs_verifyer/mp_config_sections.cpp.md) borrows a
section from one file into another.

## `r_clsid` and `r_token`

**Contract** — two readings of a value against a fixed vocabulary. The first converts a
short text into the packed class identifier the entity system uses; the second maps a
value onto a caller-supplied set of name/number pairs, failing hard when the value is not
in the set. Both exist so that configuration can name a thing rather than number it.

## `remove_line`

**Contract** — deletes one item from one section. Silently does nothing when the key is
absent. Refuses on a read-only file.

## Lifetime

**Contract** — a file configured to save at the end is written back when it is released,
and a failure to write is logged rather than raised, because by then there is nobody left
to tell. This is the only implicit write in the tool, and a rebuild is better off making
the save explicit: an automatic write on release means an aborted run still rewrites the
file it was reading.

**Notes**

- `pSettingsEx`, the process-wide handle the header declares, is never assigned by this
  tool. It is vestigial — the shape of the engine's own global settings handle, carried
  over with the fork and never used. A rebuild drops it.
