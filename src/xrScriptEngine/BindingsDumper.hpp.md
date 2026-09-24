# src/xrScriptEngine/BindingsDumper.hpp

> Declares the exported-surface dumper.

**Needs** — [`BindingsDumper.cpp`](BindingsDumper.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`BindingsDumper.cpp`](BindingsDumper.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`BindingsDumper.cpp`](BindingsDumper.cpp.md).

## Exported units

- `BindingsDumper::Options` — the indent width, whether to omit inherited members, and whether
  to omit the receiver argument from method signatures. The script engine dumps with an indent
  of four, inherited members omitted and receivers stripped.
- `BindingsDumper::Dump` — write the whole script-visible surface of a VM to a stream.
