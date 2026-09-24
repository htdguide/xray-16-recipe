# src/xrGame/trade.h

> Declares the trading session and the three kinds of party.

**Needs** — [`trade.cpp`](trade.cpp.md) · [`trade2.cpp`](trade2.cpp.md)
**Used by** — [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`ai_trader.cpp`](ai/trader/ai_trader.cpp.md) · [`trade.cpp`](trade.cpp.md) · [`trade2.cpp`](trade2.cpp.md) · [`UITalkWnd.cpp`](ui/UITalkWnd.cpp.md)
**Tier floor** — T2: a session record.

## Purpose

Declares the surface implemented across [`trade.cpp`](trade.cpp.md) — the session — and
[`trade2.cpp`](trade2.cpp.md) — eligibility, pricing and transfer. The split between the two
implementation files is arbitrary and a rebuild should merge them; this header is the honest
statement of the type.

The declaration's substance is the **party record**: a kind tag, the entity, and the
inventory owner. Both sides of a trade are described by it, and a session is exactly a pair
of them plus a flag. Having the kind as a stored tag rather than re-derived at each use is
what makes pricing readable — the price formula asks "is my side the player" rather than
attempting a conversion.

## Exported units

- `CanTrade()` — is a trade possible right now: someone eligible nearby, at the right
  distance, facing me.
- `StartTrade()` / `StartTradeEx(owner)` / `StopTrade()` / `IsInTradeState()` — the session.
- `TradeCB(starting)` — tell one's own side a session opened or closed. Traders only.
- `OnPerformTrade(money in, money out)` — the script callback after an exchange. Traders only.
- `TransferItem(item, buying, free)` — move one item and the money for it, in the right
  order, with the right callbacks.
- `GetItemPrice(item, buying, free)` — the price formula.
- `GetPartner()` / `GetPartnerTrade()` / `GetPartnerInventory()` — the other side.
- `UpdateTrade()` — empty.
- privately, `SetPartner` / `RemovePartner` — partner classification, and the helper that
  chooses which inventory a party trades out of.
