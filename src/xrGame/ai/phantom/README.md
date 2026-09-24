# src/xrGame/ai/phantom — apparitions

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
See the [chapter opener](../README.md) for the creature machinery; the phantom uses none of it.

A phantom is a ghost that flies at the player, and it is in this chapter only because it is
an entity that moves toward someone. It has no perception, no navigation, no brain, no
memories, no pack and no state manager. Its whole existence is four presentation states in
sequence — born, flying, contact or shot, gone — driven by animation completion callbacks
rather than by any decision.

## What makes it different, and why that matters

**It targets the player unconditionally.** Not an enemy chosen by an enemy manager: the
player, looked up once at spawn and again whenever the reference is found missing. A phantom
cannot be distracted, cannot lose its target and will not attack anything else.

**It steers, it does not path.** Each update it turns toward the player at an authored
angular rate and advances at an authored speed. The navigation mesh is not consulted; walls
are not consulted. A phantom flies through the level.

**It is invisible to the AI and deaf.** At load it removes itself from the spatial
categories that make an object visible to creature perception and reactive to sound. No
creature can see it, no creature can hear it, and it can hear nothing. It exists only for
the player.

**It dies to anything.** Its health is set to a thousandth of a unit at spawn, so any hit at
all kills it. That is how "shoot the ghost and it bursts" is implemented — not with a
special case, but by making it so fragile that the ordinary damage path always finishes it.

**Contact is a hit on the player, self-inflicted.** When the phantom's bounding sphere
overlaps the player's it switches to the contact state and *hits itself* with a large
fire-wound, which is what destroys it; the damage to the player is a separate psychic hit
delivered when that state ends. So the phantom's death and the player's damage are two
independent events that merely coincide, and a rebuild that fuses them will change what the
death effects look like.

**It destroys itself.** Every terminal state leads to an idle state whose only action is to
remove the object. Nothing has to track a phantom it created.

## How a phantom is configured

The three numbers — flight speed, turn rate, and the strength of the psychic hit on
contact — come from the configuration section, as do a particle effect and a sound for each
of the four states. The *animations* do not: the four clips are looked up on the model by
fixed names (`birth_0`, `fly_0`, `contact_0`, `shoot_0`), so a model without those clips
cannot be a phantom. The three non-looping clips are required to stop at their end, because
their completion callback is what advances the state.

When the spawn record names no model, one is drawn at random from a list in the section and
the choice is broadcast so that every client sees the same apparition.

Not to be confused with the psy dog's illusions in
[`monsters/pseudodog/`](../monsters/pseudodog/README.md), which *are* full creatures with
brains of their own.

## What could not be recovered

- The contact impact is dealt as a fire-wound of a thousand power with a hundred impulse,
  both fixed in code. The numbers only have to exceed the phantom's own thousandth of a unit
  of health, so their size is arbitrary; nothing says why these two.
- A note in the spawn path calls the killer-identity reset a workaround for a crash with
  dynamically created phantoms. What the underlying fault was is not recoverable.

| Twin | Role |
|---|---|
| [`phantom.cpp`](phantom.cpp.md) | An apparition on rails: born, flies at one target under a turn-rate limit, and ends on contact or on being shot — with every phase's length decided by its animation. |
| [`phantom.h`](phantom.h.md) | Declares the phantom: a scripted apparition that flies at the player, does psychic damage on contact, and dies to any hit. |
