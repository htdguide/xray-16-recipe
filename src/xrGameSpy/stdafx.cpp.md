# src/xrGameSpy/stdafx.cpp

> The translation unit that materialises the precompiled header.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T4. It does not survive a rebuild.

## Purpose

One line, whose only job is to give the compiler something to compile the precompiled
header into. It is an artefact of a C++ build system and has no counterpart in a rebuild.

## State

`Stateless.`
