# src/xr_3da/resource.h

> The numeric identifiers under which the executable's icons, splash image and pre-window error dialog are embedded in the binary.

**Needs** — _(nothing in this repository)_

**Used by** — [`embedded_resources_management.h`](../xrEngine/embedded_resources_management.h.md)

**Tier floor** — T1: it names things embedded in the executable image and addressed by integer, which only a tier that describes its own binary layout can do directly. A rebuild that ships these as data files instead is at T3.

## Purpose

The executable carries four kinds of content in its own image rather than in the game data:
**three application icons**, a **splash bitmap**, and the widget identifiers for a
**failure dialog that must be showable before a window or a graphics device exists**.
This file is the agreed numbering between whoever embeds them and whoever asks for them.

The load-bearing part is not the numbers. It is *why these four things are in the
executable and not in the archives*: all four are needed either before the virtual
filesystem is mounted, or in order to report that mounting it failed.

## State

```text
RECORD EmbeddedResources
  icon_per_game     : map<GameTitle, ImageId>   # three: one per supported retail title
  splash            : ImageId                   # shown while startup work proceeds
  failure_dialog    : DialogId                  # fields below
  out_of_address_space_title : StringId
  out_of_address_space_text  : StringId
```

**Invariants**

- There are **three icons, one per shipped game**, not one branded icon. The engine is a
  drop-in replacement for three different retail products and presents itself as whichever
  one it was launched against.
- The failure dialog's fields are fixed and named: a **file**, a **line**, a
  **description**, a **call stack** list, and a **stop** control alongside the default
  continue. That field set is the contract the assertion reporter fills in, and it is the
  reason the dialog is defined here rather than built at runtime — it must be displayable
  when the process is already in a bad state.
- Two of the identifiers name the **out-of-address-space** message specifically. The engine
  checks, before it commits to loading a level, that it can still reserve a large
  contiguous region, and this is the message shown when it cannot. See
  [Platform assumptions — Memory](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions).

**Notes** — the numeric values themselves are arbitrary and carry no meaning beyond
uniqueness; several are left over from an editor's automatic numbering and address nothing.
A rebuild should name these resources rather than number them, and keep only the
grouping above.
