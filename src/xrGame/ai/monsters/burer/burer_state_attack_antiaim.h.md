# src/xrGame/ai/monsters/burer/burer_state_attack_antiaim.h

> Declares the state that hands the creature over to the shared anti-aim ability and waits.

**Needs** — [`state.h`](../state.h.md) · [`burer_state_attack_antiaim_inline.h`](burer_state_attack_antiaim_inline.h.md)
**Used by** — [`burer_state_attack_antiaim_inline.h`](burer_state_attack_antiaim_inline.h.md) · [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md)
**Tier floor** — T3: a wrapper around an ability

## Purpose

Declares the surface implemented in [`burer_state_attack_antiaim_inline.h`](burer_state_attack_antiaim_inline.h.md).

## `BurerAntiAimState`

A leaf state whose only field that is read is `allow_anti_aim`, the arbitration latch. Four further fields — a shield start timestamp, a particle cooldown, a clip length and a started flag — are declared and never touched; the declaration is a copy of the shield state's and was not trimmed.
