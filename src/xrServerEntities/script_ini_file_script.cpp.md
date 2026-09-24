# src/xrServerEntities/script_ini_file_script.cpp

> Exports the configuration file to scripts, together with the three global configuration files and a way to build one from a string.

**Needs** — [`script_ini_file.h`](script_ini_file.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Publishes the configuration reader as the script type `ini_file`, with the read surface, the
write surface, iteration over sections, and three free functions returning the process-wide
configuration files. The export is frozen by conformance criterion 10 — every method name
here appears in shipped scripts.

## `read_line`

**Contract** — reads the *n*-th key/value pair of a section, answering the two as separate
outputs plus a success flag. Errors if the section does not exist. A value that is absent
comes back as an empty string rather than nothing, because the script side has no
distinction.

**Notes** — this is what makes a configuration section iterable from script, which is how
every data-driven mod table is read. The by-index form is the one in use; a by-key variant
exists in the source, disabled, pending confidence in the binding layer's output-parameter
policy.

## `for_each_section`

**Contract** — calls a script function once per section name in the whole file. The
companion to `read_line`: together they let a script walk arbitrary configuration.

## `create_from_string`

**Contract** — parses configuration out of a string in memory and hands back a file the
script owns. Ownership transfers to the script's garbage collector, unlike the three global
files below, which the script must not free.

## `system_ini` / `game_ini` / `openxray_ini`

**Contract** — return the three process-wide configuration files: the game's own settings,
the multiplayer game settings, and this engine's additional settings. All three are borrowed
references.

**Notes** — the third has no counterpart in the original engine. It is this project's own
settings file, and exposing it to scripts is how mods detect that they are running on this
engine rather than the original.

## `reload_system_ini`

**Contract** — destroys the process-wide configuration and re-parses it from disk, returning
the new one. A development convenience with real consequences: anything holding an interned
pointer into the old configuration now holds a dangling one. It exists because the
configuration is several thousand sections and restarting the game to test a tuning change
is slow.

## the exported surface

The read side: section and line existence, the checked readers, the class-identifier read,
the token read, line count, and the two whitespace-preserving string reads. The write side:
one writer per scalar type, save-as, remove-line, and three mode toggles (read-only,
save-on-close, allow overriding a key that already exists). Section count and the two
iteration helpers complete it.
