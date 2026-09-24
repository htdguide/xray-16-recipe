# src/xrEngine/IInputReceiver.cpp

> Stack push and pop for an input receiver, and the synthetic release burst that runs when one loses the input.

**Needs** — [`IInputReceiver.h`](IInputReceiver.h.md) · [`xr_input.h`](xr_input.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Three delegations and one real algorithm. The algorithm is the one that keeps the game from getting stuck running forward when the player opens the console.

## `capture` / `release`

**Contract** — Push this receiver onto the input stack, or remove it. See [`xr_input.cpp`](xr_input.cpp.md) for the stack's rules.

## `on_deactivate`

**Contract** — The default behaviour when a receiver stops being the input's owner: synthesize a *release* event for every key, mouse button, gamepad button and gamepad axis that is currently held, delivering them to this receiver.

```text
FUNCTION on_deactivate()
  FOR EACH keyboard key that is down:        deliver keyboard release
  FOR EACH mouse button that is down:        deliver mouse release
  FOR EACH gamepad button that is down:      deliver controller release with a zeroed axis state
  FOR EACH gamepad axis that is deflected:   deliver controller release with a zeroed axis state
```

**Invariants** — Every press a receiver saw is matched by a release, even when the receiver stops receiving events between the two. Without this, a receiver that tracks held keys — which is most of them — is left believing a key is still down forever. The visible symptom is a player who keeps walking after opening the console.

**Notes** — The axis state accompanying the synthesized gamepad releases is zeroed rather than being the real current deflection. That is correct: the receiver is being told the control is *at rest as far as it is concerned*, not what the hardware is doing, because the hardware's state now belongs to whoever took the input.

**Notes** — The scan covers the entire key space rather than a tracked set of held keys, because the receiver does not keep one — the input layer does. It runs once per input-stack transition, which is rare, so the cost of the scan does not matter.

## `on_activate`

**Contract** — Default is to do nothing. There is no symmetric synthetic-press burst: a receiver that becomes active does *not* learn about keys that were already held when it arrived. That is deliberate — the player pressing a key to close the console must not have that same press act in the game underneath.
