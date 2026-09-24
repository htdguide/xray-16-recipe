# src/xrCore/Debug/DXTrace.inl

> Formats a failure — file, line, caller's note, code name and hexadecimal value — onto the debugger's output stream, and optionally offers to break into the debugger.

**Needs** — [`dxerr.cpp`](dxerr.cpp.md) · [`dxerr.h`](dxerr.h.md) · [`DXGetErrorString.inl`](DXGetErrorString.inl.md)
**Used by** — [`dxerr.cpp`](dxerr.cpp.md)
**Tier floor** — T1: it writes to a debugger channel and can trigger a breakpoint instruction, neither of which exists above the platform.

## Purpose

The body of the trace entry point, in its own file for the same double-compilation reason as its siblings. It is the interactive half of the error library: the part a developer sees while stepping, as opposed to the part that ends up in a log.

## `DXTrace` — the body

**Contract** — emit a diagnostic to the debugger's output channel, optionally raise a modal dialog, and **return the code unchanged** so the call wraps an expression. Writes nothing to the engine's own log. Blocks for as long as the dialog is up, which on a rendering thread means the frame stops. Formats into fixed stack buffers and never allocates.

```text
FUNCTION trace(file, line, code, note, offer_debugger) -> code
  IF file IS PRESENT
    emit "<file>(<line>): "
  IF note IS PRESENT AND NON-EMPTY            # bounded at 1024 characters
    emit note, then a space
  emit "hr=<symbolic name of code> (0x<code as eight hex digits>)"
  emit a newline

  IF offer_debugger AND the platform has modal dialogs
    compose: the file, the line, the code line, and — if a note was given —
             "Calling: <note>", then "Do you want to debug the application?"
    show it over the foreground window as a yes/no error dialog
    IF the answer is yes THEN break into the debugger
  RETURN code
```

**Invariants** — the return-the-code-unchanged rule is what lets a call site wrap a result in place, and it is the reason the release-build wrappers in [`dxerr.h`](dxerr.h.md) can collapse to the bare expression without changing the surrounding code. Any rebuild must keep it.

The caller's note is length-bounded before it is used, on the assumption it may not be terminated. That is defensive and cheap and worth keeping.

The file-and-line prefix is emitted as a separate write from the rest, so that interleaved output from other threads splits the line. Nothing depends on it staying whole.

**Notes** — the modal dialog is the file's one genuinely questionable behaviour, and a rebuild should think before reproducing it. It is raised over the foreground window, which during fullscreen rendering is the game itself, and answering it wrongly leaves the graphics device in a state the engine did not expect. Its value is real — it catches a failure at the exact call that produced it, with the stack intact — but the engine's own assertion path ([`../xrDebug.cpp`](../xrDebug.cpp.md)) already provides that, more carefully. The honest reading is that this is a vendor sample's ergonomics that came along with the table, and that the *table* is what the engine wanted.

Breaking into the debugger unconditionally when no debugger is attached terminates the process. The source does not check; neither does the engine's use of it, because the wrappers that raise the dialog are compiled out of release builds entirely.
