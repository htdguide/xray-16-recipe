# src/xrScriptEngine/ScriptEngineScript.cpp

> The script engine's own exports: the logging, prefetch and bit-manipulation globals the shipped
> scripts call, and the profiler surface they drive from inside the game.

**Needs** — [`ScriptEngineScript.hpp`](ScriptEngineScript.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`script_profiler.hpp`](script_profiler.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`ScriptEngineScript.hpp`](ScriptEngineScript.hpp.md)

**Tier floor** — T3: a list of small procedures published to script. It sits in this module only
because the things it publishes are the module's own.

## Purpose

This is the script engine's leaf of the export graph. Two registration procedures live here: the
engine's own global functions, and the profiler's class and namespace. Both are declarative
registrations collected by [`ScriptExporter.cpp`](ScriptExporter.cpp.md); neither runs until
export time.

Everything it exports is *named exactly* as the shipped scripts call it. The names, arities and
the fact that a function exists at global scope rather than inside a table are all part of
criterion 10 and cannot be tidied.

## State

Stateless, apart from the script-visible stopwatch described below, whose state is per instance:

```text
RECORD ProfileTimer
  started_at  : timestamp
  accumulated : duration
  calls       : int (64-bit)
  nesting     : int
```

## Global functions

**Contract** — Published at global scope, callable from any script.

| Name | Contract |
|---|---|
| `log(text)` | Writes a message-kind line to the engine log and the script transcript; also mirrored to an attached external debugger. Suppressed entirely in a shipping build, which is why the shipped scripts' chatter costs nothing there. |
| `log1(text)` | Writes to the engine log unconditionally, in every build. |
| `error_log(text)` | Writes an error line, dumps the script call stack, mirrors to the debugger, and then **fails the process**. A script calling this is asserting that continuing is wrong. |
| `flush()` | Forces the engine log and the script transcript to disk. Developer builds only. |
| `flush1()` | Forces the engine log to disk in every build. |
| `print_stack()` | Dumps the current script call stack to the log without failing. |
| `prefetch(name)` | Loads a script namespace now, rather than leaving it to the first undefined-global read. The shipped scripts use it to move load cost out of gameplay. |
| `verify_if_thread_is_running()` | Fails unless a coroutine is currently being resumed. The shipped prelude calls it before yielding, because yielding outside a coroutine is otherwise a confusing failure far from its cause. |
| `bit_and`, `bit_or`, `bit_xor`, `bit_not` | Integer bit operations. They exist because the language version has none, and predate the JIT's bit library; the shipped scripts use these names. |
| `editor()` | Whether this VM belongs to the editor host rather than the game. |
| `user_name()` | The player profile name the engine was started with. |

**Notes** — The pairing of `log`/`log1` and `flush`/`flush1` is not redundancy: the unsuffixed
forms are the ones the shipped scripts call everywhere and are compiled out of a shipping build,
and the suffixed forms are the ones a modder uses when they want output from a release build.
A rebuild that keeps only one of each will either flood the release log or silence the modder.

## `profile_timer` — the script-visible stopwatch

**Contract** — A value type scripts construct, start, stop and read. Accumulates wall-clock
duration across start/stop pairs and counts the pairs. Reads out as microseconds. Supports copy
construction, addition (summing both duration and count), ordering by accumulated duration, and
conversion to text.

**Invariants**

- Start and stop nest: an inner start while already running only deepens a counter, and only the
  outermost stop closes the interval. Without this a recursive script function would record the
  same interval several times over.
- The call *count* increments on every start, including nested ones, so the count is calls and
  the duration is occupancy. Those are deliberately different measures.
- A stop with no matching start is ignored rather than failing; a clock that appears to move
  backwards contributes nothing.

## The `profiler` namespace

**Contract** — Published as a table, this is the in-game control surface for
[`script_profiler.cpp`](script_profiler.cpp.md): query whether profiling is active and in which
mode, start it in either mode (sampling optionally with an interval), stop, reset the collected
data, log a report with an optional entry limit, and save a report to a file. Three integer
constants name the modes.

**Notes** — The shipped prelude script inspects whether a JIT is present and installs its own
profiling hook when it is not; that interaction is why disabling the JIT and enabling the hook
profiler at the same time breaks the shipped scripts. See
[`script_engine.cpp`](script_engine.cpp.md).
