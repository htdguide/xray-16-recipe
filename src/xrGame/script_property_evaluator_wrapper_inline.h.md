# src/xrGame/script_property_evaluator_wrapper_inline.h

> The script evaluator adapter's constructor.

**Needs** — [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

One constructor, forwarding a subject and a name to the evaluator it specializes. Both
arguments default to nothing and to empty text, so a script may declare an evaluator
before it knows what it evaluates and bind the subject later through `setup`.

**Notes**

The name is a diagnostic label: it appears in the message when a script evaluator returns
something that is not a yes-or-no answer. That is the only thing it is used for, and it is
the reason a default-constructed evaluator's errors are harder to attribute.
