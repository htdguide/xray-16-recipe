# src/xrGame/smart_zone.h

> A restrictor volume that additionally asks to be woken every frame, so that a script can watch who is inside it.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_zone_script.cpp`](script_zone_script.cpp.md)
**Tier floor** — T2: a one-decision subclass of a registered client object

## Purpose

The base restrictor is a *passive* volume: it constrains where creatures may go and is
consulted by the pathfinder, but it has nothing to do per frame, so it declines scheduler
registration. A smart zone is the same volume with one answer flipped — it **does**
register with the scheduler, so it receives a per-frame update and its entered/exited
events reach the script layer promptly.

That single override is the entire file. It is worth its own type because the decision it
encodes — "this volume is watched, that one is merely consulted" — has a real cost:
scheduler slots are a budgeted resource (see the scheduler entry in the glossary), and
making every restrictor updatable would spend that budget on volumes nobody observes.

## State

`Stateless.` — everything is inherited from the restrictor; see
[`space_restrictor.cpp`](space_restrictor.cpp.md).

## `CSmartZone`

**Contract** — a restrictor client object whose answer to "should I be registered with the
scheduler" is always yes. Everything else — shape loading, spawn, net-update, save/load,
the inside/outside test — is the restrictor's.

**Notes** — the file also declares that this class is exported to the script layer under
the game-object facade. The export itself is elsewhere; what matters to a rebuild is that
a smart zone is script-visible and a plain restrictor's script surface is the same one, so
scripts distinguish the two by which they spawned, not by a different interface.
