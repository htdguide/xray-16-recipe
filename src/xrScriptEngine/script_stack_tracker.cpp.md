# src/xrScriptEngine/script_stack_tracker.cpp

> A shadow copy of a coroutine's call stack, maintained by the debug hook so that the stack can
> still be printed after the coroutine has failed.

**Needs** — [`script_stack_tracker.hpp`](script_stack_tracker.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md)

**Used by** — [`script_stack_tracker.hpp`](script_stack_tracker.hpp.md)

**Tier floor** — T2: a fixed-capacity stack of frame descriptions updated from the interpreter's
hottest callback. The capacity and the cost of the hook are the reasons it is not written more
abstractly.

## Purpose

The interpreter can be asked for its call stack — but only while the stack still exists. Some of
the failures worth reporting unwind it first, and a coroutine that has already stopped has no
stack to inspect. This keeps a running copy, updated on every call and return, so the engine can
print where a coroutine *was* rather than where it is.

It exists only in developer builds, and it is the reason the per-line hook is installed there:
the hook's cost is the price of post-mortem stack traces.

## State

```text
RECORD StackTracker
  frames : list<FrameInfo>   # fixed capacity of 256 entries, never reallocated
  depth  : int               # frames[0 .. depth-1] are live; depth <= 256
```

**Invariants**

- Capacity is fixed at 256. Script recursion deeper than that stops being recorded rather than
  growing the store or failing; the frames that *are* recorded stay correct. A fixed ceiling is
  right here because the tracker runs on the interpreter's hottest callback and must not
  allocate.
- `depth` never goes below zero. Returns can outnumber calls when the hook is installed partway
  through execution, which is exactly what happens for a coroutine created while the engine is
  already running.

## `script_hook`

**Contract** — Called by the engine's hook for every call, return, tail return, line and count
event on this coroutine. Maintains the shadow stack. Does not allocate and does not touch the
value stack.

```text
FUNCTION script_hook(event)
  SWITCH event
    CALL ->
      IF depth >= capacity THEN RETURN          # silently stop recording beyond the ceiling
      read frame 0 into frames[depth]
      IF depth > 0 THEN refresh frames[depth-1] from frame 1
      depth = depth + 1
    RETURN, TAIL RETURN ->
      IF depth > 0 THEN depth = depth - 1
    LINE, COUNT ->
      frames[depth].current_line = the interpreter's current line
```

**Notes** — Refreshing the *caller's* frame on every call is what keeps the caller's current
line accurate: the caller's line advanced between its own last line event and this call, and
without the refresh the printed trace points at where the caller was several statements ago.
That is the one non-obvious line in this file and it is the reason the traces are usable.

## `print_stack`

**Contract** — Logs the shadow stack outermost-frame-first as error-kind lines, formatting
native frames by name and script frames as source, line and function name. **Resets the depth to
zero afterwards**, so the tracker is ready for the next coroutine run and a second print does
not repeat a stale trace.
