# src/xrScriptEngine/script_profiler.cpp

> Two ways to find out where script time goes: exact call accounting through the interpreter's
> hook, and statistical sampling through the just-in-time compiler.

**Needs** — [`script_profiler.hpp`](script_profiler.hpp.md) · [`script_profiler_portions.hpp`](script_profiler_portions.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)

**Used by** — [`script_profiler.hpp`](script_profiler.hpp.md)

**Tier floor** — T2: the hook mode runs on the interpreter's hottest callback and must key a
measurement by a synthesised string without allocating per event — the cost of the measurement
is the constraint.

## Purpose

Script is where a modded game spends its unexplained time, and the two questions modders ask —
"which function is slow" and "where is the program actually executing" — have different answers
and need different instruments. This provides both, switchable at run time from the console or
from script, and both report to the log and to a file.

## State

```text
RECORD ScriptProfiler
  engine            : ScriptEngine
  mode              : ENUM { none, hook, sampling }
  active            : bool
  hook_portions     : map<text, HookPortion>          # keyed by a synthesised callee;caller trace
  sampling_log      : list<SamplingPortion>           # one entry per sampling callback
  sampling_interval : int (milliseconds)              # clamped to 1 .. 1000, default 10
```

**Invariants**

- `active` and `mode` move together: stopping clears both and discards the collected data.
- A profiler may be constructed before the VM exists. It records the requested mode and attaches
  when the VM appears, which is what makes profiling from the command line work at all — the
  interesting time is startup.
- The hook is never *detached*. Only one hook may exist in the interpreter and the profiler
  cannot know whether it installed the one that is there, so stopping merely makes the callback
  return immediately. A rebuild with a multiplexed hook can do better.

## Construction

**Contract** — Reads three command-line switches — profile by default, profile in hook mode,
profile in sampling mode — and starts accordingly. A profiler is only created at all when the
host asks for one, and never for the editor host.

## `Start` / `StartHookMode` / `StartSamplingMode`

**Contract** — Start in the named mode, ignoring the request when already active. Both modes
clear their collected data on start. Hook mode attaches the interpreter hook; sampling mode
attaches the compiler's sampling profiler at the requested interval, clamped. When the VM does
not exist yet, both record the intent and return, leaving the attach to the reinitialisation
hook.

**Notes** — Sampling mode requires the just-in-time compiler; with an interpreter-only VM it
reports that and does nothing. The check for the compiler's presence is deliberately made
against the command line rather than by probing the VM, because probing pushes values onto the
stack and this may be called from inside an error path where the stack must not move.

## `Stop` / `Reset`

**Contract** — `Stop` detaches what can be detached, clears both stores, and returns to the
inactive state. `Reset` clears the stores without changing whether profiling is running.

## `OnLuaHookCall` — the hook-mode measurement

**Contract** — Called for every call, return and tail-return event while hook mode is active;
line events are discarded immediately. Attributes the event to a *call edge* — callee together
with its caller — rather than to a function, and starts or stops that edge's accumulator.

```text
FUNCTION on_hook_event(event)
  IF NOT active OR mode is not hook OR event is a line event
    RETURN
  callee = frame 1 ; caller = frame 2 ; innermost = frame 0
  IF callee or caller could not be read
    RETURN                                   # too close to the stack's base to attribute

  name = callee.name, falling back to innermost.name, then to "?"
  IF callee has no name and is defined at line 0
    name = "script-body"                     # a file's top level, which has no name
  caller_name = caller.name, falling back to "[C]"

  IF callee and caller are in the same source
    key = "<name>:<line>;<caller_name>@<caller source>:<caller line>"
  ELSE
    key = "<name>@<source>:<line>;<caller_name>@<caller source>:<caller line>"

  IF event is a call      THEN portion(key).start()
  IF event is any return  THEN portion(key).stop()
```

**Invariants** — Keying by the edge rather than the function is what makes the report actionable:
a utility called from forty places shows up forty times with forty different costs, and the
expensive caller is visible. The cost is that the key space is larger and that each event
synthesises a string.

**Notes** — The two key shapes exist so that a call within one file does not repeat the file name
twice. The separator characters are chosen so the key doubles as a folded-stack line.

## `LogHookReport` / `LogSamplingReport`

**Contract** — Print a bounded report to the engine log and flush it. The hook report is printed
twice, ordered by total duration and then by call count, each truncated to an entry limit
(default 128), with per-entry percentages of the total and an average per call. The sampling
report aggregates samples by function name across the whole log and prints the top entries by
sample count. Both report their totals.

**Notes** — Two orderings rather than one because the two questions differ: the duration ordering
finds the function to optimise, the count ordering finds the function to stop calling. Total
duration can legitimately be zero over a short capture, and the report special-cases that rather
than dividing by it.

## `SaveHookReport` / `SaveSamplingReport`

**Contract** — Write the full collected data, unbounded, to a file under the logs root named
after the application and the user. The hook report is one line per edge with its trace, count,
average and total. The sampling report is one line per callback: the VM state character, the
folded stack, and the sample count — the format a flame-graph tool consumes directly.

**Notes** — The sampling output's file extension and line format are chosen to be readable by
existing flame-graph tooling without conversion. That is the whole reason the folded form is
built at capture time rather than at report time.

## `OnReinit` / `OnDispose`

**Contract** — The two edges of the VM's life. `OnDispose` stops a sampling capture *before* the
VM handle becomes invalid, because the compiler's profiler can only be stopped through the state
that started it and would otherwise be impossible to detach. `OnReinit` re-attaches the active
mode to the new VM. Both are no-ops when nothing is active.

**Invariants** — This pairing is why the script engine notifies the profiler on both sides of
its reinitialisation rather than simply replacing the VM. Skipping the dispose leaves the
compiler's profiler pointing at freed memory.

## `AttachLuaHook`

**Contract** — Installs the engine's hook for call, return and line events, *unless a hook is
already installed*, in which case it succeeds only if the installed hook is the engine's own.
Reports whether a usable hook is in place.

**Notes** — This is the one place that enforces the interpreter's single-hook rule across the
profiler, the coroutine stack tracker and the external debugger. The three cannot coexist; the
debugger wins, then the profiler, then the tracker.

## Memory accounting

**Contract** — Reports the VM's live byte count, assembled from the collector's kilobyte count
and its byte remainder. Recorded with every sampling entry so that a capture shows allocation
growth alongside time.

**Notes** — This is the reading that lets the engine decide when to step the collector, and is
the reason
[Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
demands an inspectable byte count rather than just a collect call.
