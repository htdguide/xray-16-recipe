# src/xrGame/game_news.cpp

> Persists one news item: five fields in a fixed order, and nothing else.

**Needs** — [`game_news.h`](game_news.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`date_time.h`](date_time.h.md)
**Used by** — [`game_news.h`](game_news.h.md)
**Tier floor** — T3: field-by-field serialization

## Purpose

The implementation half of the news record, and it is almost empty: the record is plain
data, so only its persistence needs code. It is a separate file because the header is
included widely and the serialization pulls in the alife simulation.

## State

`Stateless.` The record's state is declared in [`game_news.h`](game_news.h.md).

## `save` / `load`

**Contract** — writes and reads five fields in one fixed order: type, caption, text, receive
time, texture name. Symmetric, with no version tag of its own — the containing save file's
version covers it.

**Invariants** — the order is the format. Note that it is **not** the declaration order: the
texture name is written last, after the timestamp, because it was added after the format was
already in use and appending was the only compatible place to put it. A rebuild must keep the
serialization order, not the record order.

The display duration is **not persisted**. A reloaded news item reverts to the default
duration, losing any value a script set. That is a genuine omission rather than a decision —
the field was added to the record and not to the format.

**Notes** — a commented-out formatter shows what the record was once expected to do: compose
a single display line as "hours:minutes, caption text", splitting the in-world timestamp into
calendar parts. It was moved into the screen layer, which is the right place for it; the
record stays pure data. The rectangle within the texture was likewise cut from the format,
and its serialization lines remain commented beside the live ones — evidence that the format
was trimmed rather than designed.
