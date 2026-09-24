# src/xrGame/ui/UIListItemServer.cpp

> One row of the server browser: three status icons and six text columns laid out left to right from authored widths, each column's text cut to fit, and the row able to compose the console command that joins its server.

**Needs** — [`UIListItemServer.h`](UIListItemServer.h.md) · [`UIEditKeyBind.cpp`](UIEditKeyBind.cpp.md) · [`xrUICore/ListBox/UIListBoxItem.h`](../../xrUICore/ListBox/UIListBoxItem.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`UIListItemServer.h`](UIListItemServer.h.md)
**Tier floor** — T3.

## Purpose

A row in the server list. The list itself is fed by a matchmaking service that no longer
exists — see
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
— but the row's own contract is worth keeping, because it is where the browser's *display
shape* and its *join action* are both defined.

## State

```text
RECORD ServerRow EXTENDS ListRow
  icons   : password-required, dedicated, account-required
  columns : name, map, game mode, player count, latency, version
  info    : the row's whole data record, kept so the join command can be composed later

RECORD ServerInfo
  name, address, map, game, players, ping, version : text
  icons  : (password, dedicated, punkbuster, account)   # four flags, three drawn
  index  : int    # the row's position in the service's own result set
```

**Invariants**

- The row keeps its whole information record, not just what it displays. The **address** is
  never shown and is the only field the join command needs.
- The row's tag is set to the service's own result index, so the browser can map a selected
  row back to its result without searching.

## Layout

**Contract** — icons first, then columns, each placed at the running sum of its predecessors'
widths. Icon size is taken from the *actual measured height of one icon texture*, scaled by
the runtime horizontal aspect factor, and every icon is square at that size and vertically
centred in the row. Column widths come from the caller.

**Notes** — measuring the icon rather than authoring its size is how the row stays correct
across the icon sets the three games ship at different resolutions; chapter 15 makes the same
point about texture coordinates being normalised from a page's reported size.

The aspect factor is applied to the icon's **size**, not only to its position, so icons stay
square in screen pixels and become non-square in canvas units. The columns are not corrected
at all.

Three of the four status flags are drawn. The anti-cheat icon's column is commented out and
its slot reused by the account-required icon — **which still loads the anti-cheat texture**.
A player therefore sees the anti-cheat symbol where the account symbol was meant. Recorded as
a live cosmetic defect.

## Filling a row

**Contract** — name, map and game mode are passed through the **localization table** and then
**cut to fit their column**; player count, latency and version are shown verbatim. The three
icons are shown or hidden from the flags. The row's tag becomes the service index.

**Notes** — localizing the server's *name* is surprising and deliberate: the shipped official
servers were named by identifier and the table turned those into display names. A name with no
table entry passes through unchanged, which is what happens for every third-party server.

Cutting is the same trailing-character trim the key-binding cell uses, shared as a free
function declared locally here rather than in a header — see
[`UIEditKeyBind.h`](UIEditKeyBind.h.md). It is called with the **map column's** font for the
server name rather than the server column's; the two are the same font in every shipped
layout, so the bug is invisible.

## Composing the join

**Contract** — build the console command that connects to this row's server: the client verb,
the server's address, and the player's name, account password and server password as named
parameters in one parenthesised argument.

**Notes** — the connection parameters travel as **one string with an ad-hoc key/value
syntax**, because the console takes one argument. The four keys are frozen against the shipped
console command's parser. This is the point at which the browser becomes a connection; the
row composes it and someone else executes it, which is the chapter's event-not-call rule once
more.
