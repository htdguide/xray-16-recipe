# src/xrGame/ai_sounds.cpp

> The display-name table for sound *kinds*: the mapping from the AI-perception sound classification to the text a designer or a debug view sees.

**Needs** — [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a static name table.

## Purpose

Every sound the game emits carries an AI classification alongside its audio — what kind of
event made it — because creatures react to the *kind*, not to the waveform. The kinds
themselves are declared elsewhere as a bitfield; this file supplies their human-readable
names, so that configuration and tooling can name a kind in text and debug views can print
one.

It is a separate file from the declarations because a name table is data and the
declarations are used in headers everywhere.

## State

```text
# A name-to-value table, terminated by an empty entry, covering:
#   undefined
#   item:    picking up, dropping, taking, hiding, using
#   weapon:  shooting, empty click, bullet hit, recharging
#   creature: dying, injuring, step, talking, attacking, eating
#   anomaly: idle
#   world object: breaking, colliding, exploding
#   world: ambient
```

**Notes** — The table's variable name says *anomaly type* while its contents are sound
kinds; the name is wrong and nothing depends on it.

The displayed names use "NPC" where the underlying identifiers say "monster": in this
engine the *monster* classification covers every non-player creature including human
stalkers, and the display strings were softened. A rebuild should pick one word; the
identifiers are the ones the configuration files use.

Several kinds declared in the header are absent from this table — it is a display
convenience, not the authoritative enumeration, and a lookup that misses simply has no
name.
