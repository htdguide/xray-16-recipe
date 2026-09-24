# src/xrGame/object_property_evaluators.h

> Declares the eleven observers through which the object-handling planner reads the real state of a weapon or a thrown object — implemented in [`object_property_evaluators.cpp`](object_property_evaluators.cpp.md).

**Needs** — [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`object_property_evaluators_inline.h`](object_property_evaluators_inline.h.md)
**Used by** — [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md) · [`object_property_evaluators.cpp`](object_property_evaluators.cpp.md) · [`object_property_evaluators_inline.h`](object_property_evaluators_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the evaluator classes the object-handling planner installs per item. Substance is in
[`object_property_evaluators.cpp`](object_property_evaluators.cpp.md); the templated base is in
[`object_property_evaluators_inline.h`](object_property_evaluators_inline.h.md).

Exported units:

- `CObjectPropertyEvaluatorBase` — the base: an evaluator that holds an item and the creature.
- Three aliases reused across the planner: the constant evaluator, the shared-storage member
  evaluator, and the base narrowed to a plain game object.
- `CObjectPropertyEvaluatorState` — is the weapon in a named state (or not, by a flag).
- `CObjectPropertyEvaluatorWeaponHidden` — is the weapon effectively out of the hands.
- `CObjectPropertyEvaluatorAmmo` — does the creature carry ammunition for this barrel.
- `CObjectPropertyEvaluatorEmpty` — is the magazine empty.
- `CObjectPropertyEvaluatorFull` — is the magazine at capacity.
- `CObjectPropertyEvaluatorReady` — can the weapon fire right now.
- `CObjectPropertyEvaluatorQueue` — has a burst finished.
- `CObjectPropertyEvaluatorNoItems` — are the creature's hands effectively empty.
- `CObjectPropertyEvaluatorMissile` — is the thrown object in a named state.
- `CObjectPropertyEvaluatorMissileStarted` — has the throw gesture begun.
- `CObjectPropertyEvaluatorMissileHidden` — is the thrown object out of the hands.
