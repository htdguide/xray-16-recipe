# src/xrScriptEngine/script_profiler.hpp

> Declares the script profiler and the constants that bound it.

**Needs** — [`script_profiler.cpp`](script_profiler.cpp.md) · [`script_profiler_portions.hpp`](script_profiler_portions.hpp.md) · [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`console_commands.cpp`](../xrGame/console_commands.cpp.md) · [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md) · [`script_profiler.cpp`](script_profiler.cpp.md) · [`script_profiler_portions.hpp`](script_profiler_portions.hpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_profiler.cpp`](script_profiler.cpp.md), and fixes
the numbers a rebuild should carry over.

## Exported units

- `CScriptProfilerType` — none, hook or sampling. The numeric values are published to script as
  named constants, so they are part of the script surface.
- `CScriptProfiler` — lifecycle (`Start`, `Stop`, `Reset`), reporting (`LogReport`,
  `SaveReport`, and the per-mode forms), VM lifecycle hooks (`OnReinit`, `OnDispose`), and the
  hook callback. It registers its own script surface declaratively, like every other exporting
  class.

## Constants

| Constant | Value | Why |
|---|---|---|
| default mode | hook | Exact attribution is what a modder asks for first; sampling needs a compiler that may not be there. |
| report entry limit | 128 | Enough to reach past the obvious top of the profile, short enough that the log stays readable. |
| sampling interval | 10 ms | The compiler's own default. |
| maximum sampling interval | 1000 ms | Above one second a capture tells you nothing; the value is clamped rather than rejected. |

Three command-line switches select the startup mode: profile by default, profile in hook mode,
profile in sampling mode.
