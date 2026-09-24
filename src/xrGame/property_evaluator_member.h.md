# src/xrGame/property_evaluator_member.h

> An evaluator that answers by comparing *another* question's answer to a literal — the planner's way of aliasing and negating conditions.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md) · [`property_storage.h`](property_storage.h.md) · [`property_evaluator_member_inline.h`](property_evaluator_member_inline.h.md)
**Used by** — [`agent_manager_properties.h`](agent_manager_properties.h.md) · [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`property_evaluator_member_inline.h`](property_evaluator_member_inline.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md)
**Tier floor** — T3: one lookup and two comparisons

## Purpose

Two plans often need the same fact under different names, or need a fact's *negation*, or
need a fact answered against a *different creature's* answer board. Writing a fresh
evaluator for each of those would duplicate the measurement. This one reads an existing
answer out of a property storage, compares it to a literal, and reports whether the
comparison matched — which covers aliasing (`equality = true`, value = true), negation
(`equality = false`), and cross-subject questions (by handing it a foreign storage).

## State

```text
RECORD MemberEvaluator             # extends Evaluator
  question : int    # which question to read out of the storage
  value    : bool   # what to compare its answer against
  equality : bool   # true: answer must equal `value`; false: it must differ
  # inherited `storage` may be pinned at construction, overriding the planner's
```

**Invariant** — when a storage is supplied at construction it **wins over** the one the
planner offers at `setup`. That is the whole mechanism for asking a question about another
creature: pin its answer board and the planner's own board is ignored for this evaluator.

## `evaluate`

**Contract** — reads `question` from the bound storage and returns whether the match
matched the requested polarity. Fails if the question has never been answered.

```text
FUNCTION evaluate() -> bool
  RETURN (storage.property(question) == value) == equality
```

## `setup`

**Contract** — binds the subject as usual, but substitutes the construction-time storage
when one was pinned. See the invariant above.

**Notes** — the double comparison reads oddly but is exactly right: the inner comparison is
the question, the outer one is the polarity. A rebuild writing this as
`if equality then a == b else a != b` produces identical behaviour and reads better.
