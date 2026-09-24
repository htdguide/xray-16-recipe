# src/xrAICore/pch.cpp

> The single translation unit that exists so the shared prelude can be compiled once.

**Needs** — [`pch.hpp`](pch.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: a build artefact with no content.

## Purpose

Exists only to give the build system something to compile in order to produce the precompiled
prelude. It contains nothing.

A rebuild deletes this file and has no replacement for it.

## Stateless.
