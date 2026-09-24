# src/xrCore/xr_shortcut.h

> A key binding: a scancode plus modifier bits, packed into 16 bits so a binding table is an array of integers.

**Needs** — [`_flags.h`](_flags.h.md)
**Used by** — [`xrCore.h`](xrCore.h.md) · [`PropertiesListTypes.h`](../xrServerEntities/PropertiesListTypes.h.md)
**Tier floor** — T1: it is a byte-packed union of two 8-bit fields with a 16-bit view of the pair, and that 16-bit value is what a binding is compared and stored as.

## Purpose

Key bindings are stored by **scancode**, not by character — the platform assumption says so, because bindings must survive a keyboard-layout change. A binding is that scancode plus which modifiers must be held, and this type is the pair.

## State

```text
RECORD Shortcut                 # byte-packed, exactly 2 bytes
  key       : int (8-bit)       # the scancode
  modifiers : int (8-bit)       # a bit set, see below
  # The two fields are also addressable as one 16-bit value, which is what
  # comparison and storage use -- a binding table is an array of 16-bit
  # integers and lookup is an integer compare.
```

The modifier bits:

| Bit | Meaning |
|---|---|
| 0x20 | shift |
| 0x40 | control |
| 0x80 | alt |

**Invariants** — The three modifier bits occupy the **upper** part of the byte, leaving the lower five bits unused. Nothing uses them, and a rebuild is free to pick any encoding *unless* it must read a settings file the original wrote — the packed 16-bit value is what appears there.

## Exported units

- **Construct** — from a scancode plus three booleans, or empty.
- **Similarity** — equal modifiers and equal scancode. This is the comparison the input layer runs per key event.

## Notes

Modifier state is exact, not permissive: a binding on control-plus-a does not fire when control-shift-a is pressed. That is what makes a modifier a discriminator rather than a filter, and it is why the three bits are compared as a unit rather than masked.
