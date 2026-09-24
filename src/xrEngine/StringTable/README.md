# src/xrEngine/StringTable

Every piece of text the player reads — a menu label, an item name, a loading tip, the
caption on a key-binding row — is stored under an identifier and resolved through this
module. Nothing in the engine or the game spells a player-visible string in its own source;
they all name a key and ask here.

## Where it sits

A leaf of [`src/xrEngine`](../README.md). It rests on the virtual filesystem and the XML
reader from [`src/xrCore`](../../xrCore/README.md), and on the binding layer
([`xr_level_controller.h`](../xr_level_controller.h.md)) for one reason: a stored string may
contain an *action reference*, which is substituted with whatever key the player has bound
to that action at the moment the string is loaded. That is the only outward dependency, and
it is what makes "press ⟨use⟩ to open" display the right key.

It is built once during startup, in parallel with everything else that does not need a
device, and rebuilt in place when the language changes.

## Ideas to hold first

**The language is a directory, not a flag.** A localization is a set of XML files under a
per-language path; selecting a language re-points a logical filesystem root and reloads. The
same identifiers exist in every language, and a missing one falls back to *the identifier
itself*, which is why an untranslated string in this game appears on screen as
`ui_st_something` rather than as an empty space. That fallback is deliberate: a visible key
is a bug report.

**The text is not UTF-8.** Each localization ships in a single-byte codepage and is
translated to the engine's internal representation on load (system requirements §4). A
language also carries a **font prefix** and a **currency symbol**, because the glyph set and
the money format are properties of the localization rather than of any screen.

**Substitution happens at load, not at draw.** An action reference inside a string is
replaced once, when the table is built, which is why the table is rebuilt when the key
bindings change.

## Twins

| File | Role |
|---|---|
| [`StringTable.cpp`](StringTable.cpp.md) | Every piece of player-visible text, keyed by identifier, in the language the installation is configured for. |
| [`StringTable.h`](StringTable.h.md) | Declares the localized string table and the language selection it performs at startup. |
