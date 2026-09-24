# src/xrUICore/pch.cpp

> An empty translation unit that exists only to produce the precompiled header.

**Needs** — [`pch.hpp`](pch.hpp.md)
**Used by** — reached through its declarations in [`pch.hpp`](pch.hpp.md); callers name that, not this file.
**Tier floor** — T4: build plumbing.

## Purpose

Build-system scaffolding with no content: the toolchain needs one source file to compile in
order to emit the shared precompiled header that every other file in the module then
consumes. A rebuild in a language with a module system deletes this file and does not
replace it with anything.

## State

`Stateless.`
