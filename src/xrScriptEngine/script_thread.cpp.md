# src/xrScriptEngine/script_thread.cpp

> One script coroutine: created from a namespace or from console text, resumed once per turn,
> and retired the moment it stops yielding.

**Needs** — [`script_thread.hpp`](script_thread.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)

**Used by** — [`script_thread.hpp`](script_thread.hpp.md)

**Tier floor** — T2: it creates and resumes a coroutine of a guest interpreter and must keep
that coroutine's stack balanced across a failure.

## Purpose

The unit of long-running script. A coroutine here always runs the same shape of code: a call
expression compiled as its own chunk, which calls either `<namespace>.main()` or, for the
console path, a function the engine synthesised from the text the player typed.

Two forms exist because there are two ways a script enters the world: declared in the level's
configuration, and typed at the console. They are the same machinery, differing only in how the
body gets into the VM.

## State

```text
RECORD ScriptThread
  script_name : text       # the namespace, or the literal "console command"
  coroutine   : handle     # its own interpreter state, registered against the owning engine
  reference   : int        # registry reference keeping the coroutine alive
  active      : bool       # false once it has finished or failed; never becomes true again
```

**Invariants**

- `active` is monotonic: a coroutine that stops yielding is finished forever. The process
  destroys it rather than trying to restart it.
- The coroutine handle is registered against the owning engine before anything can call back
  from it, and unregistered on destruction.
- Nothing may be left on the coroutine's stack when it yields. A shipped script that passes a
  value to yield is a defect the engine asserts on, because the value would silently accumulate.

## Construction

**Contract** — Prepares a coroutine and the call that will run inside it. Reports itself
inactive rather than failing when anything goes wrong; the process then discards it.

```text
FUNCTION create(engine, name, from_text, reload)
  IF NOT from_text
    script_name = name
    engine.process_file(name, reload)              # load the namespace if it is not loaded
    body = "<name>.main()"
  ELSE
    script_name = "console command"
    wrap `name` as the body of a uniquely-named global function and run that definition now
    IF the definition failed to compile or to run
      report and RETURN inactive
    body = "<that function>()"

  coroutine = new_coroutine(engine.vm)
  IF developer build
    install the line/call/return hook on the coroutine
    # the external debugger's hook wins when one is attached; only one hook may exist
  compile `body` as a chunk named "@_thread_main" on the coroutine's stack
  active = true
```

**Notes**

- The body is compiled *on the coroutine's own stack* and left there; resuming the coroutine is
  what runs it. This is why the coroutine's stack is expected to be empty on every yield: the
  only thing that should ever be on it is the chunk being run.
- The console form synthesises a named global function rather than compiling the text directly
  into the coroutine, because the text is a *statement sequence*, and the coroutine needs a
  *call*. Naming the function globally is the cheap way to bridge the two. The name is long and
  unlikely enough to collide that the original does not defend against a script defining it.
- The load-time reload flag exists so the console can re-run a script the player has just
  edited: the file is re-read even though the namespace already exists.

## `update` — resume once

**Contract** — Resumes the coroutine and reports whether it is still alive. Exactly one of three
things happens: it yields (alive), it returns (finished), or it fails (finished, with the error
reported and the script call stack dumped in developer builds). Sets the engine's current-thread
marker for the duration and clears it on every exit path, including a thrown failure.

```text
FUNCTION update() -> bool
  IF NOT active THEN FAIL WITH "cannot resume a dead coroutine"
  engine.current_thread = self
  TRY
    status = resume(coroutine)
    IF status is a failure
      report(status); dump the coroutine's call stack; active = false
    ELSE IF status is not "yielded"
      active = false                    # ran to completion
    ELSE
      ASSERT the coroutine's stack is empty    # nothing may be passed to yield
  ON any failure
    active = false
  engine.current_thread = none
  RETURN active
```

**Notes** — The current-thread marker is what makes "am I inside a coroutine" answerable from
script, which the shipped prelude checks before yielding. Clearing it on the failure path as
well as the success path is the invariant that matters; a leaked marker makes every later yield
appear legal.

## Destruction

**Contract** — Hands itself back to the engine, which releases the registry reference that kept
the coroutine alive and unregisters its handle. The release is guarded against failure and
ignores one, because it happens during teardown where nothing can be done about it.

**Notes** — The original carries a switch recording that the binding layer of its vintage had
defects around coroutines, under which the reference release is skipped entirely. The switch is
on: the release does *not* happen in the shipped configuration, so a finished coroutine's
registry slot leaks for the life of the VM. A rebuild should release it; the leak is bounded by
the number of scripts ever started, which is why it was tolerable.
