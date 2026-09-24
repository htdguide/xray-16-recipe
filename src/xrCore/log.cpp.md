# src/xrCore/log.cpp

> The log: an in-memory line buffer that exists before any file does, a file writer attached once the filesystem is up, and a callback that lets the console mirror everything.

**Needs** — [`log.h`](log.h.md) · [`xrCore.h`](xrCore.h.md) · [`FS.h`](FS.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`FileSystem.h`](FileSystem.h.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [`xrDebug.h`](xrDebug.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`log.h`](log.h.md)
**Tier floor** — T2: line buffering and file appending. Only the stack-allocated formatting buffers are T1 habits, and they exist to keep logging out of the heap — which matters because the failure path logs.

## Purpose

The log is the engine's only diagnostic channel and the only artifact a bug report contains. Two constraints shape it: it must work **before** the filesystem is mounted (bring-up failures are the ones worth reading), and it must work **while the process is dying** (the failure path flushes it before showing anything). Both are why every line is kept in memory as well as written.

## State

```text
RECORD LogState                  # process-global
  lines          : list<text>    # every line ever logged, in order, kept forever
  writer         : optional<FileWriter>
  file_name      : text (path)
  suppressed     : bool          # logging to a file was disabled at bring-up
  callback       : optional<(text) -> void>
  callback_armed : bool          # the callback may be muted without unsetting it
  force_flush    : bool          # flush after every line
  lock           : mutex
```

**Invariants**

- **The line list is never trimmed.** It is reserved for a thousand lines at bring-up and grows from there; a full session's log is resident. This is what lets the console display scrollback the engine never wrote to disk, and what lets the file writer, when it finally opens, emit everything that happened before it existed.
- **Every line in the list is a single line.** A logged string containing line breaks is split before it enters; nothing downstream has to handle embedded breaks.
- **An empty line is stored as a single space.** A genuinely empty line is indistinguishable from a formatting accident in a text file, and several readers of the log treat a blank line as a section break.
- **The lock covers the list, the callback and the writer together**, so a line reaches all three in the same order on every thread.

## `log`

**Contract** — Takes one string, splits it on line breaks, and emits each piece. Allocates its split buffer on the stack, sized from the input — so logging never allocates and can be called from the failure path and from a signal handler. Thread-safe.

```text
FUNCTION log(text) -> void
  buffer = stack space for length(text) + 1
  j = 0
  FOR EACH ch IN text
    IF ch is a line break THEN
      terminate buffer at j
      IF buffer is empty THEN buffer = " "     # never store a truly empty line
      emit(buffer)
      j = 0
    ELSE
      buffer[j] = ch; j = j + 1
  terminate buffer at j
  emit(buffer)                                 # the tail, even when it is empty

FUNCTION emit(line) -> void
  LOCK log DURING
    write line to the platform's debug channel
    append line to lines
    IF callback_armed AND callback exists THEN callback(line)
    IF writer exists THEN
      write line followed by carriage-return/line-feed
      IF force_flush THEN flush()
```

**Notes** — The line terminator written to the file is carriage-return plus line-feed on every platform. That is a frozen expectation of the log-reading tools and of bug reports pasted into issue trackers, not a platform concern.

## `message`

**Contract** — Formats into a 2048-byte stack buffer and logs the result. Truncates silently at the buffer size. This is the entry point almost every caller uses.

**Notes** — 2048 is the limit on a single log message. Callers that exceed it — dumps of large structures — split themselves.

## Typed appenders

**Contract** — A family that takes a message and one value and logs "message value": a string, each signed and unsigned integer width, a real, a 3-vector rendered as `(x,y,z)`, and a 4x4 matrix rendered as four rows on four lines. Each sizes a stack buffer from the message's length plus a fixed allowance for the value (11 characters for a 32-bit integer, 64 for anything wider or floating-point) and formats into it.

**Notes** — Appending a null string logs the message alone rather than the word "null". The fixed allowances are generous by design; the comment in the source that a float's text form "should be no more than 40 characters, but we count with slight overhead" is the whole reasoning.

## `create_log`

**Contract** — Opens the log file and drains the in-memory backlog into it. Called once the filesystem can resolve paths. Takes a flag that suppresses file logging entirely (used by tools that must not write).

```text
FUNCTION create_log(suppress) -> void
  reserve space for a thousand lines
  suppressed = suppress

  unique = command line asked for unique logs
  IF unique THEN
    name = application_name + "_" + user_name + "_" + local_timestamp + ".log"
  ELSE
    name = application_name + "_" + user_name + ".log"
  IF the logs root resolves THEN name = that root + name

  IF suppressed THEN RETURN

  IF NOT unique THEN
    # Exactly one previous run is preserved, under a .bkp extension.
    # Without this, the log of the run that crashed is destroyed by the
    # run the user starts to reproduce the crash.
    rename name -> same name with extension ".bkp", overwriting

  writer = open(name)
  IF writer exists THEN
    FOR EACH line IN lines           # everything logged before the file existed
      write line
    flush()
    publish writer under the lock
  IF the command line asked for forced flushing THEN force_flush = true
```

**Invariants** — The backlog is drained *before* the writer is published under the lock, so a concurrent logger cannot interleave into the middle of the backlog.

**Notes** — The timestamp in the unique-log name is local time formatted day-month-year and hour-minute-second with separators that are legal in filenames on every target. The single backup file is the smallest thing that solves the "the crash log was overwritten by the retry" problem, and it is worth keeping.

## `flush` / `close_log`

**Contract** — Flush pushes the writer's buffer to disk and does nothing when logging is suppressed; it is called on every failure before anything is displayed. Close flushes, releases the writer, and clears the line list.

**Invariants** — The line list is cleared on close, which means anything logged after teardown is lost from memory as well as from the file. Teardown is the last thing the process does.

## `set_callback`

**Contract** — Installs a function that receives every subsequent line, returning whatever was installed before, so callbacks can be chained or restored. Taken under the lock. The console installs one; so does the crash reporter.

**Invariants** — The callback runs **inside the log lock**, on the logging thread. It must not log, must not block, and must not allocate if the logging thread might be the failure path.

## `log_file_name`

**Contract** — Reports the resolved log path, for the crash reporter to attach.
