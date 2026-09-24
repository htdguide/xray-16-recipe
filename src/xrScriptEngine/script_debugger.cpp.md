# src/xrScriptEngine/script_debugger.cpp

> The engine's half of an external source-level debugger: breakpoints, stepping, watches and a
> thread list, exchanged with a separate editor process over a local message channel.

**Needs** — [`script_debugger.hpp`](script_debugger.hpp.md) · [`script_lua_helper.hpp`](script_lua_helper.hpp.md) · [`script_callStack.hpp`](script_callStack.hpp.md) · [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md) · [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md) · [`mslotutils.h`](mslotutils.h.md)

**Used by** — [`script_debugger.hpp`](script_debugger.hpp.md)

**Tier floor** — T2: a state machine over a message channel. The channel itself is T1 and lives
in [`mslotutils.h`](mslotutils.h.md); nothing here needs to be.

## Purpose

A modder debugging shipped script logic wants breakpoints and a variable pane, not log lines.
This is the engine side of that: it decides, on every line the interpreter executes, whether to
stop; when it stops, it hands the editor the call stack, the locals and the thread list, and
then *blocks the whole engine* until the editor says to continue.

It is a developer-build feature and is compiled out of a shipping build. It exists only on
Windows, because the channel it speaks over is a Windows facility.

## State

```text
RECORD ScriptDebugger
  mode          : ENUM { none, step_into, step_over, step_out, run_to_cursor, break_now, stop }
  depth         : int              # call depth since the current step began; signed
  breakpoints   : list<Breakpoint> # file basename plus line
  editor_present: bool             # re-tested on every send
  target_file, target_line         # for run-to-cursor
  threads       : DebuggerThreads  # the coroutine list, refreshed at each stop
  call_stack    : CallStack        # the frames shown in the editor
  lua           : LuaHelper        # everything that touches the VM
```

**Invariants**

- `editor_present` is not a connection; it is the answer to "does the editor's channel endpoint
  exist right now", re-asked before every send. The link is therefore stateless and survives the
  editor being started and stopped mid-session.
- When no editor is present the debugger is inert: every hook returns immediately and nothing
  blocks. This is what makes it safe to compile in.
- A stop point is a *hard block*: the engine's frame loop does not advance while the editor is
  deciding. Nothing tries to keep rendering.

## The stop decision

**Contract** — Called for every line the interpreter executes while an editor is attached.
Drains any pending editor messages first, so a break request typed while running takes effect
on the next line.

```text
FUNCTION on_line(file, line)
  drain editor messages
  IF mode is stop THEN RETURN
  IF   has_breakpoint(file, line)
    OR mode is break_now
    OR mode is step_into
    OR (mode is step_over AND depth <= 0)
    OR (mode is step_out  AND depth <  0)
    OR (mode is run_to_cursor AND file matches target AND line == target_line)
      enter_break(file, line)
      ask the editor for its current breakpoint set

FUNCTION on_call_or_return(is_call)
  IF mode is stop THEN RETURN
  depth = depth + (is_call ? 1 : -1)
```

**Invariants** — Step-over and step-out are the same mechanism at two thresholds: both reset the
depth counter to zero when the step begins, and both stop when the counter says execution has
returned to, or out of, the frame the step started in. That is the whole of stepping; there is
no frame identity involved, which is why a coroutine switch during a step confuses it.

**Notes** — The run-to-cursor condition compares the file *for inequality* in the original,
which makes the mode fire on the target line of the wrong file. It is unreachable in practice
because the editor's run-to-cursor message never sets the mode, so the feature does not work;
a rebuild should implement it properly rather than reproduce this.

## `DebugBreak` — what a stop shows

**Contract** — Refreshes the coroutine list and sends it, draws the current call stack and the
globals, then tells the editor to come to the front and blocks until the editor sends a
resume-class message.

**Invariants** — The stack-trace level is reset to the innermost frame on every stop, so the
editor's variable pane always opens on the frame that stopped.

## `ErrorBreak`

**Contract** — Stops at a script error, if an editor is attached, showing the same state a
breakpoint stop shows. This is the reason the script engine routes its error path through the
debugger when one exists: a failing script becomes a live debugging session instead of a log
entry.

## The message protocol

**Contract** — One message per action in each direction, each a length-prefixed sequence of
integers, floats and strings in a fixed buffer. The engine sends: connection opened and closed,
a log line, a jump to a file and line, activate yourself, clear and append stack-trace entries,
clear and append local and global variables, a thread list, and a watch-expression result. The
editor sends: go, break, the three step modes, run to cursor, stop debugging, select a
stack-trace level, a breakpoint set, select a thread, request a table's contents, and evaluate a
watch expression.

**Invariants** — Editor messages are classified into those that *resolve a modal stop* and those
that do not. Only the first class releases a blocked engine; a watch evaluation or a breakpoint
update arriving while stopped is handled and the engine stays stopped. Getting that distinction
wrong either hangs the engine or resumes it on a message that was not a resume.

**Notes** — Waiting for the editor is a poll with a ten-millisecond sleep. That is fine for a
stopped engine and is the simplest thing that works across a datagram channel with no readiness
signal.

## Breakpoint matching

**Contract** — A breakpoint matches when the line numbers are equal and the *base names* of the
files match, case-insensitively, with the extension and the directory discarded on both sides.

**Notes** — Matching on base name alone is a decision, not a shortcut: the engine knows a script
by its logical path inside the virtual filesystem, and the editor knows it by wherever the
modder has it checked out. Nothing can reconcile those two, so the shared part — the file's own
name — is what is compared. The consequence is that two scripts with the same name in different
directories share breakpoints. Script namespaces are flat in practice, so this does not bite.

## Watch evaluation

**Contract** — Prefixes the watched expression with a return and evaluates it in the stopped
frame's environment; see [`script_lua_helper.cpp`](script_lua_helper.cpp.md) for how the frame's
locals are made visible to it.
