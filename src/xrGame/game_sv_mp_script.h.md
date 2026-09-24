# src/xrGame/game_sv_mp_script.h

> A multiplayer game mode whose rules are supplied entirely from script: the engine provides the session plumbing and declines to decide anything.

**Needs** — [`game_sv_mp.h`](game_sv_mp.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`game_sv_mp_script.cpp`](game_sv_mp_script.cpp.md) · [`xrServer_process_spawn.cpp`](xrServer_process_spawn.cpp.md)
**Tier floor** — T2: a game mode that is a set of overrides, most of them empty

## Purpose

The shipped multiplayer modes (deathmatch, team deathmatch, artefact hunt) each hard-code a
scoring and round policy. This one hard-codes none: it inherits the whole multiplayer
session — players, teams, connection handling, phase machine — and overrides every rule hook
with an empty body or a permissive answer, leaving the rules to be written in script against
the class exported here. It is the extension point rather than a mode anybody plays.

There is no `.cpp` in this slice; the file is both the declaration and, by way of what each
override does *not* do, the specification.

## State

`Stateless` beyond what the multiplayer session base owns.

## `game_sv_mp_script`

**Contract** — a multiplayer session whose type name and rule set come from script. Its
defaults, which are the load-bearing part:

- **Update** — passes straight through to the base session; no per-frame rule runs.
- **Player connect / disconnect** — handled (the session must still track who is present),
  unlike the scoring hooks.
- **Kill and hit notifications** — accepted and discarded. Script is expected to observe
  these through its own event surface if it wants them.
- **Touch** — always permits ownership transfer. The engine's default multiplayer modes
  refuse pickups by rule (wrong team, wrong phase); this mode refuses nothing.
- **Detach** — no rule.
- **Player state creation** — the base's default record, with no mode-specific extension.

**Invariants** — the permissive `Touch` answer is the one override a rebuild must not
"improve" into a deny-by-default. A script mode that has not yet installed a pickup rule
must still let players pick things up, or the mode is unplayable before its script loads.

**Notes** — three small helpers pack and unpack a hit's power and impulse into and out of a
network message. They exist here because the hit message's payload layout is shared with the
other multiplayer modes and is part of the frozen wire format; a rebuild should own that
packing in one place rather than in each mode.

The class is registered with the script virtual machine so a mod can derive from it. That
registration is what the whole file is for.
