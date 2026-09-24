# src/xrGame/ContextMenu.cpp

> A data-driven list of named commands rendered as text and dispatched into the engine's event bus when one is picked.

**Needs** — [`ContextMenu.h`](ContextMenu.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md)
**Used by** — [`ContextMenu.h`](ContextMenu.h.md)
**Tier floor** — T3: a list of strings, a text draw and an event signal

## Purpose

A menu whose entries are authored entirely in configuration: each key/value line of one
ltx section becomes one entry, where the key is the label shown to the player and the
value names an engine event plus a parameter string. Choosing an entry signals that event
with the parameter. Nothing in the file knows what any entry does, which is the point —
it is a way to expose engine commands to a level designer without code.

This is vestigial machinery from the era before the XML-driven UI toolkit; the shipped
games route their menus through that instead. A rebuild may leave it out and lose
nothing, but the recipe records it because the event-name-plus-parameter convention it
encodes also appears elsewhere.

## State

```text
RECORD MenuItem
  label      : text          # the ltx key, shown to the player
  event      : event handle  # resolved once at load from a name
  parameter  : text          # passed opaquely to the event's handlers

RECORD ContextMenu
  title : text
  items : list<MenuItem>     # in the section's declaration order
```

Invariant: every entry's event handle is created at load and lives as long as the menu;
tearing the menu down must release each handle and each string, or the engine's event
registry accumulates dead subscriptions.

## `Load`

**Contract** — reads one configuration section and turns every line in it into an entry.
The value is split on the first comma into an event name and a parameter; both halves may
be empty. The event name is resolved through the engine's event registry, which creates
the event if no one has declared it yet. Order of entries follows the section, because the
player selects by index.

## `Render`

**Contract** — draws the title then the entries, numbered from zero, at two fixed text
heights and in two caller-supplied colours. Purely visual; no hit-testing and no
selection highlight, which tells you the menu was always driven by number keys.

## `Select`

**Contract** — signals the event of the entry at the given index, passing the entry's
parameter string. An index outside the list is ignored rather than faulted, because the
caller is a keypress.

**Notes** — the parameter travels through the event bus as an opaque machine word that
happens to hold a pointer to the entry's string. A rebuild should pass the string itself;
the payload being pointer-width is an artifact of the event bus's signature, not a
decision.
