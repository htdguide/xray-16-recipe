# src/Layers/xrAPI/stdafx.h

> The module's compilation prelude: it pulls in the project-wide common prelude and nothing else.

**Needs** — [`Common/Common.hpp`](../../Common/Common.hpp.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md) · [`xrAPI.cpp`](xrAPI.cpp.md)
**Tier floor** — T4: a build-system artifact with no runtime meaning.

## Purpose

A precompiled-header root, which is a build-time optimization and not a decision the system
makes. It is worth one observation and then nothing: the prelude it includes is the same
prelude every module in the project uses, and that prelude itself includes the service
environment's declaration. That is how `GEnv` becomes visible in every translation unit of
the engine without anyone including it deliberately — a fact that matters when reading any
other chapter, because a use of the environment is never announced by an import.

A rebuild has no equivalent of this file. The ambient visibility it grants is the thing to
*avoid* reproducing: see the rebuild note in [`README.md`](README.md).
