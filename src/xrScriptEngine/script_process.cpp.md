# src/xrScriptEngine/script_process.cpp

> A named group of script coroutines, stepped one per frame, with the collector stepped alongside.

**Needs** — [`script_process.hpp`](script_process.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`script_thread.hpp`](script_thread.hpp.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)

**Used by** — [`script_process.hpp`](script_process.hpp.md)

**Tier floor** — T2: a round-robin over coroutines with an explicit per-frame budget, plus a
manual collector step. The budget is what keeps it off T3.

## Purpose

The engine runs script coroutines, not script callbacks, for the long-lived AI and mission logic
that shipped with the games: each such script has a `main` that loops and yields. A *process* is
the owner of a set of those coroutines. There are two, created and destroyed with the thing they
belong to — one for the loaded level, one for the multiplayer match — so that a level unload
kills exactly the coroutines that belonged to that level.

## State

```text
RECORD ScriptProcess
  name        : text                    # "level" or "game"; appears in the debugger's thread list
  threads     : list<ScriptThread>      # running coroutines, in creation order
  pending     : list<PendingScript>     # requested but not yet started
  cursor      : int (wraps)             # which thread gets this frame

RECORD PendingScript
  name        : text        # a namespace name, or a literal chunk of script text
  is_text     : bool        # true when `name` is source to run, not a script to load
  reload      : bool        # force the script file to be re-read even if already loaded
```

**Invariants**

- A frame starts exactly one pending script and resumes exactly one running coroutine. Both
  bounds are deliberate; see below.
- A coroutine that reports itself finished is removed from the list in the same frame, and the
  cursor is stepped back one so that the coroutine that slid into the vacated slot is not
  skipped.

## `update` — one frame of script

**Contract** — Starts every pending script, resumes exactly one coroutine, drains whatever the
scripts wrote to the interpreter's error stream into the engine log, and in developer builds
steps the collector. Does nothing when no coroutine is alive. Called once per frame by whichever
module owns the process.

```text
FUNCTION update()
  start_pending()
  IF threads IS empty THEN RETURN

  clear the captured error-stream buffer
  cursor = cursor + 1
  chosen = cursor MOD count(threads)
  IF NOT threads[chosen].resume()
    destroy threads[chosen]
    remove it from the list
    cursor = cursor - 1          # the next frame must not skip the thread that moved down
  IF the captured buffer is non-empty
    terminate and flush it, then log it as script output

  IF developer build
    step the collector by one increment      # ignore any failure
```

**Notes**

- *One coroutine per frame* is the whole cost model. A level can have a few dozen script
  coroutines, most of them yielding immediately; resuming them all every frame would put the
  script layer on the frame budget's critical path for no benefit, because these scripts are
  written to poll at human timescales. The consequence a rebuild inherits: **a script's
  effective tick rate is the frame rate divided by the number of live coroutines in its
  process**, and the shipped scripts are tuned to that.
- *Capturing the error stream.* Anything the scripts write to the interpreter's standard error —
  including the interpreter's own diagnostics — is redirected into a fixed buffer at
  initialisation and drained here, so that script output reaches the engine log rather than a
  console nobody reads. The buffer is fixed-size and shared, which is why it is cleared at the
  top of every update rather than after use; a rebuild with a per-VM sink does not need either.
- *Stepping the collector here* rather than at a collection point of the interpreter's choosing
  is how script allocation is kept off the frame's tail. It is guarded to developer builds in
  the original, which is almost certainly an oversight rather than a decision: the shipping
  build leaves the collector to run on its own schedule, which is the configuration most likely
  to produce a visible hitch. A rebuild should step it in every build.

## `run_scripts`

**Contract** — Drains the pending list, creating one coroutine per entry through the engine (so
that the new coroutine is registered before it can call back). A coroutine that fails to
construct, or that constructs inactive, is discarded rather than kept. Entries are taken from
the back of the list, so a burst of requests starts in reverse order of arrival — which nothing
depends on.

## `add_script`

**Contract** — Queues a script to be started on the next update. Takes the namespace name (or,
for the console path, literal script text), whether it is text, and whether to force a reload of
the file. Does not touch the VM.

## Construction and destruction

**Contract** — A process is constructed from a name and a *comma-or-space separated list* of
script namespace names, each of which becomes a pending entry. That list comes from the game's
configuration, so which scripts a level runs is data, not code. Destruction destroys every
coroutine it owns, which is what makes a level unload clean.
