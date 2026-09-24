# src/xrCore/Debug — making a crash report worth reading

Part of chapter 6, [`src/xrCore`](../README.md). Seven files: one that walks a stack, and
six that decode a number into a sentence.

## What this module is responsible for

When the engine dies, one thing decides whether the bug is fixable: what the log says. This
directory produces the two pieces of that.

**A call stack**, formatted as one line per frame with as much detail as the host can give —
module, address, demangled function with a byte offset, source file and line. It must work
from inside a crash handler, where almost nothing is safe, and it must work from a *frozen*
thread's register snapshot rather than from the handler's own frames.

**A name and a sentence for a numeric failure code.** The graphics and audio devices report
failures as packed 32-bit values that the host platform's own message service does not know,
so a failed device creation would otherwise log a bare hexadecimal number. Six of the seven
files are the adopted vendor table that fixes that.

The module owns neither the reporting policy nor the dialog a user sees — those belong to
[`../xrDebug.cpp`](../xrDebug.cpp.md), which is this directory's only real consumer inside
the engine.

## Where it sits

It rests on [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
for the symbol machinery and the register snapshot, and on
[`../Threading/Lock.hpp`](../Threading/Lock.hpp.md) for the serialization the symbol service
forces. [`../xrDebug.cpp`](../xrDebug.cpp.md) consumes both halves; chapter 9's network
client consumes the error decoder directly.

## The load-bearing ideas

**A faulting thread cannot walk its own stack.** By the time the handler runs, the frames of
interest are below the handler's. So there are two entry points: one that captures the
current registers, and one that takes a snapshot captured at the fault. Only the second is
useful in a crash report.

**A trace is always produced.** Three implementations sit behind one name, offering three
very different levels of detail, and the weakest returns a single line saying so. No caller
checks for failure, and none should have to.

**The symbol service is serialized process-wide and re-initialized per trace.** That is a
cost taken deliberately — symbols are not worth holding open for a facility used only when
something is already wrong — but it also means a second thread faulting behind the first
will block on the lock rather than report. A rebuild should use a bounded acquisition and
degrade to an address-only trace.

**The error decoder's value is one third of its size.** Two thirds of its table is the host
platform's general error space, which any modern platform decodes for itself; the useful
third is the graphics, audio and imaging codes it does not. The description path already
reflects this — it asks the platform first and only falls back to its own table — and a
rebuild should keep the structure and shrink the table.

**Most system codes appear in two forms.** Raw error number, and that number packed into a
result value. Both must map to the same name, because which one a caller sees depends on
which layer returned it.

**Four of these files exist only because one table had to compile twice.** The bodies were
split out so they could be included once per character width. A rebuild has one function per
job and deletes the scheme; the twins say which of the four files carries real content.

## The twins

| File | Role |
|---|---|
| [`StackTrace.h`](StackTrace.h.md) | Declares the two ways to ask for a call stack: from here, or from a thread frozen at a fault. |
| [`StackTrace.cpp`](StackTrace.cpp.md) | **The stack walker**, in three tiers of detail, with the architecture-specific register seeding and the process-wide lock. Substantive. |
| [`dxerr.h`](dxerr.h.md) | Declares the error decoder: name, description, and the trace that returns its argument. |
| [`dxerr.cpp`](dxerr.cpp.md) | The double-compilation scaffold — plus the two things in it that are real: codes composed by hand, and the dual raw/packed matching. |
| [`DXGetErrorString.inl`](DXGetErrorString.inl.md) | **The table**: ~3,200 codes to symbolic names, and what a rebuild should keep of it. |
| [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md) | Ask the platform first; fall back to ~300 hand-written sentences for the codes it cannot explain. |
| [`DXTrace.inl`](DXTrace.inl.md) | The interactive diagnostic: format to the debugger channel, optionally offer a breakpoint, return the code unchanged. |
