# src/editors/xrWeatherEditor/engine_include.hpp

> Pulls in the engine-facing contract with the compilation mode the boundary requires.

**Needs** — [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md)
**Tier floor** — T1: it exists solely to control how code is compiled across a two-language module boundary.

## Purpose

It includes [`engine.hpp`](../../Include/editor/engine.hpp.md) with the surrounding translation unit switched to unmanaged compilation, then switches back.

## State

`Stateless.`

## Notes

The whole file is the problem it solves: **the editor library is compiled in a mode where types default to the managed runtime's rules, and the engine contract is a set of types the unmanaged host must be able to lay out.** Compiling the contract's declarations under the wrong rules would produce a type the host cannot call.

A rebuild in one language deletes this file. A rebuild that keeps the two-language split meets the same requirement and solves it however its toolchain allows — but must solve it, and must solve it for every header that crosses.

It is worth noting that the file is redundant with the same bracket written inline in [`entry_point.cpp`](entry_point.cpp.md) and [`ide_impl.hpp`](ide_impl.hpp.md). Whether this header or the inline brackets came first is not recoverable; a rebuild needs one, not both.
