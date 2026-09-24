# src/xrGame/pda_space.h

> The two kinds of message the player's handheld device can carry.

**Needs** — _(none)_
**Used by** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`PdaMsg.h`](PdaMsg.h.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md)
**Tier floor** — T4: an enumeration

## Purpose

One enumeration in its own file so the device's screens, the dialogue system and the task
system can each name a message kind without including each other.

## State

`Stateless.`

## The message kind

```text
ENUM PdaMessageKind : int (32-bit)
  dialog     # a line of conversation, addressed to the player by someone
  info       # an informational notice, from the world rather than a person
  count      # the number of kinds; not a kind itself
```

**Invariants** — the trailing value is a count sentinel, used to size per-kind arrays. It must
remain last, and a rebuild adding a kind must add it before the sentinel.

The distinction between the two is *who is speaking*, not what is said: a dialog message comes
from a character and participates in the conversation history, an info message is the world
telling the player something. The device presents them differently.
