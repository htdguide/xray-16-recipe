# src/xrCore/_flags.h

> A bit field with names for its operations, so that flag manipulation reads as intent instead of as masking.

**Needs** — [`xr_types.h`](xr_types.h.md)
**Used by** — [`xrCompressDifference.cpp`](../utils/xrCompress/xrCompressDifference.cpp.md) · [`Bone.hpp`](Animation/Bone.hpp.md) · [`LocatorAPI_defs.cpp`](LocatorAPI_defs.cpp.md) · [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md) · [`vector.h`](vector.h.md) · [`xrCore.h`](xrCore.h.md) · [`xr_ini.h`](xr_ini.h.md) · [`xr_shortcut.h`](xr_shortcut.h.md) · [`GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`NET_Shared.h`](../xrNetServer/NET_Shared.h.md) · [`gametype_chooser.h`](../xrServerEntities/gametype_chooser.h.md) · [`script_flags_script.cpp`](../xrServerEntities/script_flags_script.cpp.md) · [`Sound.h`](../xrSound/Sound.h.md)
**Tier floor** — T2: an integer and bit operations on it. The widths are load-bearing because several of these are serialized.

## Purpose

Almost every record in the engine that has boolean state keeps it in one of these rather than as separate booleans, for two reasons that both matter: a flag word costs one to eight bytes where a dozen booleans cost a dozen, and — the load-bearing one — **several of these words are written to disk and to the wire verbatim**. An entity's state flags, a mount point's flags, a visual's flags: all are one of these, and the bit positions are therefore part of the frozen formats.

## State

```text
RECORD Flags<W>          # W is 8, 16, 32 or 64 bits
  bits : int (W-bit)
```

The four widths are all used. Width is not an implementation detail: a serialized flag word is read back at its declared width, so narrowing or widening one changes the byte layout of whatever contains it.

## `Flags`

**Contract** — a value type, trivially copyable, with no invariants of its own: any bit pattern is legal. Every operation is total, allocates nothing, and mutating forms return the record so calls chain.

```text
get()                -> the whole word
zero()                  set every bit to 0
one()                   set every bit to 1
invert()                flip every bit
invert(mask)            flip the bits in mask        # note: mask form, not whole-word
assign(word | other)    replace wholesale
set(mask, on)           set the mask's bits if on, clear them if not
is(mask)             -> every bit of mask is set          # ALL
is_any(mask)         -> at least one bit of mask is set   # ANY
test(mask)           -> identical to is_any
or(mask)                set the mask's bits
and(mask)               clear every bit outside mask
equal(other)         -> whole words match
equal(other, mask)   -> the masked parts match
```

**Invariants** — the distinction between `is` and `is_any` is the one place this type is easy to misuse: `is` requires *all* the named bits, `is_any` requires *one*. A single-bit mask makes them identical, which is why the error hides. `test` is a third name for `is_any` and exists only for readability at call sites; a rebuild should keep one name for one meaning.

The two-argument `invert` takes the *whole* other record and replaces this one with its complement, while the one-argument form takes a *mask* and flips only those bits. Same name, different arity, different meaning — worth separating in a rebuild.

**Notes** — the whole file is what the brief calls incidental: a language with named bit sets deletes it. What survives is the requirement that the serialized flag words keep their exact widths and bit positions, because the shipped data and the shipped saves carry them.

Setting every bit uses the all-ones pattern of the signed interpretation, which is the same bit pattern in every width in use here; a rebuild should write it as an explicit all-ones mask rather than as a negative literal.
