# src/xrGame/ai/monsters/burer — the burer

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The burer is the chapter's clearest example of an **arbitrated ability set**. It has four
distinct attacks plus a defensive posture, and the interesting part is not any one of them
but the fixed order in which a single test each tick decides between them.

## The arbitration

Every tick, in this order: gravity wave if it is off cooldown; shield if the creature has
just been hurt; anti-aim if the player is drawing a bead on it; telekinesis if there is
anything loose to throw; otherwise reposition and keep facing the player.

The ordering is the design. Offence outranks defence, defence outranks evasion, and the
fallback is never "stand still" — a burer with nothing to do is always moving to a better
distance. Three range bands decide where: close in, back off, or sidestep somewhere random.

## The abilities

**A travelling gravity wave** that walks across the floor toward its victim. The wind-up
doubles as the weapon's range indicator: the further away the enemy, the longer the burer
holds the charge before releasing. A player can read the distance from the animation, which
is the only ranged attack in the game that telegraphs its own reach.

**Telekinesis.** Sweep the space around the enemy, around the creature, and around the
midpoint between them for anything loose within a mass window; lift it; throw it at the
enemy's head, one at a time. A separate, unconditional sweep snatches any live grenade
nearby — which is what makes a burer immune to the player's most obvious counter.

**A bullet shield** that consumes fire-wounds for an authored time, dropped early if the
player is reloading.

**Anti-aim**, the shared evasion ability, entered by a state that does nothing but start it
and wait.

**A scanning trance** no other creature has: after sensing something at range the burer
stops and scans, which is both a tell and a vulnerability.

## What could not be recovered

- **One melee state sits in the attack tree's dispatch table and is never selected.** It is
  the shared close-quarters attack, fenced by two distances; no branch of the arbitration
  reaches it. A burer therefore has no answer to an enemy that closes all the way in, which
  is either the design or the bug — the source does not say which.
- **The instant close-range gravity strike is wired into the creature's control slots and
  never activated.** Unlike the melee state it is fully implemented and would work; nothing
  turns it on.
- About thirty authored numbers sit behind the three physical abilities. They are data, but
  the *relationships* between them — which cooldown must exceed which wind-up for the
  arbitration to stay stable — are nowhere written down and must be rediscovered by testing.

## Twins

| Twin | Role |
|---|---|
| [`burer.cpp`](burer.cpp.md) | The burer's definition and its three physical abilities: a gravity wave that walks across the floor toward its victim, a bullet shield that eats fire-wounds, and a stamina drain that knocks the weapon out of the player's hands. |
| [`burer.h`](burer.h.md) | Declares the burer: a creature that fights with telekinesis, a travelling gravity wave and a bullet shield, with about thirty authored numbers behind those three abilities. |
| [`burer_fast_gravi.cpp`](burer_fast_gravi.cpp.md) | A gravity strike with no travel and no wind-up: face the enemy, and the moment the animation reaches its break point, hit. |
| [`burer_fast_gravi.h`](burer_fast_gravi.h.md) | Declares a close-range instant gravity strike, wired into the burer's control slots and never activated. |
| [`burer_script.cpp`](burer_script.cpp.md) | Exposes the burer class to the script layer under its frozen name. |
| [`burer_state_attack.h`](burer_state_attack.h.md) | Declares the burer's attack tree: the arbiter that picks between gravity, telekinesis, shield, anti-aim and plain repositioning. |
| [`burer_state_attack_antiaim.h`](burer_state_attack_antiaim.h.md) | Declares the state that hands the creature over to the shared anti-aim ability and waits. |
| [`burer_state_attack_antiaim_inline.h`](burer_state_attack_antiaim_inline.h.md) | Twelve lines of behaviour: start the anti-aim ability, stand and face the enemy while it runs, finish when it finishes. |
| [`burer_state_attack_gravi.h`](burer_state_attack_gravi.h.md) | Declares the gravity attack: charge, hold for longer the further the enemy is, release a wave. |
| [`burer_state_attack_gravi_inline.h`](burer_state_attack_gravi_inline.h.md) | The wind-up is the weapon's range indicator: the further away the enemy, the longer the burer holds the charge before letting the wave go. |
| [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) | The burer fight, arbitrated in a fixed order every tick: gravity if it is off cooldown, shield if it has just been hurt, anti-aim if the player is drawing a bead, telekinesis if there is anything to throw — and otherwise keep its distance and face the player. |
| [`burer_state_attack_melee.h`](burer_state_attack_melee.h.md) | Declares a close-quarters attack the burer's attack tree registers and never selects. |
| [`burer_state_attack_melee_inline.h`](burer_state_attack_melee_inline.h.md) | The shared close-quarters attack, fenced by two distances — and unreachable, because nothing selects it. |
| [`burer_state_attack_run_around.h`](burer_state_attack_run_around.h.md) | Declares the burer's repositioning move: pick a spot, run to it, keep facing the enemy on arrival. |
| [`burer_state_attack_run_around_inline.h`](burer_state_attack_run_around_inline.h.md) | Three range bands decide where a burer goes when it cannot attack: close in, back off, or sidestep somewhere random. |
| [`burer_state_attack_shield.h`](burer_state_attack_shield.h.md) | Declares the bullet shield as a timed behaviour state. |
| [`burer_state_attack_shield_inline.h`](burer_state_attack_shield_inline.h.md) | Put the shield up, stand and face the player behind it, and drop it when the authored time runs out — or early, if the player is reloading. |
| [`burer_state_attack_tele.h`](burer_state_attack_tele.h.md) | Declares the telekinetic attack: find loose objects, lift them, throw them one at a time — and catch live grenades on the way past. |
| [`burer_state_attack_tele_inline.h`](burer_state_attack_tele_inline.h.md) | Sweep the space around the enemy, the creature and the midpoint between them for anything loose in a mass window, lift it, and throw it at the enemy's head — with a separate, unconditional sweep that snatches any live grenade nearby. |
| [`burer_state_manager.cpp`](burer_state_manager.cpp.md) | The burer's mood chart, with one state no other creature has: a scanning trance it drops into after sensing something at range. |
| [`burer_state_manager.h`](burer_state_manager.h.md) | Declares the burer's top-level state selector. |
