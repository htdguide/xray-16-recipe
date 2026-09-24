# src/xrCore/cdecl_cast.hpp

> Turns a capture-free lambda into a plain function pointer with a specific calling convention, for handing to foreign code.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it exists solely to name a calling convention at a foreign-function boundary.

## Purpose

Several libraries the engine links against take a callback as a plain function pointer with a stated calling convention. A capture-free lambda can become such a pointer, but the conversion does not name the convention, so a callback written inline fails to match the library's expectation on the one architecture where conventions differ.

This helper performs that conversion explicitly, including for variadic callbacks.

**Notes** — Entirely a language artifact. The problem it solves — *passing an inline function to foreign code that dictates its calling convention* — appears at every foreign-function boundary in every tier, and each solves it its own way. A rebuild on a 64-bit-only target, where there is one convention, deletes this file and notices nothing.

## Exported units

- **Convert a callable to a function pointer** — deducing the signature from the callable's call operator, in a fixed-argument and a variadic form.
