# src/Common/Compiler.inl

> Fills in the handful of code-generation controls the engine needs but the language does not standardize — inlining, alignment, symbol visibility, the debugger trap, and whether exceptions exist.

**Needs** — [`Platform.hpp`](Platform.hpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Platform.hpp`](Platform.hpp.md) · [`PlatformLinux.inl`](PlatformLinux.inl.md)
**Tier floor** — T1: it controls inlining, alignment and symbol export, and emits a raw trap instruction per architecture. Nothing above T1 exposes these.

## Purpose

The engine asks its compiler for six things no language standard guarantees, and every
compiler spells them differently. This file is the single translation table, selected by
the compiler classification made in [`Platform.hpp`](Platform.hpp.md). It is a fill-in, not
a module: it defines names, and something else decides when it is read.

## State

Stateless.

## Inlining controls

**Contract** — three strengths must be expressible and are all used: *suggest* inlining,
*insist* on it, and *forbid* it.

Insisting is load-bearing in the math layer, where a vector operation that is not inlined
costs more in call overhead than it does in arithmetic, and in the guard code around
per-frame hot loops. Forbidding is load-bearing in crash handling and in a few functions
whose stack frame must be identifiable in a backtrace — if they are folded into their
caller the backtrace loses the frame that names the failure.

**Notes** — the engine carries three short aliases for these (a legacy of the original
codebase) that the source itself marks for removal. They are three names for the same three
strengths; a rebuild needs the three strengths, not the six names.

## Alignment control

**Contract** — a declaration must be able to demand that a type or object begins on a given byte
boundary. The math layer's 4-wide float types demand 16, and buffers handed to the graphics
and audio devices demand whatever those devices ask for — see
[§2 Tier](../../SYSTEM-REQUIREMENTS.md#2-tier).

## Debugger trap

**Contract** — halt into an attached debugger at this exact point, or, with no debugger attached,
terminate the process in a way a crash handler can catch. It must be a *statement*, usable
inside an assertion's failure branch.

```text
FUNCTION debug_break()
  # one of, chosen at build time by architecture:
  #   x86 / x86-64   : the one-byte breakpoint instruction
  #   arm32          : the breakpoint encoding for the current instruction set
  #                    (thumb and arm encode it differently)
  #   arm64          : the breakpoint instruction
  #   anything else  : raise the trap signal if the target has one,
  #                    otherwise execute an illegal instruction
```

**Notes** — the fallbacks are ordered by how much they preserve. A real breakpoint instruction
resumes cleanly if the developer continues; the trap signal is catchable and carries the
right meaning; the illegal-instruction fallback is last because it reports the wrong cause.
A rebuild whose runtime offers a breakpoint intrinsic should use it and keep the ordering
in mind only if it has to fall back.

## Symbol visibility

**Contract** — a declaration must be able to say "this crosses a dynamic-library boundary",
in the two directions: exported by the library that defines it, imported by the one that
uses it.

**Invariants** — the renderer and the game are loaded as dynamic libraries in the non-static
build configuration, so the interfaces between them must be marked. In the static build the
marking must still compile and must cost nothing.

**Notes** — one compiler needs the two directions spelled differently and the other does not;
that asymmetry is incidental. The decision that survives is *which* boundaries are dynamic,
which is a property of the module layout, not of this file.

## Interface layout hint

**Contract** — a pure-interface type may be marked as never instantiated directly, letting the
compiler omit the table-pointer setup its constructor would otherwise emit. Purely a size
and startup-cost optimization, and a no-op on the compiler that does not offer it.

## Unreachability assertion

**Contract** — asserts to the optimizer that a condition holds, licensing it to delete the code
that would run if it did not. Used where a range has already been checked by a layer above
and the check would otherwise be repeated in an inner loop.

**Notes** — this is a promise, not a check: if the condition is false the program's behaviour is
undefined rather than wrong-but-defined. It is only safe where the invariant is established
by construction, and every use site should say by what.

## Exception availability

**Contract** — the build may be made with exceptions disabled, so the engine must be able to
spell "this function does not throw" in a way that compiles either way.

**Invariants** — the script binding layer is the part that cares: it must be compilable both to
exceptions and to an error callback — see
[Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer).

## Debug-configuration normalization

**Contract** — several spellings of "this is a debug build" arrive from different build systems;
they are collapsed into one name before any other header reads it. The `mixed`
configuration counts as a debug build for this purpose: it keeps assertions.

## Renamed standard functions

**Contract** — a small set of C string and process functions exist under different names on
different systems (case-insensitive compare, case conversion in place, stack allocation,
integer to text, file deletion). Each is given one engine-side name and mapped per
compiler.

**Notes** — the file explains its own choice: the mapping is done by introducing a *new* name
rather than by redefining the standard one, because a redefinition that lands before the
system header is read would rename the system's own declaration too. That is a real trap
and the reasoning transfers to any rebuild that shims a standard name — shim under a new
name, never over the old one.

The set itself is a symptom rather than a decision. Almost every entry is a case-insensitive
string operation, which exists because the game data references files with inconsistent
case; see [§4 Platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions),
where the folding rule is fixed at ASCII-only.
