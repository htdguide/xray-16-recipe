# src/Common/PlatformWindows.inl

> The Windows fill-in — the platform whose conventions the rest of the engine already assumes, so it mostly trims the system headers down and supplies empty conversions.

**Needs** — [`Platform.hpp`](Platform.hpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Platform.hpp`](Platform.hpp.md)
**Tier floor** — T1: it exists to reach operating-system services and to fix the widths of types that cross that boundary.

## Purpose

The engine was written against Windows and the game data was authored there, so this
fill-in is the shortest of the four: the conventions it would have to translate are already
the engine's own conventions. Its real work is limiting what the system headers bring in,
and stating the three things the other fill-ins have to emulate.

## State

Stateless.

## Filesystem name folding

**Contract** — the virtual filesystem asks the platform two questions when it normalizes a path:
*should the lookup key be lowercased?* and *should the stored name be lowercased?* On a
case-insensitive filesystem the answer is: fold the lookup key, leave the stored name
alone.

**Invariants** — exactly one of the two folds is active on any platform, and the pair is
inverted on the POSIX fill-ins. Getting this backwards does not fail loudly — it makes a
subset of the shipped assets silently unfindable.

## Path separator conversion

**Contract** — converting between the engine's canonical separator and the host's is a no-op
here, because they are the same character. Both directions exist anyway so that callers do
not branch on platform.

## Local time decomposition

**Contract** — decompose a wall-clock instant into local calendar fields, writing into a
caller-supplied record and returning it, or nothing on failure. The caller supplies the
storage because the call must be safe from several threads at once.

## Error text

**Contract** — render a system error number as human-readable text into a caller-supplied
buffer, bounded by that buffer's size.

## Process identity type and signed size type

**Contract** — names for "an operating-system process identifier" and "a signed count of bytes",
both at the platform's natural width. These exist as named types because they appear in
engine interfaces that the POSIX fill-ins must satisfy identically.

## Notes

Most of the file is a list of exclusions that tell the system headers not to declare
subsystems the engine never touches — menus, clipboard, cryptography, help, metafiles and
about forty more. That is a compile-time cost control, not a design decision, and it
vanishes in any rebuild. One exclusion is worth carrying: the one that suppresses the
system's `min`/`max` definitions, because they are macros that would otherwise capture
every ordinary use of those words in engine source. Any rebuild that shims a platform
should note that macro-shaped shims leak into unrelated code, which is the same lesson the
compiler fill-in draws about renamed standard functions.

The file also requests a minimum operating-system version — the baseline is Windows 7 —
which fixes which system services may be called without a runtime check.
