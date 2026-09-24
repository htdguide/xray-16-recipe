# src/xrCore/Animation/Envelope.hpp

> An animation curve: a keyed, interpolated scalar over time, with a shape per key and a behaviour outside the keyed range.

**Needs** — [`Envelope.cpp`](Envelope.cpp.md) · [`interp.cpp`](interp.cpp.md) · [`../FS.h`](../FS.h.md)
**Used by** — [`BoneEditor.cpp`](BoneEditor.cpp.md) · [`Envelope.cpp`](Envelope.cpp.md) · [`Motion.cpp`](Motion.cpp.md) · [`Motion.hpp`](Motion.hpp.md) · [`interp.cpp`](interp.cpp.md) · [`PostProcess.cpp`](../PostProcess/PostProcess.cpp.md) · [`PostProcess.hpp`](../PostProcess/PostProcess.hpp.md)
**Tier floor** — T1: the key record is written and read as a packed layout with quantized fields, and version 1 reads the shape from a 32-bit field that version 2 stores in 8.

## Purpose

Every authored animation channel in the engine — a bone's translation on one axis, a weather parameter over the day, a post-process intensity — is one of these. It is a direct adoption of a 1990s animation package's envelope model, including its key shapes and its extrapolation behaviours, and the vocabulary is that package's. Adopting it wholesale is why the engine can read curves the original authoring tools exported.

This header defines the curve's data and the key record's on-disk layout, so it carries that substance. The curve *operations* are in [`Envelope.cpp`](Envelope.cpp.md) and the evaluation is in [`interp.cpp`](interp.cpp.md).

## State

```text
ENUM KeyShape                      # how the segment ENDING at this key is drawn
  tcb      = 0    # tension/continuity/bias spline — the default
  hermite  = 1    # explicit incoming and outgoing tangents
  bezier   = 2    # tangents expressed as handle lengths
  linear   = 3
  stepped  = 4    # hold the previous value until this key
  bezier2  = 5    # two-dimensional handles: time AND value

ENUM EndBehaviour                  # what happens outside the keyed range
  reset     = 0   # zero
  constant  = 1   # hold the nearest key's value
  repeat    = 2   # wrap the time into the range
  oscillate = 3   # wrap, mirroring on odd cycles
  offset    = 4   # wrap, adding one range-worth of value per cycle
  linear    = 5   # extrapolate along the end tangent

RECORD Key                         # packed, no alignment padding
  shape      : int (8-bit)
  value      : real
  time       : real (seconds)
  tension    : real
  continuity : real
  bias       : real
  param      : real[4]             # handle data; meaning depends on shape

RECORD Envelope
  behaviour : EndBehaviour[2]      # [0] before the first key, [1] after the last
  keys      : list<Key>            # ordered by increasing time
```

**Invariants** — keys are kept sorted by time; every operation that inserts one preserves that, and evaluation depends on it. Two keys at the same time (within an epsilon) are treated as one — inserting at an occupied time overwrites the value rather than adding a key.

The meaning of the four handle parameters is shape-dependent and is *not* uniform:

```text
# hermite / bezier : param[0] = incoming tangent, param[1] = outgoing tangent
# bezier2          : param[0] = incoming handle TIME offset
#                    param[1] = incoming handle VALUE offset
#                    param[2] = outgoing handle TIME offset
#                    param[3] = outgoing handle VALUE offset
# tcb / linear / stepped : unused
```

## Key serialization — two versions

**Frozen.** The difference between them is the reason a version tag exists at all.

```text
FUNCTION save_key(w, k)
  write_real(w, k.value)
  write_real(w, k.time)
  write_u8  (w, k.shape)
  IF k.shape IS NOT stepped THEN               # a stepped key has no curve data
    write_quantized_16(w, k.tension,    -32, +32)
    write_quantized_16(w, k.continuity, -32, +32)
    write_quantized_16(w, k.bias,       -32, +32)
    write_quantized_16(w, k.param[0..3], -32, +32)      # four of them

FUNCTION load_key_v1(r, k)                     # the older form: no quantization
  k.value := read_real(r)
  k.time  := read_real(r)
  k.shape := low 8 bits of read_u32(r)         # shape occupied 32 bits
  k.tension := read_real(r); k.continuity := read_real(r); k.bias := read_real(r)
  read k.param[0..3] AS ONE RAW BLOCK of four reals
  # NOTE: no stepped special case. A v1 stepped key still carries all the data.

FUNCTION load_key_v2(r, k)                     # matches save_key exactly
  ...
```

**Invariants** — the quantization range is **-32 to +32** for all seven curve parameters, giving a resolution of about one part in a thousand over that range. Tension, continuity and bias are conventionally in -1 to +1; the generous range exists because the handle parameters are not bounded that way. A value outside the range is a write-time error.

The **stepped** shape is the one that skips its curve data, and the skip is keyed on the literal value 4 rather than on the enumeration name in the source. A rebuilder reordering the enumeration breaks the format.

## `CEnvelope`

The operations are contracted in [`Envelope.cpp`](Envelope.cpp.md): finding and inserting and deleting keys, scaling a time range, measuring the curve's length, rotating every value by a constant, and reducing a constant curve to two keys. Evaluation is contracted in [`interp.cpp`](interp.cpp.md).

Serialization of the envelope itself, as opposed to its keys:

```text
FUNCTION save_envelope(w, e)
  write_u8 (w, e.behaviour[0])
  write_u8 (w, e.behaviour[1])
  write_u16(w, count(e.keys))        # caps a curve at 65535 keys
  FOR EACH k IN e.keys DO save_key(w, k)

FUNCTION load_envelope_v1(r, e)
  read e.behaviour AS ONE RAW BLOCK of two 32-BIT integers
  count := read_u32(r)               # 32-bit count in v1
  ...

FUNCTION load_envelope_v2(r, e)
  e.behaviour[0] := read_u8(r); e.behaviour[1] := read_u8(r)
  count := read_u16(r)
  ...
```

**Notes** — the width shrink from 32 to 8 bits for the behaviours and from 32 to 16 for the count is the whole of the version-2 change, together with the key quantization. The motive was size: a skinned model has six curves per bone and a hundred bones, so four bytes saved per field is measured in megabytes across a game's animation set.

## Text form

**Contract** — an envelope can also be read from a plain-text form emitted by the authoring package: a brace-delimited block, a key count, then one line per key beginning with the word `Key` and carrying nine space-separated reals, then a line naming the two behaviours. The nine reals are value, time, shape, and then six more whose meaning depends on the shape — for a tension-continuity-bias key they are tension, continuity and bias followed by two tangents; for a two-dimensional bezier key they are the four handle offsets.

**Invariants** — exactly nine numbers must parse, and exactly two behaviours; anything else is a malformed file and is fatal. There is no writer for this form: it is read-only, an import path from the authoring tool.

**Notes** — the shape dispatch in the text reader is written as two independent tests rather than a selection, with the result that a tension-continuity-bias key also takes the `else` branch of the second test and picks up two tangent values. That is intentional and matches the exporter.
