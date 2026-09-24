# src/xrGame/ui/CExtraContentFilter.cpp

> Answers "may this player see this piece of content", for bonus material that shipped locked.

**Needs** — [`CExtraContentFilter.h`](CExtraContentFilter.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`CExtraContentFilter.h`](CExtraContentFilter.h.md)
**Tier floor** — T3: table lookup plus one per-machine settings read

## Purpose

Some retail editions shipped extra multiplayer content — skins, maps — that is present in the
game data but must stay invisible unless the player's machine carries the proof of purchase.
This module is the single predicate the menus consult. It is a separate file because the
policy is one sentence long and the screens that obey it are many.

## State

```text
RECORD Pack
  name    : text        # the configuration section that lists the pack's contents
  enabled : bool        # resolved once at construction from the per-machine settings store
  content : list<text>  # the names this pack unlocks

RECORD Filter
  packs : list<Pack>
```

Invariant: a name appears in at most one pack. The configuration is authored that way and
nothing re-checks it, so a name listed twice silently takes the first pack's verdict.

## `IsDataEnabled`

**Contract** — Given a content name, return whether it may be shown. Linear scan of every
pack's content list; the first pack that lists the name decides. **A name that appears in no
pack is enabled** — the filter is a deny-list of locked content, not an allow-list of
permitted content, so content that predates the mechanism keeps working.

```text
FUNCTION is_data_enabled(name) -> bool
  FOR EACH pack IN packs
    IF name IN pack.content THEN RETURN pack.enabled
  RETURN true                      # unknown content is not extra content
```

## construction

**Contract** — Reads a configuration section whose every line is `pack name = settings key`.
For each line it records the pack and asks the per-machine settings store for that key,
treating the integer value 1 and nothing else as unlocked. It then reads the pack's own
section, whose line *names* (values ignored) are the content the pack unlocks. A pack whose
own section is absent unlocks nothing but still exists, which is how a pack is soft-disabled
without editing the table.

**Notes** — The per-machine settings store is a platform service the recipe treats as part of
[Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services);
the original reaches it through a Windows-only path, so on every other platform the lookup
fails, every pack resolves to locked, and the extra content stays hidden. That is a bug the
source itself flags rather than a decision, and a rebuild should pick a portable store.

The verdict is resolved **once, at construction**. Unlocking content while the game runs has
no effect until the filter is rebuilt.
