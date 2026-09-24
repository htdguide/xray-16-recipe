# src/xrCore/PostProcess/PostProcess.cpp

> Loads an authored screen effect from its frozen binary form and evaluates its eleven curves at a time to produce one parameter block.

**Needs** — [`PostProcess.hpp`](PostProcess.hpp.md) · [`PPInfo.hpp`](PPInfo.hpp.md) · [`Animation/Envelope.hpp`](../Animation/Envelope.hpp.md) · [`FS.h`](../FS.h.md) · [`LocatorAPI.h`](../LocatorAPI.h.md)
**Used by** — [`PostProcess.hpp`](PostProcess.hpp.md)
**Tier floor** — T1: a versioned binary format read field by field in a fixed order.

## Purpose

Two jobs. First, the **file format**: eleven parameter tracks written back to back with no index, no lengths and no names, so position *is* identity — a reader that skips or reorders one field mis-decodes everything after it. Second, the **playback object**: a loaded effect that can be advanced to a time and asked for the resulting parameter block.

## State

```text
RECORD Animator
  params        : CPostProcessParam[11]   # indexed by pp_params; order is the file order
  block         : SPPInfo                 # the parameters write directly into this
  name          : text
  factor        : real                    # current intensity, clamped to [0.001, 1]
  desired_factor: real
  factor_speed  : real
  stopping      : bool
  cyclic        : bool
  start_time    : real                    # -1 means "not started"
  length        : real                    # invariant: the max of every track's length
```

**Invariant** — each of the eleven parameter objects holds a pointer into `block`; evaluating a parameter writes its float straight into the block. That aliasing is the whole design: there is no gather step. A rebuild that separates the curves from the block must add one, and must keep the eleven-to-eleven mapping exactly, because the mapping is also the file order.

**Invariant** — `length` is recomputed after every load and equals the longest of the eleven tracks. A shorter track simply holds its last value past its end, which is the keyframed-curve evaluator's behaviour, not this file's.

## File format

```text
FILE <name>.ppe
  version : int (32-bit)            # 1 or 2

  # then, in exactly this order, with no separators:
  track 0  : ColorTrack   base colour
  track 1  : ColorTrack   additive colour
  track 2  : ColorTrack   grey colour (the per-channel weights)
  track 3  : ValueTrack   grey amount
  track 4  : ValueTrack   blur
  track 5  : ValueTrack   duality, horizontal
  track 6  : ValueTrack   duality, vertical
  track 7  : ValueTrack   noise intensity
  track 8  : ValueTrack   noise grain
  track 9  : ValueTrack   noise rate

  # version 2 only:
  track 10 : ValueTrack   colour-map influence
  cm_tex1  : zero-terminated text    # the colour-mapping texture name

RECORD ValueTrack
  curve : Envelope                  # see Animation/Envelope.hpp for its own layout

RECORD ColorTrack
  base  : real (32-bit)             # serialized, never applied — see Notes
  red   : Envelope
  green : Envelope
  blue  : Envelope
```

**What the version gates** — version 2 adds the colour-mapping influence track and the texture name; nothing else changed. A version 1 file simply ends after track 9, and the reader must stop there. The writer always emits version 2.

**Notes** — the loader reads the version word and then *does not check it* against anything except the `>= 2` test for the tail. A file with a nonsensical version decodes as version 1 and the reader stops early, which is benign. The original had an assertion here and it is commented out; a rebuild should reject an unknown *higher* version rather than guess.

A colour track's leading scalar is read into a field, written back out on save, and never read by anything. It is a vestige of an earlier format. Preserve it byte for byte on the round trip — a tool that drops it shifts every subsequent field.

## `Load`

**Contract** — takes a name and resolves it against the current level's directory first and the shared animation directory second, unless told to treat the name as a host path. A name found in neither is a fatal error with a diagnostic naming the file — loading a missing effect is treated as a broken installation, not a recoverable condition. A file whose extension is not the effect extension is also fatal, with a message about multi-animation files that no longer describes anything real. On success the eleven tracks are filled and the total length is recomputed.

```text
FUNCTION load(name, from_virtual_filesystem)
  IF from_virtual_filesystem
    path <- resolve name under "$level$"
    IF not found: path <- resolve name under "$game_anims$"
    IF not found: FAIL WITH MissingEffect(name)
  ELSE
    path <- name
  IF extension(path) is not the effect extension
    FAIL WITH NotAnEffectFile
  reader <- open(path)
  version <- reader.int32
  FOR i IN 0 .. 9
    params[i].load(reader)
  IF version >= 2
    params[10].load(reader)
    block.cm_tex1 <- reader.zero_terminated_text
  close(reader)
  length <- max over params of track length
```

**Notes** — the level directory is searched *before* the shared one, which is how a level overrides a globally named effect. That precedence is a data-authoring contract and must survive.

## `Save`

**Contract** — writes the version word, then the eleven tracks in order, then the colour-map texture name. Always writes version 2. Used only by the effect editor.

## `Process` — evaluate at a time

**Contract** — advances every track to the given time (each writing its value into the block), clamps the intensity factor into `[0.001, 1]`, copies the block out to the caller, and reports that the effect is still alive. Never reports completion.

**Notes** — this function is a stub of what it once was, and the removed code is still visible in the original as comments: the intensity factor was meant to decay over time toward zero when stopped, the block was meant to be blended from neutral toward the authored values by that factor, and the effect was meant to report completion when the factor reached its floor. None of that happens. In the shipped engine an effect plays at full strength until its owner removes it, and `Stop` only records a rate that nothing consumes.

A rebuild has a choice here and should make it deliberately: either implement the fade the fields plainly describe — which changes how every shipped effect looks as it ends — or delete the factor, the desired factor, the rate and the stop flag, which are otherwise dead weight. Reproducing the original's behaviour means the latter.

The time passed in is the *absolute* animation time, not a delta, despite the parameter being named for one — the tracks are evaluated at it, not advanced by it. The `start_time` field, which would make a delta meaningful, is set to a sentinel and never used.

## `CPostProcessValue` and `CPostProcessColor`

**Contract** — a value parameter loads one keyframed curve and, on evaluation, writes its sample into the float it was constructed around. A colour parameter loads a scalar and three curves and writes three samples into the three channels of the colour it was constructed around. Both report their length as the longest of their curves and their key count as that of their first curve — for a colour, the red channel stands for all three.

The editing operations — insert, delete, update, read a key, clear — apply to whichever channel an index selects, red for 0, green for 1, blue for anything else. Every one of them **forces the inserted or updated key's tension, continuity and bias to zero**, which pins the interpolation to the simple case and is why effects authored through the editor never use the curve evaluator's full expressiveness even though the format can carry it.

**Notes** — reporting a colour's key count from the red channel alone assumes the three channels are keyed in lockstep. The editing operations maintain that by always touching all three together; a file authored by other means need not, and such a file will be mis-edited. Load and evaluation are unaffected.

The lookups that follow an insert search for the key by time within a hundredth of a second and then dereference the result without checking it was found. A key inserted at a time that rounds differently would crash. In practice insert-then-find-the-same-time always succeeds; a rebuild should have insert return the key it made.

## `ResetParam` and `Create`

**Contract** — `Create` builds the eleven parameter objects and binds each to its field in the block, in the order the file format demands. `ResetParam` destroys and rebuilds exactly one of them, which is how the editor clears a track. The two carry the same eleven-way binding table, written out twice.

**Notes** — that duplication is a hazard worth removing: a rebuild should declare the binding once, as data — a table of (index, kind, which field of the block) — and drive construction, reset, load and save from it. The order in that table *is* the file format, and stating it once makes that visible.
