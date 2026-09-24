# src/xrPhysics/PHDisabling.h

> Declares the two-window sleep detector and the three mixes of it — translational,
> rotational, and both.

**Needs** — [`PHDisabling.cpp`](PHDisabling.cpp.md) · [`DisablingParams.h`](DisablingParams.h.md)
**Used by** — [`PHCharacter.h`](PHCharacter.h.md) · [`PHDisabling.cpp`](PHDisabling.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHElementNetState.cpp`](PHElementNetState.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md)
**Tier floor** — T2: accumulators and comparisons, once per body per step.

## Purpose

Declares the surface implemented in [`PHDisabling.cpp`](PHDisabling.cpp.md). Anything that
owns a body inherits one of these and gets sleep management for free; the inheriting class
supplies the body, and what "disable" and "re-enable" mean for it.

## Exported units

- **`SDisableVector`** — one motion accumulator: the running sum of per-step changes plus
  the previous sample. Reports the magnitude of a single step's change and of the whole
  window's.
- **`SDisableUpdateState`** — the verdict of one window: a *may sleep* flag and a *must
  wake* flag, which are not complements. Combining two windows is `and` on may-sleep and
  `or` on must-wake, which is the pessimistic rule — see the implementation.
- **`CBaseDisableData`** — the shared machinery: the step counter, the two window verdicts,
  the current sleep state, and the once-per-step driver. Demands from an implementor: the
  body, what to do on sleep, on wake, and how to sample the short and long windows.
- **`CPHDisablingBase`** — adds the two accumulators, the thresholds, and the threshold
  comparison with its hysteresis.
- **`CPHDisablingTranslational`** — samples position and linear velocity.
- **`CPHDisablingRotational`** — samples orientation and angular velocity.
- **`CPHDisablingFull`** — both, requiring *both* to agree before sleeping and *either* to
  object before staying asleep.

## Notes

The character controller uses the translational form alone
([`PHCharacter.h`](PHCharacter.h.md)): a character's body has its rotation pinned upright
every step, so a rotational test on it would measure nothing. Rigid-body shell elements use
the full form. The rotational form alone has no shipped user.
