# src/xrGame/game_sv_artefacthunt_process_event.cpp

> Artefact hunt's two extra server events: a player entered or left a team's base.

**Needs** — [`game_sv_artefacthunt.h`](game_sv_artefacthunt.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md)
**Tier floor** — T3: a dispatch on an event type

## Purpose

One dispatch function in a file of its own — a file split by the original for compilation
reasons rather than design ones, and a rebuild should fold it into
[`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md). It is listed because the mirror
must be complete.

## State

`Stateless.`

## `OnEvent`

**Contract** — handles the two events artefact hunt adds to the inherited set, and passes
everything else to the base. Each carries a player's entity identifier and a base
identifier; the handlers do the work.

```text
FUNCTION on_event(packet, type, time, sender)
  IF type is entered_team_base THEN
    on_enter_base(packet.read_int16, packet.read_int8)
  ELSE IF type is left_team_base THEN
    on_leave_base(packet.read_int16, packet.read_int8)
  ELSE base.on_event(packet, type, time, sender)
```

**Invariants** — the base identifier is used **as it arrives**, with no adjustment. Compare
with
[`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md),
which subtracts one from the same field for the same event: the two modes disagree about
whether the base identifier authored into the level is one-based or zero-based. The level
data is one-based; artefact hunt therefore reads it one too high, and its own base handling
compensates elsewhere. A rebuild should normalize once, at the level loader, and delete both
conventions.
