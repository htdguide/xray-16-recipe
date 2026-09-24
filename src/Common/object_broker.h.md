# src/Common/object_broker.h

> The single name a consumer includes to get the whole serialization-and-lifetime vocabulary — load, save, clone, destroy, compare.

**Needs** — [`object_interfaces.h`](object_interfaces.h.md) · [`object_type_traits.h`](object_type_traits.h.md) · [`object_comparer.h`](object_comparer.h.md) · [`object_cloner.h`](object_cloner.h.md) · [`object_destroyer.h`](object_destroyer.h.md) · [`object_loader.h`](object_loader.h.md) · [`object_saver.h`](object_saver.h.md)
**Used by** — [`editor_environment_manager.cpp`](../editors/xrWeatherEngine/editor_environment_manager.cpp.md) · [`editor_environment_suns_manager.cpp`](../editors/xrWeatherEngine/editor_environment_suns_manager.cpp.md) · [`property_collection.hpp`](../editors/xrWeatherEngine/property_collection.hpp.md) · [`pch.h`](../utils/mp_balancer/pch.h.md) · [`pch.h`](../utils/mp_configs_verifyer/pch.h.md) · [`problem_solver.h`](../xrAICore/Components/problem_solver.h.md) · [`patrol_point.cpp`](../xrAICore/Navigation/PatrolPath/patrol_point.cpp.md) · [`graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`graph_vertex.h`](../xrAICore/Navigation/graph_vertex.h.md) · [`EntityCondition.cpp`](../xrGame/EntityCondition.cpp.md) · [`GameTask.cpp`](../xrGame/GameTask.cpp.md) · [`UIGameAHunt.cpp`](../xrGame/UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../xrGame/UIGameCTA.cpp.md) · [`UIGameDM.cpp`](../xrGame/UIGameDM.cpp.md) · _and 18 more_
**Tier floor** — T4: it names a group. The group's own floor is T1.

## Purpose

The five operations of the object broker are written as five files because they are five
independent recursions, but no consumer wants four of the five: the entity layer needs all
of them and asks for them under one name. This file is that name, and it holds nothing
else.

The split between the five is not arbitrary — each is a separate traversal with its own
dispatch ladder and its own demands on a type — but the *packaging* is, and a rebuild is
free to publish them as one module.

## State

Stateless.

## Notes

The family's shared rules — the dispatch ladder, the ownership convention on text, the
container encoding — are stated once in [the directory README](README.md), and each of the
five twins assumes them.
