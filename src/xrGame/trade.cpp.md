# src/xrGame/trade.cpp

> The trading session: who the two parties are, and the fact that one is open.

**Needs** — [`trade.h`](trade.h.md) · [`Actor.h`](Actor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai/trader/ai_trader.h`](ai/trader/ai_trader.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`trade.h`](trade.h.md)
**Tier floor** — T2: a two-party session record.

## Purpose

Every inventory owner carries one of these. It holds the *session*: am I trading, with whom,
and what kind of party am I. The arithmetic of a trade — eligibility, pricing, the transfer
itself — is in [`trade2.cpp`](trade2.cpp.md), and the split between the two files is
arbitrary; a rebuild should merge them.

The decision worth carrying over is the **party classification**. A trading party is one of
three kinds — the player, a dedicated trader, or an ordinary stalker — and which kind it is
changes the pricing (a trader values artefacts by its own table), the callbacks (only a
trader is told a trade opened) and what the player is told (only a trade involving the
player fires a script callback). Classification happens once, at construction for one's own
side and at partner selection for the other's, rather than at each use.

## State

```text
RECORD Trade
  self          : Party
  partner       : Party                     # kind `none` when no session
  in_session    : bool
  last_trade_at : int (ms)                  # server clock at session start
  artefact_tasks_dirty : bool               # a trader bought an artefact; tell the server
  nearby        : list<GameObject>          # scratch for the eligibility scan

RECORD Party
  kind      : none | trader | stalker | player
  entity    : optional<reference to Entity>
  inventory : optional<reference to InventoryOwner>
```

**Invariants** — `kind` is `none` exactly when there is no partner; the two other fields are
cleared together with it, so a stale entity reference can never outlive a session. A party's
kind is never re-derived: once classified, it stands for the life of the object.

## Construction

**Contract** — classifies the owner it belongs to, in a fixed order: dedicated trader first,
then the player, then an ordinary stalker. An owner matching none of the three is left
unclassified and cannot trade.

**Invariants** — the order is load-bearing because the classes overlap: a dedicated trader is
also a stalker by derivation, so testing for stalker first would classify every trader as a
stalker and lose the trader-specific pricing and callbacks. A rebuild whose party kinds are
an explicit tag rather than a derived type has no ordering problem — which is the better
design, and the classification then disappears entirely.

## `SetPartner(entity)`

**Contract** — classify a candidate partner and record it. Same order as above, with one
extra condition: **the candidate must not be oneself**. Reports whether a partner was set.

**Invariants** — the self-check is applied to each of the three cases rather than once
before them, which means an entity that is oneself falls through to the next case and, at
the end, fails. The effect is the same as one check up front; the shape is an artifact.

## `StartTrade` / `StartTradeEx(owner)`

**Contract** — open a session. `StartTradeEx` additionally selects the partner first. Opening
stamps the **server** clock, not the local one, and lowers the artefact-tasks flag.

**Notes** — the server clock is the right one because trades have to be consistent between
the two sides of a multiplayer connection, and a session is a shared fact. The stamp is
written and, in the shipped code, never read — see below.

## `StopTrade`

**Contract** — close the session and forget the partner. Both, always: a session left open
with a partner cleared would let the next price query dereference nothing.

## `TradeCB(starting)`

**Contract** — notify one's own side that a session opened or closed, but **only if one is a
dedicated trader**. Ordinary stalkers and the player are not told.

**Notes** — this is the hook a trader uses to switch into its trading animation and dialog
state. That it exists as a separate call rather than being folded into `StartTrade` and
`StopTrade` is because the user-interface layer decides when the *screen* opens, which is
not the same moment as when the session record opens.

## `OnPerformTrade(money_in, money_out)`

**Contract** — after a completed exchange, fire a script callback on one's own side with the
two money totals — again, **only if one is a dedicated trader**. This is the hook a modder
uses to make a trader react to what it has just bought.

## `UpdateTrade`

**Contract** — empty. A per-cycle hook that does nothing in the shipped game; the session
needs no upkeep. A rebuild drops it.

## What could not be recovered

- The session start time is stamped and never read anywhere in the tree. It looks like the
  remains of a trade cooldown that was removed.
- The artefact-tasks flag is set when a trader buys an artefact (in
  [`trade2.cpp`](trade2.cpp.md)) and cleared when a session opens, but nothing reads it
  either. Its name says a server synchronisation was meant to be triggered by it.
- The inventory a trade draws from is selected through a helper that, in a disabled
  alternative beside it, drew a dedicated trader's goods from a *separate storage* rather
  than from its carried inventory. The shipped code uses the carried inventory for everybody.
  Whether the separate storage was abandoned or moved is not recoverable here.
