# src/utils/mp_balancer/entry_point.cpp

> The balancing tool's entry — it brings up the core layer against the game's own filesystem description and runs exactly one of two modes named by a single argument.

**Needs** — [`pch.h`](pch.h.md) · [`wpn_collection.hpp`](wpn_collection.hpp.md) · [`statistics_collector.hpp`](statistics_collector.hpp.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrCore/xrDebug.h`](../../xrCore/xrDebug.h.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: process startup and a two-way branch.

## Purpose

The composition root of the balancing tool. It answers three questions and has no other
job: where the game's data lives, which of the two passes to run, and what to tear down
afterwards.

The first answer is the interesting one. The tool does **not** mount a single folder the
way the archive packer does; it initializes the core layer against the game's own
filesystem description file, so every logical root the game defines — the previous game's
configuration, the patch's configuration, the writable application data folder — resolves
exactly as it does inside the engine. That is what lets the two passes name their inputs
by root rather than by path, and it is why the tool must be run from a real game
installation rather than against a copied folder.

## State

Stateless. The two passes own everything.

## `main`

**Contract** — initializes the crash handler and the core layer with logging enabled,
selects the console's text encoding, loads the weapon collection unconditionally, then runs
whichever pass the single argument names. An unrecognized argument, or none, loads
everything and then does nothing — it neither reports the problem nor prints a usage
message. Blocks for the duration of the pass, including on operator input in the merge
mode. Returns nothing: the process always reports success.

```text
FUNCTION main(arguments)
  install_crash_handler()
  initialize_core(app_name = "mp_balancer", log_to_file = true,
                  filesystem_description = "fsgame.ltx")
  set_console_encoding(the game's single-byte Cyrillic encoding)

  collection <- new MergeRun
  collection.load_all()                   # always: both modes need it

  IF arguments.count == 2
    IF arguments[1] == "export_configs"
      collection.extract_all()            # the interactive merge
      collection.save_new_configs()
    ELSE IF arguments[1] == "made_csv"
      export <- new ExportRun(collection)
      export.load_settings()
      export.save_files()                 # the spreadsheet grids

  shut_down_core()
```

**Notes**

- **The console encoding is load-bearing.** The comments this tool prints and preserves are
  written in the developers' own language in a single-byte encoding, and the whole value of
  the interactive merge is that the operator can read the comment explaining the number
  they are being asked about. Selecting the wrong encoding does not corrupt the output —
  the comments pass through as bytes — it only makes the conversation unreadable. A rebuild
  working in a Unicode world has to decode those bytes explicitly, because the files
  themselves carry no encoding declaration.
- Loading the collection **before** looking at the argument means an unrecognized mode
  still parses a whole game's configuration, which takes seconds and produces the item
  listing on the console. That is not a decision; it is the branch being in the wrong
  place.
- Two mode names, matched exactly, with no abbreviation and no flag syntax. They are worth
  preserving only because any script that drives the tool spells them out.
