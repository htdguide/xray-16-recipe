# src/Common/Common.hpp

> The single prelude every translation unit in the engine opens with.

**Needs** — [`Config.hpp`](Config.hpp.md) · [`Platform.hpp`](Platform.hpp.md) · [`FSMacros.hpp`](FSMacros.hpp.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md)
**Used by** — [`stdafx.h`](../Layers/xrAPI/stdafx.h.md) · [`pch.h`](../utils/mp_balancer/pch.h.md) · [`pch.h`](../utils/mp_configs_verifyer/pch.h.md) · [`StdAfx.h`](../utils/xrCompress/StdAfx.h.md) · [`pch.hpp`](../utils/xrMiscMath/pch.hpp.md) · [`pch.hpp`](../xrAICore/pch.hpp.md) · [`stdafx.h`](../xrCDB/stdafx.h.md) · [`stdafx.h`](../xrCore/stdafx.h.md) · [`stdafx.h`](../xrEngine/stdafx.h.md) · [`stdafx.h`](../xrGameSpy/stdafx.h.md) · [`stdafx.h`](../xrMaterialSystem/stdafx.h.md) · [`stdafx.h`](../xrNetServer/stdafx.h.md) · [`stdafx.h`](../xrParticles/stdafx.h.md) · [`pch.hpp`](../xrScriptEngine/pch.hpp.md) · _and 1 more_
**Tier floor** — T4: nothing here computes. It is a build-time aggregation that a language with a module system expresses as one import, or deletes entirely.

## Purpose

Four unrelated things have to be in scope before any engine source compiles: the
compile-time feature switches, the platform and compiler detection with its per-platform
fill-ins, the logical filesystem root names, and the global service-locator struct through
which every module reaches its peers. This file is the one place that names all four, so
that no source file has to remember the order they must be pulled in.

The order is the only decision in the file: feature switches first (they gate what the
platform layer defines), platform second (it defines the type names and inline helpers the
rest rely on), then the vocabulary headers.

## State

Stateless.

## Notes

A rebuild in a language with real modules has no equivalent of this file and should not
invent one: the four things it gathers are four independent modules, and each consumer
should name the ones it actually uses.

The file also runs a build-time check that the project's vendored dependencies were
actually fetched, on one compiler only. That is a build-system concern, not a program
concern, and belongs in the build description rather than in source.
