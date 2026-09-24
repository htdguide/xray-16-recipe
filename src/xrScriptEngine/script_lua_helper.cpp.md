# src/xrScriptEngine/script_lua_helper.cpp

> Everything the debugger does that touches the VM: the hooks, the stack trace, the variable
> panes, and evaluating an expression as if it were standing in a stopped frame.

**Needs** — [`script_lua_helper.hpp`](script_lua_helper.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)

**Used by** — [`script_lua_helper.hpp`](script_lua_helper.hpp.md)

**Tier floor** — T2: it manipulates the interpreter's value stack and global table directly, and
its correctness is a stack-balance argument.

## Purpose

The debugger proper is a state machine over a message channel; this is its hands. Separating
them means the protocol can be reasoned about without the stack discipline and vice versa — and
the stack discipline here is the delicate part, because all of it runs *inside* a paused script.

## State

```text
# process-wide, because the interpreter's hooks are plain functions with no context
instance : optional<LuaHelper>     # the one helper; set on construction, cleared on destruction
state    : handle                  # the VM or coroutine currently being inspected
frame    : optional<FrameInfo>     # the frame the last hook reported
```

**Invariants** — Every entry point returns immediately when there is no instance. The hooks are
installed into the interpreter and can fire after the debugger has been destroyed, which is the
only reason the guard exists.

## `PrepareLua` / `UnPrepareLua`

**Contract** — `PrepareLua` installs the debugger's error handler as a global function, installs
the line/call/return hook, pushes the handler onto the stack and reports its index. The caller
passes that index to its protected call, so that a failure inside the call reaches the handler
*with the failing stack still intact*. `UnPrepareLua` removes the handler from the stack at the
index it was given.

**Invariants** — The handler must be pushed *below* the call's arguments; the reported index is
absolute, not relative, for exactly that reason. This bracket is the only reason the script
engine's file loading takes an explicit handler index.

## `PrepareLuaBind`

**Contract** — Replaces the binding layer's protected-call handler and, when the layer is
compiled without exceptions, its error callback, with the debugger's own. From then on a failure
inside any bound call stops in the editor rather than logging and dying.

## The hooks

**Contract** — One hook receives every event and dispatches: call, return and tail return adjust
the debugger's step depth; a line event offers the debugger a stop point. Both first ask the
interpreter for the frame's source and line. A frame whose source is not a file is ignored —
only script *files* have breakpoints.

**Invariants** — The hook asserts that it leaves the value stack exactly as it found it. That
assertion is the entire correctness argument for running arbitrary debugger work on every line
of a live game.

## The error handler

**Contract** — Runs as the message handler of a failed protected call, with the failing stack
still present. Builds a traceback by walking the stack and concatenating one line per frame,
writes it to the editor, stops at the failing line, and then **fails the process**.

```text
FUNCTION build_traceback()
  FOR level = 1 upward WHILE a frame exists
    IF level > 12 AND not yet elided
      IF more than 10 further frames exist
        emit an ellipsis and skip forward to the last 10
      mark elided
      CONTINUE
    emit "<level>- <source>:<line>: " followed by a description of the frame:
      a named global, local, field or method  -> "in function `<name>'"
      the file's top level                    -> "in main chunk"
      a native function                       -> its source
      anything else                           -> "in function <source:line>"
```

**Notes** — The elision keeps the first twelve and the last ten frames of a deep trace and drops
the middle, because a runaway recursion otherwise produces thousands of identical lines and the
useful ends are the two that get lost. The numbers are conventional rather than derived.

The handler is fatal after reporting: with an editor attached the stop has already happened and
the modder has seen the state, so there is nothing to continue into. This is a *different* policy
from the no-debugger path in [`script_engine.cpp`](script_engine.cpp.md), where the operator may
choose to ignore the error and carry on — which is the same load-bearing divergence between
build and tooling configurations that runs through this whole chapter.

## `DrawStackTrace`

**Contract** — Clears the editor's stack pane and appends one entry per script frame, outermost
to innermost, each carrying a display description, the source file and the current line. Frames
that are not files are skipped.

## `DrawLocalVariables`

**Contract** — Clears the variable pane and appends every local of the frame the editor has
selected. A local holding a table is appended as a header and then expanded one level, with
members named by their dotted path; the expansion does not recurse further, because a deep
object graph would take longer to send than the modder will wait.

## `Eval` — evaluating a watch in a stopped frame

**Contract** — Compiles the expression, runs it, and renders its result as a type-and-value
string. A compile or run failure is reported as its message with the source-position prefix
stripped. The value stack is restored to its entry depth in every case.

The interesting part is that the expression must see the *stopped frame's locals*:

```text
FUNCTION evaluate(expression) -> text
  saved = new empty table
  FOR EACH local `name` IN the selected frame
    saved[name] = the current global `name`      # remember what we are about to hide
    global `name` = the local's value            # make the local visible as a global
  result = compile and run `expression`
  FOR EACH name, value IN saved
    global name = value                          # put every hidden global back
  RETURN result
```

**Invariants** — Every global the shadowing touched is restored, including the ones that were
absent (restored to absent). Failing to restore leaves the *running game* with a global replaced
by a dead local, which would be a debugger that corrupts the program it is debugging.

**Notes** — Shadowing the globals is a blunt instrument, chosen because the language version has
no way to evaluate a chunk against a frame's locals directly. A rebuild whose interpreter can
set a chunk's environment to a view over the frame — or can evaluate in the frame — should do
that instead; this technique is not re-entrant and is not safe if anything else runs while the
shadowing is in place, which is true here only because the engine is stopped.

## `Describe` / `DrawVariable`

**Contract** — Render a value for display: its type name, and for numbers, strings and booleans
a truncated rendering of the value. Tables render as a header with no value. Other types render
as type alone. Truncation limits are small and deliberate — the variable pane shows an
identification, not a value dump.

**Notes** — Userdata is *not* expanded. The commented-out attempt in the original shows the
intent; the dump of a bound object's contents ended up in the engine's own stack dump instead
(see [`script_engine.cpp`](script_engine.cpp.md)), which is the better place for it because it
works with no editor attached.

## `DrawGlobalVariables`

**Contract** — Clears the editor's global pane and then iterates the globals table without
sending anything. The traversal is intact and the emit is commented out in the original, so the
global pane is always empty. A rebuild should either send them or drop the message; the current
state is neither.
