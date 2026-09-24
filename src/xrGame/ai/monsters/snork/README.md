# src/xrGame/ai/monsters/snork — the snork

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

Three small things make a snork: it leaps, it senses the player through walls, and it snarls
once before a fight — half the time.

## What is actually its own

**The pounce**, which is not its own code at all: it is the shared leap ability, configured.
The snork's contribution is the tuning and the clip triple.

**Sensing the player through walls.** The snork answers the base's "can I feel at a distance"
question yes, which admits it to a perception path that does not require line of sight. It is
the reason a snork in a tunnel is already coming for you.

**A coin-flip snarl.** On the tick combat begins, the brain arms a pre-fight vocalisation,
and it plays on about half of all fights. Once armed it fires once and does not rearm for
that engagement. The randomness is deliberately cosmetic: a snarl every time becomes a
warning the player relies on, and never becomes nothing.

## What could not be recovered

- **`snork_jump.cpp` is entirely commented out.** A snork-specific leap controller was
  written, with fields that survive and a body that does not, and nothing constructs it. What
  is legible from the remains is a *flanking* pounce — a leap that arrives to one side rather
  than head-on — which the shipped creature does not have.

## Twins

| Twin | Role |
|---|---|
| [`snork.cpp`](snork.cpp.md) | A creature defined by three small things: it leaps, it senses the player through walls, and it snarls before a fight — but only half the time, and only once. |
| [`snork.h`](snork.h.md) | Declares the snork: a leaping creature whose distinctiveness is the pounce, a coin-flip snarl before each fight, and the ability to sense the player through walls. |
| [`snork_jump.cpp`](snork_jump.cpp.md) | Entirely commented out. The only thing recoverable from it is the flanking pounce the snork was going to have and does not. |
| [`snork_jump.h`](snork_jump.h.md) | A snork-specific leap controller that was abandoned: the fields survive, every method is commented out, and nothing constructs it. |
| [`snork_script.cpp`](snork_script.cpp.md) | Registers the snork with the script layer as a named type deriving from the script-visible game object. |
| [`snork_state_manager.cpp`](snork_state_manager.cpp.md) | The snork's brain: the baseline selector with caution ahead of curiosity, plus the one line that arms the pre-fight snarl on the tick combat begins. |
| [`snork_state_manager.h`](snork_state_manager.h.md) | Declares the snork's brain: eight registered states and a selector that arms the pre-fight snarl. |
