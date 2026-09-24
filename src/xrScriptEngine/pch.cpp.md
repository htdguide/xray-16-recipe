# src/xrScriptEngine/pch.cpp

> Nothing; it exists to give the compilation prelude a translation unit to be built from.

**Needs** — [`pch.hpp`](pch.hpp.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T4: build scaffolding.

## Purpose

An artifact of how one toolchain materialises a precompiled prelude. It contributes no code and
survives a rebuild as nothing at all.

## State

Stateless.
