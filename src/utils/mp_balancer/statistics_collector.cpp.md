# src/utils/mp_balancer/statistics_collector.cpp

> Exports the multiplayer item set as spreadsheet tables — one file per item family, one row per item, one column per named configuration key.

**Needs** — [`statistics_collector.hpp`](statistics_collector.hpp.md) · [`wpn_collection.hpp`](wpn_collection.hpp.md) · [`tools.hpp`](tools.hpp.md) · [`xr_ini_ex.h`](xr_ini_ex.h.md) · [`pch.h`](pch.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: reading a text format and writing a text format.

## Purpose

Balancing a weapon set means comparing thirty weapons across forty numbers, and a
configuration file is exactly the wrong shape for that: it is one weapon per screen. This
pass transposes it — the operator names a set of keys, and gets back a grid with the items
down the side and the keys across the top, which is what a spreadsheet wants.

It is the second of the tool's two modes and it is strictly read-only with respect to the
game data: nothing it writes is ever read back by the engine. The grids are an authoring
aid whose output goes into a spreadsheet, gets edited by a human, and comes back as
hand-written configuration. That round trip is not automated and was never meant to be.

## State

```text
RECORD TableDefinition
  prefix  : text          # a section-name stem; also the output file's base name
  columns : list<text>    # the configuration keys to export, in column order

RECORD ExportRun
  source : MergeRun        # borrowed; already holds the loaded configuration and item list
  tables : map<text, TableDefinition>   # keyed by prefix, kept in sorted order
```

**Invariants**

- **An item belongs to exactly one table: the one whose prefix is the longest that matches
  it.** Unlike the merge pass, which sorts its rules once by length and takes the first
  hit, this pass scans every table and keeps the longest match. The two arrive at the same
  answer when the prefixes nest, which is the only case the shipped job description
  contains — but they are different algorithms and only this one is actually
  specificity-correct.
- **An item that matches no table is omitted from every table, silently.** There is no
  catch-all and no diagnostic. A key set that forgets a family loses it from the export
  with no sign.
- **Every row has the same columns, and a key the item does not define is written as an
  empty cell** rather than skipped, because a grid with ragged rows is not a grid.
- The values exported are the ones the **previous game's** configuration resolves to,
  including inheritance. This pass never consults the patch: it is a picture of what the
  starting point looks like, not of what a merge would produce.

## `load_settings`

**Contract** — reads the table definitions from a dedicated section of the same job
description the merge pass uses, one table per line: the key is the prefix and the output
name, the value is the comma-separated column list. A line with no value defines a table
with no columns, which produces a file holding only the item names. Allocates one column
list per table; the run owns them.

```text
FUNCTION load_settings()
  FOR position IN 0 .. line_count(job, "csv_settings") - 1
    key, value <- line_at(job, "csv_settings", position)
    IF key IS none THEN CONTINUE
    tables[key] <- TableDefinition(key, split_value(value))   # tools.hpp
```

## `get_most_acceptable_group`

**Contract** — returns the table an item belongs to, or nothing. Compares each table's
prefix against the start of the item's name and keeps the longest that matches. Never
fails.

```text
FUNCTION table_for(item) -> optional<TableDefinition>
  best <- none
  FOR EACH table IN tables
    IF item STARTS WITH table.prefix
      IF best IS none OR length(table.prefix) > length(best.prefix)
        best <- table
  RETURN best
```

## `save_file`

**Contract** — writes one table to a file named for its prefix with the spreadsheet
extension appended. First row is the column headers, preceded by a header for the item-name
column; each following row is one item and its values. Every cell is quoted and cells are
comma-separated; rows end with the two-character line terminator the spreadsheet readers of
the day expected. Overwrites the destination; fails hard if it cannot be opened.

```text
FUNCTION save_table(table)
  rows <- text
  emit_row(rows, ["section_name"] + table.columns)

  FOR EACH item IN source.items                  # price-list order, not alphabetical
    IF table_for(item) IS NOT table THEN CONTINUE
    cells <- [item]
    FOR EACH column IN table.columns
      cells.append(value_of(source.previous, item, column) OR "")
    emit_row(rows, cells)

  write_all(open_for_write(table.prefix + ".csv"), rows)
```

**Invariants**

- Each cell is wrapped in quotes and followed by a separator, and the trailing separator is
  then removed from the row. That is a construction convenience, not a format rule; what
  the format requires is a separator *between* cells and none after the last.
- **Nothing is escaped.** A configuration value containing a quote, a comma or a line
  break produces a malformed row. Shipped values are numbers and short identifiers, so it
  never happened; a rebuild should escape properly rather than rely on that.
- Row order is the price list's order, so the grid and the price section line up
  row-for-row. Sorting the rows would be friendlier to read and would break that
  alignment.

## `save_files`

**Contract** — writes every table in turn. Pure iteration; no ordering guarantee beyond
the table map's own.

**Notes**

- The whole file's text is built in memory and written once. For a few dozen rows that is
  simply simpler than streaming, and the reserve hints in the original are performance
  noise with no bearing on a rebuild.
