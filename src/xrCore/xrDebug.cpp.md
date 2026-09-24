# src/xrCore/xrDebug.cpp

> The failure path: assertion reporting, the fatal-error dialog with its three outcomes, signal and unhandled-exception installation, stack capture, and the crash-report snapshot.

**Needs** — [`xrDebug.h`](xrDebug.h.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md) · [`Debug/StackTrace.h`](Debug/StackTrace.h.md) · [`os_clipboard.h`](os_clipboard.h.md) · [`log.h`](log.h.md) · [`Threading/ScopeLock.hpp`](Threading/ScopeLock.hpp.md) · [`xrMemory.h`](xrMemory.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrsharedmem.h`](xrsharedmem.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`StackTrace.cpp`](Debug/StackTrace.cpp.md) · [`xrDebug.h`](xrDebug.h.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md)
**Tier floor** — T1: it runs *after* the process is already broken — inside a signal handler, inside an unhandled-exception filter, with the heap possibly corrupt — so it must not allocate, must not unwind, and must be able to walk the machine stack and read a saved register context.

## Purpose

Correctness in this engine is asserted continuously and checked nowhere else; there is no test suite. This file is what an assertion *does*. Its job is to turn a failed predicate into the most useful artifact obtainable from a dying process — a formatted report, a stack trace, the log flushed to disk, the whole thing on the clipboard — and then to offer the operator a choice about what happens next.

It is a separate file because it must be reachable from every module, including before the filesystem exists and after the window is gone.

## State

```text
RECORD DebugState                       # process-global, all statically initialized
  window_handler      : optional<WindowHandler>  # set once the window exists
  user_config_handler : optional<ConfigHandler>  # names the settings file to attach to a report
  previous_filter     : optional<ExceptionFilter># whatever owned the unhandled path before us
  out_of_memory_hook  : optional<function>
  bug_report_file     : text                     # an extra file to attach to a crash report
  error_after_dialog  : bool                     # a failure has already been shown; suppress the next
  show_error_message  : bool                     # false in the shipping build unless asked on the command line
  fail_lock           : mutex                    # serializes the whole failure path
```

Two invariants matter more than the rest:

- **The failure path is single-threaded by force.** The lock is taken before anything is printed and held until the dialog closes. Two threads failing at once would interleave their reports and race on the window; instead the second waits. A "are we already failing" probe on that lock exists so that code running *inside* the failure path can avoid recursing into it.
- **The window handler is notified on both edges of the dialog**, inside the lock, so a fullscreen window is dropped to windowed before a modal box appears and restored after. Doing this outside the lock lets two threads fight over the display mode.

## `gather_info`

**Contract** — Formats a failure into a caller-supplied buffer, writes it to the log, flushes the log, then appends a stack trace and copies the whole thing to the clipboard. Does not allocate on the heap (the buffer is the caller's, 4096 bytes at every call site). Returns nothing; the buffer is the product.

```text
FUNCTION gather_info(buffer, location, expression, description, arg1, arg2) -> void
  # Fixed field order. Tools and bug reports parse this, so it is stable.
  emit "FATAL ERROR"
  emit "Expression    : " + (expression, or "<no expression>")
  emit "Function      : " + location.function
  emit "File          : " + location.file
  emit "Line          : " + location.line

  IF description contains a line break THEN
    # A multi-line description is already formatted prose (a made-up
    # message from a caller); print it whole rather than as a field.
    emit description, then arg1, then arg2, each on its own line
  ELSE
    emit "Description   : " + description
    IF both args present THEN emit "Argument 0" and "Argument 1"
    ELSE IF one arg present THEN emit "Arguments     : " + arg1

  log(buffer)
  flush_log()                      # the log must survive what happens next

  IF a debugger is attached OR the operator asked for no stack traces THEN RETURN

  trace = capture_stack()
  # Skip the first two frames: they are this function and its caller inside
  # the failure machinery, never the code that actually failed.
  FOR EACH frame IN trace FROM index 2
    log(frame); append frame to buffer
  flush_log()
  copy buffer to the clipboard      # so a reporter can paste without finding the log
```

## `fail`

**Contract** — The single entry point for every assertion. Takes a per-call-site "ignore from now on" flag by reference, the source location, the failed expression's text, and up to three description strings. Blocks until the operator answers. Returns which of three outcomes was chosen. Never returns normally on the abort path.

```text
FUNCTION fail(ignore_always, location, expression, description, arg1, arg2) -> Outcome
  LOCK fail_lock DURING
    notify window handler: a dialog is about to appear
    error_after_dialog = true
    gather_info(buffer, location, expression, description, arg1, arg2)
    append the three-choice explanation to the buffer
    flush_log()

    IF running as a plugin inside another application THEN
      # No choice is offered: the host owns the process. Show and continue.
      show_message(buffer, simple)
      outcome = abort                       # deliberately NOT taken from the dialog
    ELSE
      outcome = show_message(buffer, three-way)   # default abort if suppressed
      IF outcome is try_again THEN
        error_after_dialog = false
        restore the display mode
      ELSE IF outcome is ignore THEN
        error_after_dialog = false
        ignore_always = true                 # this call site never fires again
        restore the display mode
      ELSE                                   # abort, or a dialog that failed to appear
        hand the buffer to the crash reporter as the user message
        IF a window exists and no debugger is attached THEN
          tell the window handler to tear the window down, because the
          break below will otherwise leave a frozen fullscreen surface
        BREAK INTO THE DEBUGGER               # with none attached this reaches the reporter
    notify window handler: the dialog is gone
  RETURN outcome
```

**Invariants** — The "ignore always" flag is **per call site**, held in storage local to the expanded assertion, not per predicate and not global. Choosing *continue* silences exactly the one assertion the operator was looking at. This is the design decision that makes assertions survivable in a modded game: a mod that trips one assertion every frame can still be played.

The three outcomes are worth naming as a contract, because they are the whole user model of a failing engine: **cancel** aborts and produces a report; **try again** resumes from the failed predicate once; **continue** resumes and never asks about this predicate again.

## `fatal`

**Contract** — Formats a message and enters the failure path with the ignore flag pre-set, so no dialog choice can resume execution. This is the path every unrecoverable data error takes — a missing configuration section, a bad chunk, a duplicate section.

## `do_exit`

**Contract** — A *clean* refusal to continue, as distinct from a failure: the message is shown, the window handler is told the process is going down, and the process is terminated immediately without unwinding. Used when the engine has decided it cannot run at all — no renderer, missing game data — where a stack trace would say nothing. Never returns.

**Notes** — Termination is immediate and hard rather than a normal return through the entry point, because the engine's global destruction order is not safe once the graphics device is gone.

## Signal and handler installation

**Contract** — Installed per thread on spawn and torn down on exit, so that a failure on a worker thread reports the same way as one on the main thread.

The set that is trapped, and what each is reported as:

| Trapped | Reported as |
|---|---|
| illegal instruction | "illegal instruction" |
| floating-point error | "floating point error" |
| segmentation fault | "segmentation fault" — **debug builds only** |
| abort | "application is aborting" |
| terminate request | "termination with exit code 3" |
| interrupt | released, not trapped |
| invalid runtime-library argument | "invalid parameter", with the offending expression |
| pure virtual call | "pure virtual function call" |
| allocation failure | the out-of-memory path below |
| unexpected termination | a report with no expression |

**Invariants** — The segmentation-fault trap is **absent from release builds** on purpose. Catching it there would convert a crash the operating system's reporter can analyse into a hand-rolled report from a process whose state is unknown; the crash reporter's own filter, installed separately, is better placed.

The whole installation is skipped entirely when the build is under a memory sanitizer, whose own handlers must win.

**Notes** — The assertion handler of the windowing library is redirected into the same path, so a failure inside it produces the engine's report rather than the library's dialog, and its three-way answer is mapped onto the same three outcomes.

## Out-of-memory path

**Contract** — Invoked when an allocation fails. If a caller installed a hook, it runs and nothing else happens. Otherwise the engine compacts what it can and reports what it found, then fails fatally with the requested size.

```text
FUNCTION on_out_of_memory(requested) -> void
  IF a hook is installed THEN hook(); ELSE
    compact_memory()                       # sweeps the string and blob interners
    log process heap usage
    log the string interner's savings and record count
    log the blob interner's savings
  fatal("Out of memory. Memory request: N K")
```

**Notes** — Reporting the interners' statistics here is the only place they are used. The number that matters when memory runs out is how much of the working set is *names and shared blobs* versus everything else, and that determines whether the answer is a smaller level or a real leak.

## Crash-report integration

**Contract** — On platforms with a crash reporter, a pre-error hook runs *before* the process dies and attaches context: the user's settings file (located through the virtual filesystem, falling back through two roots and then to a bare name), any extra file a module registered, and a memory snapshot. The snapshot's detail level is chosen from the command line — default is data segments plus indirectly-referenced memory plus thread data, with a full-memory mode available.

Once the filesystem is up, the log file's absolute path is registered as an attachment, and the report directory is pointed at the user's data root.

**Invariants** — The filesystem lookup for the settings file runs inside a structured-exception guard and falls back to a hard-coded name, because by the time this runs the filesystem's own state may be the thing that is broken.

**Notes** — Whether a dialog appears at all is decided differently per build: an interactive build always offers one; a shipping build shows nothing unless the operator asked for it on the command line, and a dedicated server or a run in silent mode saves a report instead of asking. A rebuild should keep the distinction — an automated server must never block on a modal box — and can drop the vendor reporter entirely, keeping only the report text and the stack trace.

## `log_stack_trace`

**Contract** — Captures and logs the current stack under a caller-supplied header. Used by tooling that wants to know how it got somewhere without failing.

## Unhandled-exception filter

**Contract** — Last resort. Captures a stack trace from the *saved register context* of the faulting thread rather than from the handler's own stack — the context is copied out and restored around the walk, because the walk consumes it. Logs the trace and the last operating-system error, flushes, optionally shows one box, hands a snapshot to the crash reporter, and then chains to whatever filter was installed before. Suppressed entirely if a failure dialog has already been shown, so one fault does not produce two reports.

**Notes** — Chaining rather than replacing is deliberate: the vendor reporter installs its own filter, and the engine wants to run first (to get the engine-shaped trace into the log) and then let it run (to produce the dump).
