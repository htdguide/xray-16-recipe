# src/xrGame/ai/monsters/pseudodog/psy_dog_aura.h

> Declares the psy dog's ambient dread: a fading screen treatment on the player, and the rule that decides when it should be on.

**Needs** — [`pp_effector_custom.h`](../../../pp_effector_custom.h.md) · [`psy_dog_aura.cpp`](psy_dog_aura.cpp.md)
**Used by** — [`psy_dog.cpp`](psy_dog.cpp.md) · [`psy_dog.h`](psy_dog.h.md) · [`psy_dog_aura.cpp`](psy_dog_aura.cpp.md)
**Tier floor** — T2: a fade factor per scheduled tick, handed to the camera stack

## Purpose

Declares the surfaces implemented in [`psy_dog_aura.cpp`](psy_dog_aura.cpp.md). Two objects, deliberately split: one is a screen effect that knows only how to fade itself in, hold, and fade out; the other is the creature-side rule that decides whether the effect should exist at all.

The split is load-bearing. The effect is owned by the player's camera stack and outlives any single decision; the rule is owned by the creature and runs on the creature's schedule. Merging them would put a per-creature decision inside something the camera destroys at will.

## `PsyDogAuraEffector`

A three-phase timed screen effect — fading in, holding, fading out — carrying the phase, when the phase began, and how long a fade takes. It exposes one command beyond the effector contract: *switch off*, which moves it from any phase into the fade-out.

## `PsyDogAura`

The creature-side controller. It holds the creature, the player, and two timestamps: when the player last saw one of this creature's phantoms, and when one of those phantoms last saw the player. It is reinitialised with the creature, torn down on the creature's death, and asked once per scheduled creature tick whether the effect should be running.

**Notes** — the tuned look of the effect (desaturation, blur, noise, colour ramps) is loaded from a configuration section named in the creature's own section, not fixed here. The creature names the section; the shared effector-controller machinery reads it.
