# src/xrGame/trade_factors_inline.h

> Construction and reads of the price multiplier pair, each guarded against an invalid
> number.

**Needs** — [`trade_factors.h`](trade_factors.h.md)
**Used by** — [`trade_factors.h`](trade_factors.h.md)
**Tier floor** — T2: two fields and four validity checks.

## Purpose

The whole body of [`trade_factors.h`](trade_factors.h.md). A rebuild puts these on the type
and deletes this file; what survives is the checking policy.

## `construct(friend_factor, enemy_factor)`

**Contract** — stores both, checking each is a valid real number first.

## `friend_factor()` / `enemy_factor()`

**Contract** — the stored value, checked again on the way out.

**Notes** — checking on read as well as on write looks redundant and is not quite: these
records are held inside containers that are rebuilt when configuration is reloaded, and a
read of a freed or partly-written entry is what the second check catches. In diagnostic
builds only; the shipped build is a field read.

The reason the checks exist at all is that an invalid factor does not crash. It multiplies
into a price, the price becomes invalid, the floor of an invalid price is an arbitrary
integer, and the player's money changes by an arbitrary amount — which surfaces hours later
as a save game with impossible wealth. A rebuild in a language with no invalid-number state
for reals has nothing to check; one that has should check here, because this is the last
place the value is still identifiable.
