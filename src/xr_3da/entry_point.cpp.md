# src/xr_3da/entry_point.cpp

> The composition root — it names the renderer candidates and their preference order, decides whether the game layer is attached at all, hands both to the application, and wraps the whole run in the one failure handler that cannot be installed from inside.

**Needs** — [`stdafx.h`](stdafx.h.md) · [`xrEngine/x_ray.h`](../xrEngine/x_ray.h.md) · [`xrEngine/EngineAPI.h`](../xrEngine/EngineAPI.h.md) · [`xrGame/xrGame.h`](../xrGame/xrGame.h.md) · [`Include/xrRender/xrRender.h`](../Include/xrRender/xrRender.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: it installs a process-level fault filter for a condition (stack exhaustion) that the normal error path cannot survive, and it publishes two symbols that a graphics driver reads out of the executable image before any of this code runs. Everything else on the page is T3.

## Purpose

Eighty lines of source stand between the operating system and the engine, and every one of
them is a policy decision that belongs to *this product* rather than to any module. The
engine knows how to drive a renderer but not which renderers exist. The game layer knows
how to build a world but not whether this binary is supposed to have one. The process
entry has a signature the host chooses, not the engine.

So this file answers exactly four questions, and nothing else:

1. **Which renderers are on offer, and in what order of preference.**
2. **Is the game layer attached**, or is this a bare engine.
3. **What the command line is**, as one flat string, however the host delivered it.
4. **What happens when the process is beyond saving.**

A rebuilder should keep this file (or its equivalent) trivially small and readable for
exactly that reason: it is the one place where somebody assembling a variant of the product
— a dedicated server, a tools host, a renderer-only test harness — edits a list instead of
editing the engine.

## State

```text
RECORD RendererCandidates            # module-level, fixed at build time
  slots : list<optional<RendererModule>>   # exactly two slots
  # slot 0 is the preferred renderer AND the one a dedicated server must be able to use
  # slot 1 is the fallback
  # a slot may be empty when the platform has no such backend; the consumer skips empties
```

There is no other state. The two exported integers below are not state the program reads;
they are a message to somebody else's code.

## `renderer candidate list`

**Contract** — a fixed, ordered list of the graphics backends this executable is willing to
use. The application hands it to the engine, which asks each candidate for the display
modes it supports (a probe that touches hardware and can take real time), builds a
mode-name-to-backend map from the answers, and later selects one by the name stored in the
user's settings — falling back to any candidate that reports its requirements met. See
[Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device).

**Invariants**

- The list has a **fixed length of two**, independent of how many backends the platform
  actually has. An absent backend leaves an empty slot rather than shortening the list.
  The consumer must therefore tolerate empties; the fixed length exists so that the
  application's signature does not vary by platform.
- **Slot order is preference order.** Where a native-to-the-platform backend exists it
  occupies slot 0 and the portable one occupies slot 1.
- **Slot 0 carries an extra obligation**: a dedicated server refuses to start unless slot 0
  loads. A dedicated server renders nothing, but it still needs a renderer module attached,
  because the render module is where the material and shader vocabulary is registered and
  the server must resolve the same material names the clients do.

```text
FUNCTION build_renderer_candidates() -> list<optional<RendererModule>>
  IF platform has a native graphics API
    RETURN [ native_backend_module(), portable_backend_module() ]
  RETURN [ portable_backend_module(), none ]
```

**Notes** — in the original these are two backends, one per graphics API, and the list is a
compile-time constant because a backend that is not built is not linked. A rebuild that
loads backends dynamically should preserve the *ordering* and the *slot-0 obligation* and
may drop the fixed length.

## `entry_point`

**Contract** — takes the command line as one flat string, constructs the application, runs
it to completion and returns its exit code. It blocks for the entire lifetime of the
program. It is called exactly once, on the thread the host started.

```text
FUNCTION entry_point(command_line : text) -> int
  WITH profiler_shutdown_guard          # see Notes
    game <- IF command_line CONTAINS "-nogame" THEN none ELSE game_module()
    app  <- Application(command_line, game, build_renderer_candidates())
    RETURN app.run()
```

**Invariants**

- The game module is a **single process-wide object**, not one constructed per run. It is
  a pair of factory hooks plus a persistent-state constructor; the engine calls into it and
  never owns it.
- `-nogame` detaches the game layer entirely. The engine then brings up window, device,
  console, filesystem and sound, and has no world to simulate. This is how the renderer and
  the core are exercised without the game, and it is the reason the engine must treat an
  absent game module as a supported configuration rather than a failure.

**Notes**

*Why the command line is a flat string and not a parsed structure.* Every consumer in the
engine tests for its own switch by substring search on the same shared string — the
renderer selection, the dedicated-server flag, the input-capture flag, the filesystem
configuration override, the auto-start and auto-load commands. Nobody validates it, nobody
rejects an unknown switch, and the tail of a switch is read with a scan-until-space. A
rebuild is free to parse the line properly, but must then keep the *tolerance*: unknown
arguments are ignored, not errors, because the shipped launchers and mod loaders append
arguments this engine has never heard of.

*The profiler guard.* When the optional profiler is compiled in, the process must ask it to
flush and then wait for it to finish before exiting, or the last seconds of the capture are
lost. This is a scope-exit obligation placed before the application is constructed so it
outlives it, and it is the only reason this scope exists. See
[Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging).

## `high-performance GPU request`

**Contract** — two integers exported from the executable image under names two specific
graphics-driver vendors look for. A laptop with both an integrated and a discrete adapter
inspects the executable for these symbols at launch and binds the process to the discrete
adapter when either is non-zero. No code in this program reads them.

**Notes** — this is the clearest example on the page of *incidental C++ that is actually a
load-bearing decision in disguise*. The mechanism (an exported symbol with a vendor-chosen
name and a non-zero value) is entirely an artifact of how those two drivers were told to
detect the request in the 2010s. The decision — **this application always wants the
high-performance adapter, and never asks the user** — is load-bearing and must survive a
rebuild by whatever mechanism the target platform provides. If the rebuild's platform has
no such mechanism, the consequence is a silently halved frame rate on hybrid-graphics
machines, which is worth a note in its own right.

## `process entry`

**Contract** — the host's actual entry point. It normalizes whatever the host hands it into
one command-line string, calls the run above, and converts an unsurvivable failure into a
diagnosed abort rather than a silent death.

```text
FUNCTION process_entry(host_arguments) -> int
  command_line <- join(host_arguments EXCEPT the program name, separator = " ")
  TRY
    RETURN entry_point(command_line)
  ON stack exhaustion
    reset the guard page so the reporter itself has room to run
    FAIL WITH "stack overflow"
  ON any other unhandled failure
    FAIL WITH the failure's description
```

**Invariants**

- The **program's own name is dropped** and only the remaining arguments are joined, each
  followed by a single space. An empty argument list yields an empty string, not a null one
  — every consumer scans this string without a null check.
- Arguments are joined **without quoting or escaping**. A path containing a space is
  therefore indistinguishable from two arguments. This is a real limitation of the design
  and the reason switch values are read with a scan-until-space.

**Notes**

*Why stack exhaustion is handled here and nowhere else.* The engine installs a crash
reporter that captures a backtrace and writes a dump. That reporter needs stack to run, and
the one failure that guarantees there is none is stack exhaustion — so the reporter cannot
report it. This file therefore catches that one condition *outside* the reporter's reach,
restores the guard region so a few frames of stack become usable again, and then aborts
through the engine's fatal path with a fixed message. Deep recursion in the visibility
walk, in the script layer and in the path search all reach this in practice, so it is not
theoretical.

A rebuilder must ask: *does my runtime survive stack exhaustion at all, and can my crash
reporter run after it?* If the answer is no, this handler's job is simply "print a fixed
message and die", which every runtime can do — but it must be arranged before the reporter
is installed, not after.

*Why the two host paths differ in more than plumbing.* On the platform where the fault
filter exists, only stack exhaustion is intercepted and everything else is left to the
crash reporter, which produces a better diagnosis than a catch block would. On the other
path there is no such filter, so the entry wraps the run in a general failure catch and
reports whatever description it can extract — with one deliberate hole: a failure carrying
no description is swallowed and the process returns its initial failure code. That is a
gap, not a design, and a rebuild should close it.
