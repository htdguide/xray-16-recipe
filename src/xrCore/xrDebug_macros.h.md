# src/xrCore/xrDebug_macros.h

> The assertion vocabulary: which checks survive into the shipping build, which vanish, and what each one carries into the report.

**Needs** — [`xrDebug.h`](xrDebug.h.md) · [`xrDebug.cpp`](xrDebug.cpp.md)
**Used by** — [`FixedVector.h`](FixedVector.h.md) · [`buffer_vector.h`](buffer_vector.h.md) · [`string_concatenations.h`](string_concatenations.h.md) · [`xrDebug.cpp`](xrDebug.cpp.md) · [`xrDebug.h`](xrDebug.h.md) · [`xrPool.h`](xrPool.h.md)
**Tier floor** — T2: the mechanism is textual substitution, but the *policy* — which class of check is compiled out — is a language-independent decision that a rebuild makes with whatever its tier offers.

## Purpose

There is no test suite. Invariants are asserted inline, continuously, everywhere, and this file decides which of those assertions a player's machine actually runs. That split is the file's entire content and the only thing worth carrying across: **two classes of check exist, and they behave differently in a shipping build.**

The substitution mechanism itself is incidental. What a rebuild needs is a way to capture the failing expression's *text* and its source location at the call site, and a way to give each site its own persistent "ignore from now on" flag.

## State

Stateless as a module. Each expanded check owns one bit of storage — a per-call-site "ignore always" flag, initially false, set when the operator chooses *continue*, and never reset. That flag is why the same failing predicate can be silenced individually rather than by category.

## The two classes

**Contract** — Every check names an expression, optionally a description and up to two argument strings, and on failure enters the failure path with all of it plus the source location.

| Class | In a debug build | In a shipping build |
|---|---|---|
| **Requirement** | evaluated, reports on failure | **evaluated, reports on failure** |
| **Verification** | evaluated, reports on failure | **compiled out; the expression is not evaluated** |

**Invariants** — Because a verification's expression is not evaluated in a shipping build, **a verification may not have side effects.** This is the rule the file exists to state and the one most easily broken in a rebuild: moving a check from one class to the other changes whether its expression runs.

```text
FUNCTION check(expression, description, arg1, arg2) -> void
  # One static flag per call site. Once set, the site is dead.
  IF ignore_always_for_this_site THEN RETURN
  IF expression holds THEN RETURN
  fail(ignore_always_for_this_site, here(), text_of(expression),
       description, arg1, arg2)
```

## Variants and what each adds

- **Requirement, 0–2 extra strings** — the load-bearing class. Data validation, format checks, contract checks on public entry points.
- **Verification, 0–2 extra strings** — the expensive class. Range checks inside inner loops, structural checks on containers, pre/post-conditions that restate what the caller already guaranteed.
- **Result check** — takes a platform result code rather than a boolean, and renders it as text in the report. Used at graphics- and system-call boundaries.
- **Graphics-call check** — evaluates the call, then asks the graphics device for its error state. In a shipping build the call still runs and the error query is dropped. This is the one place where compiling a check out changes the *timing* of everything around it, because the query is a pipeline stall.
- **Check-or-exit** — evaluates always, and on failure takes the clean-refusal path rather than the assertion path: no stack trace, no three-way choice, just a message and termination. For "this installation is not usable" conditions.
- **Fatal** — no expression; fails unconditionally with a formatted message.
- **Cured requirement** — evaluates always; on failure reports (debug only) and then runs a caller-supplied recovery statement (debug *and* shipping). This is the only construct that both reports and continues deterministically, and it is how the engine survives bad data in places where dying would be worse than degrading. The recovery runs in the shipping build even though the report does not.
- **No-default** — marks an unreachable branch. In a debug build it fails; in a shipping build it is an optimizer hint that the branch cannot happen, which is a real behavioural difference: reaching it in a shipping build is undefined, not caught.
- **Throwing check** — where exceptions are enabled, gathers the report into a buffer and throws it; where they are not, it degrades to a verification and therefore **vanishes in a shipping build**. A rebuild should pick one of the two and not carry the degradation.

## Location capture

**Contract** — Every check captures the file, line and enclosing function name of the *call site*, not of the failure machinery. A rebuild's equivalent must do the same or every report names this file.

**Notes** — The expression's source text is part of the report and is frequently the only useful field, since descriptions are often absent. Any rebuild mechanism that cannot recover the expression text loses most of the diagnostic value.
