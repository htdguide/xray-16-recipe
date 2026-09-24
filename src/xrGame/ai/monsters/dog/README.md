# src/xrGame/ai/monsters/dog — the blind dog

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The dog is the only creature in the game whose brain is written against its **pack** rather
than against itself. Its rest, attack, panic, feeding and danger-sound states all come from
the pack-aware library in [`../group_states/`](../group_states/README.md), which exists for
this creature and is used by nothing else.

Read the dog together with that directory. This page covers only what lives here.

## What is actually its own

**The territory latch.** A dog's pack carries one shared judgement: *is our territory under
threat*. Two independent things set it — an enemy standing inside the pack's inner home
radius, and an enemy within six units of any member — and once set, the whole pack attacks
instead of fleeing. That single shared bit is what turns a pack of individually timid animals
into something that will not back down, and it is the clearest piece of coordination in the
chapter. The six-unit distance is fixed in code.

**A numbered vocabulary of idle animations.** The dog carries a set of "flavour" clips
addressed by number, which the brain requests by index rather than by meaning. The pack
library's custom state exists to play them. This is how a resting pack looks varied without
the brain knowing what any of the animations depict.

**A jump repertoire gated twice**: on the dog's rank within the pack, and on the enemy being
above it. Low-ranked dogs do not leap, which keeps a pack from all leaping at once.

## What could not be recovered

- The six-unit "too close" distance that trips the territory latch is a bare constant.
- The mapping from animation *number* to what the clip depicts exists only in the game data.
  A rebuild that guesses will produce dogs that sniff when they should stretch.

## Twins

| Twin | Role |
|---|---|
| [`dog.cpp`](dog.cpp.md) | The blind dog: a pack creature whose distinctive machinery is a numbered vocabulary of idle "flavour" animations the brain can request by number, plus a jump repertoire gated on rank and on the enemy being above it. |
| [`dog.h`](dog.h.md) | Declares the blind dog, implemented in [`dog.cpp`](dog.cpp.md). |
| [`dog_script.cpp`](dog_script.cpp.md) | Exports the dog to the script layer as a constructible class deriving from the script-visible game object. |
| [`dog_state_manager.cpp`](dog_state_manager.cpp.md) | The dog's brain: the pack version of the standard priority selector, with a squad-wide "our territory is under threat" latch that converts fear into aggression, and hand-offs for the numbered-animation machine and the dragged corpse. |
| [`dog_state_manager.h`](dog_state_manager.h.md) | Declares the dog's brain, implemented in [`dog_state_manager.cpp`](dog_state_manager.cpp.md). |
