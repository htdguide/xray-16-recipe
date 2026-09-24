# src/xrGame/agent_manager_properties_inline.h

> The three squad evaluators' constructors, each a pass-through to the generic base.

**Needs** — [`agent_manager_properties.h`](agent_manager_properties.h.md)
**Used by** — [`agent_manager_properties.h`](agent_manager_properties.h.md)
**Tier floor** — T3: construction.

## Purpose

Each of the three squad evaluators is constructed with the squad manager it reads and a
name used in plan traces; both are simply forwarded to the generic evaluator base. The
file exists only because the original language wants inline definitions after the class
bodies; a rebuild has nothing to put here.

## Constructors

**Contract** — `EvaluatorItem`, `EvaluatorEnemy` and `EvaluatorDanger` each take an
optional squad manager (absent by default, filled by the planner at registration) and an
optional trace name, and store both. No other effect.
