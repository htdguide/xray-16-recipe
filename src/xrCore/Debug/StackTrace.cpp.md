# src/xrCore/Debug/StackTrace.cpp

> Turns a thread's stack into human-readable lines — module, address, function with byte offset, source file and line — and degrades cleanly to whatever the platform can actually tell it.

**Needs** — [`StackTrace.h`](StackTrace.h.md) · [`../Threading/ScopeLock.hpp`](../Threading/ScopeLock.hpp.md) · [`../Threading/Lock.hpp`](../Threading/Lock.hpp.md) · [`../log.h`](../log.h.md) · [`../xrDebug.cpp`](../xrDebug.cpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`StackTrace.h`](StackTrace.h.md)
**Tier floor** — T1: it reads machine registers by name, walks frames with platform symbol machinery, and must work inside a crash handler where almost nothing else is safe.

## Purpose

Every assertion failure and every crash writes a stack trace to the log, and that trace is the primary evidence a bug report carries. The whole file is therefore judged by one criterion: **it must produce something useful under conditions where the process is already broken**, and must never itself be the reason a crash report is lost.

There are three implementations behind one name, chosen at build time, and the tiers of detail they produce are genuinely different. A rebuild should expect the same spread and design its log consumer for the worst of them.

## The three tiers

| Available | What a frame line carries |
|---|---|
| A full symbol service | module path, address, demangled function plus byte offset into it, source file and line plus byte offset into the line |
| A backtrace facility with name demangling | demangled function name, or the raw symbol if demangling fails |
| Neither | one line saying traces are unavailable on this platform |

**Invariants** — a trace is always returned, never an error and never nothing. An empty result means the symbol service could not initialize, and callers treat that as "no trace" without special-casing it.

## State

The rich path holds process-wide state that must be serialized:

```text
RECORD SymbolService
  library    : optional<handle>     # resolved once, lazily, by name
  entries    : the nine functions looked up from it
  initialized: bool
lock : Lock                          # one process-wide
```

**Invariants** — the symbol service is **not reentrant and not thread-safe**, so every trace takes one process-wide lock for its whole duration. That is a real hazard in a crash handler: if the faulting thread holds that lock, a second thread faulting behind it deadlocks instead of reporting. Nothing in the source guards against it, and a rebuild should use a try-lock with a timeout so that the second reporter degrades to an address-only trace rather than hanging.

The service is initialized and torn down **per trace**, not once per process. That costs a symbol-table load on every assertion failure — expensive, and deliberately so: holding symbols open across a session costs memory for a facility used only when something has gone wrong, and re-initializing is what makes a trace work after a module has been loaded or unloaded since the last one.

## Resolving the symbol machinery

**Contract** — find the already-loaded symbol library by name; do not load it if it is absent. Look up nine entry points by name and log each one that is missing. A missing library logs once and leaves every entry point absent, which makes every subsequent trace return empty.

**Notes** — the library is *looked up*, not loaded, which means traces work only where the host process already has it — typically because a debugger or the platform's own error reporting pulled it in. A rebuild that wants traces unconditionally should load it explicitly and accept the dependency.

Resolving by name at run time, rather than linking against the library, is what lets the engine build and run on a system where it is absent. That is the decision; the mechanism (a dynamic symbol lookup) is incidental.

## `BuildStackTrace(register_snapshot, max_frames)` — the rich path

**Contract** — walk the frames described by a register snapshot, formatting each. Takes the process-wide lock for the whole walk. Returns as many frames as the walk yields, capped. Never throws; an unresolvable frame still produces a line with the module and address.

```text
FUNCTION trace(registers, cap) -> list<text>
  LOCK symbol_lock DURING
    bring up the symbol service OR RETURN empty
    frame := seeded from the registers:
               program counter, stack pointer, frame pointer
               (and on architectures with one, the backing store pointer)
    WHILE a next frame is produced AND the program counter is non-zero
          AND fewer than `cap` frames collected
      line := ""
      IF the address resolves to a loaded module
        line := line + that module's path
      line := line + " at " + the address
      IF the address resolves to a function
        line := line + " " + name + "()"
        IF the address is not the function's first byte
          line := line + " + " + the byte offset
      IF the address resolves to a source location
        line := line + " in " + file + " line " + number
        IF the address is not the line's first byte
          line := line + " + " + the byte offset
      collect line
```

**Invariants** — the seeding is the architecture-specific part and the only place in the file that must be written once per target: which named register is the program counter, which is the stack pointer, which is the frame pointer. Every architecture the engine reaches has an entry and an unknown one is a build failure rather than a silent empty trace — which is the right choice, because a silently empty trace is indistinguishable from a working one that found nothing.

The walk terminates on a zero program counter as well as on the walker reporting failure, because a corrupted stack frequently yields a zero rather than an error.

Three symbol options are requested and each is a decision: defer symbol loading until a module is actually needed (so a trace through one module does not pay for fifty), load line-number information (without which the source location half of each line is absent), and undecorate names (without which every line carries a mangled symbol).

**Notes** — the byte-offset suffixes are what make the trace useful for an optimized build, where several source lines collapse into one address range: the offset distinguishes "at the start of this line" from "somewhere inside it".

## `BuildStackTrace(max_frames)` — the current thread

**Contract** — capture the current thread's registers and delegate. The capture must be a *capture of this thread*, not a query of it, because a thread cannot ask the platform for its own context without getting the answer for the query's frame.

**Notes** — the captured snapshot is marked as complete after the fact, which the platform requires before the walker will use the floating-point and extended register halves. It is the kind of detail that is invisible until the walk silently stops at the first frame.

## The backtrace path

**Contract** — where a symbol service is unavailable but a backtrace facility exists, collect return addresses into a scratch array sized by the frame cap, resolve each to a symbol string, and demangle it if a name-demangling facility is present. **Skips its own frame**, so the caller sees its own call site first. Returns raw symbol text where demangling fails.

```text
FUNCTION trace(cap) -> list<text>
  addresses := scratch array of `cap` entries
  n := collect return addresses into it
  symbols := resolve all n to text
  FOR i IN 1 .. n-1                       # 1, not 0: skip this function
    name := symbols[i]
    IF the address resolves to a named symbol
      demangled := demangle(that name)     # reusing one growable buffer
      IF that succeeded THEN name := demangled
    collect name
```

**Invariants** — the demangling buffer is reused across frames and reallocated by the demangler as needed, then released once at the end. That is the documented contract of the facility and getting it wrong leaks per frame, which matters because a crash handler may produce a thousand.

**Notes** — this path produces no module, no address and no source location — just names. It is markedly less useful than the rich path, and the gap is why bug reports from different platforms are not comparable.

The scratch array is allocated on the stack, sized by the caller's frame cap. Asking for 1024 frames therefore consumes several kilobytes of a stack that may be the one that just overflowed. A rebuild should bound the request or use a fixed buffer.

## The fallback

**Contract** — return a single line stating that traces are not implemented for this platform. It exists so that the caller's formatting is uniform and the log says *why* there is no trace rather than showing an empty section.
