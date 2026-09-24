# src/xrCore/Compression/Coder.hpp

> The arithmetic coder under the statistical model: a carryless range coder that turns (low, high, scale) triples into a byte stream and back. **Frozen** — it defines the bit layout of every compressed save.

**Needs** — [`PPMdType.h`](PPMdType.h.md) · [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Used by** — [`Model.cpp`](Model.cpp.md)
**Tier floor** — T1: the whole algorithm is exact 32-bit unsigned arithmetic with deliberate wraparound, and the output bytes are the format.

## Purpose

The model in [`Model.cpp`](Model.cpp.md) decides *what probability* a symbol has; this file decides *how that probability becomes bits*. The split is the classic one and worth keeping: any model that produces subrange triples can drive this coder, and any coder that consumes them can replace it — but not in a stream that must still decode, because the coder's normalization rule is part of the format.

It is a *carryless* range coder: instead of propagating a carry out of the low register, it forces a normalization whenever the range gets small enough that a carry could occur. That costs a fraction of a percent of ratio and removes the carry-propagation buffer entirely.

## State

One set of registers, at file scope, shared by the encoder and the decoder — the same global-state problem the model has, and the reason the whole compressor is serialized behind one lock.

```text
RECORD Coder
  low     : int (32-bit, wraps)   # bottom of the current interval
  range   : int (32-bit, wraps)   # width of the current interval
  code    : int (32-bit, wraps)   # decoder only: the value being decoded
  sub     : Subrange              # what the model just decided

RECORD Subrange
  low   : int      # cumulative frequency below the symbol
  high  : int      # cumulative frequency at and above it
  scale : int      # total frequency; invariant: high <= scale
```

### Constants

| Name | Value | Meaning |
|---|---|---|
| `TOP` | 2²⁴ | a byte is output when the top 8 bits of `low` and `low + range` agree |
| `BOT` | 2¹⁵ | the range floor below which a carry becomes possible |

`BOT` is **2¹⁵ here, not the 2¹⁶ of the coder this was derived from.** The original text is reproduced verbatim in the file's comment block with `BOT = 1 << 16`, and the live code uses `1 << 15`. That halving lets the model use total frequencies up to 2¹⁵ where the original allowed 2¹⁶, and it changes the normalization boundary, so it changes the output bytes. It is not an optimization a rebuild may undo: the shipped compressed data was produced with 2¹⁵.

## Normalization — the one rule both sides must share

```text
FUNCTION normalize(emit_or_consume_one_byte)
  WHILE (low XOR (low + range)) < TOP
        OR (range < BOT AND SET range := (negate(low) AND (BOT - 1)))
    emit_or_consume_one_byte()          # encoder writes low >> 24
                                        # decoder folds a byte into code
    range := range << 8
    low   := low << 8
```

**Invariants** — the second clause has a side effect inside the condition, and that is load-bearing rather than clever: when the range has shrunk below `BOT` without the top bytes agreeing, the range is *truncated* to whatever fits below the next multiple of `BOT` above `low`. That truncation is what makes a carry impossible, and it must happen at exactly the same points on both sides. Encoder and decoder run identical normalization loops; a difference of one iteration desynchronizes the stream permanently.

Both `low` and `range` are 32-bit and both **rely on wraparound** — `low + range` is allowed to exceed 32 bits and discard the overflow, and `negate(low)` is the two's-complement negation of an unsigned value. A rebuild in a language that traps on overflow must mask explicitly.

## Encoding

```text
FUNCTION init_encoder()
  low := 0
  range := all_ones                       # 2^32 - 1

FUNCTION encode_symbol()                  # after the model has filled sub
  range := range / sub.scale              # integer division, truncating
  low   := low + sub.low * range
  range := range * (sub.high - sub.low)

FUNCTION flush_encoder()
  REPEAT 4 TIMES                          # disambiguate the final interval
    emit byte (low >> 24)
    low := low << 8
```

**Invariants** — the division happens *before* the multiply and its truncation is part of the arithmetic: the decoder performs the identical truncating division, so the two agree on a range that is not exactly `range / scale * scale`. Reordering for precision breaks the stream.

The flush writes exactly four bytes. The decoder primes itself with exactly four bytes. That pairing is why a compressed buffer's reported length must include the last of those four (see the off-by-one note in [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md)).

## Decoding

```text
FUNCTION init_decoder(stream)
  low := 0; code := 0
  range := all_ones
  REPEAT 4 TIMES
    code := (code << 8) OR next_byte(stream)

FUNCTION current_count() -> int           # which cumulative frequency are we in?
  range := range / sub.scale              # same truncating division as the encoder
  RETURN (code - low) / range

FUNCTION remove_subrange()                # after the model has resolved the symbol
  low   := low + range * sub.low
  range := range * (sub.high - sub.low)
```

**Invariants** — `current_count` **mutates `range`**. It is a query with a side effect, and the model calls it exactly once per symbol, before deciding. Calling it twice divides the range twice and destroys the stream. A rebuild should split it into "narrow the range to one scale unit" and "read the count", or make the mutation obvious in the name.

The source's own ancestor threw on a count at or above the total frequency — the corruption check. **This version does not check**: a corrupt stream produces a count outside the model's table and walks off the end of it. That is a real robustness gap and a rebuild should reinstate the bound, since the compressed data it reads may be a damaged save file.

## Binary contexts

A context holding exactly one symbol is coded as a single binary decision rather than through the general path, with the probability held as a scaled 16-bit number. Four operations serve it:

```text
FUNCTION bin_start(p_zero, shift) -> int   # narrow range by the shift, return the
  range := range >> shift                  #   threshold for the "expected" branch
  RETURN p_zero * range

FUNCTION bin_decode(threshold) -> bool     # decoder: did we land above it?
  RETURN (code - low) >= threshold

FUNCTION bin_correct_zero(threshold)       # the expected symbol occurred
  range := threshold

FUNCTION bin_correct_one(threshold, p_one) # it escaped
  low   := low + threshold
  range := range * p_one
```

**Notes** — the binary path exists because single-symbol contexts dominate a high-order model: most of the tree is "after this exact byte sequence, the next byte has always been X". Coding those through the general cumulative-frequency machinery would cost a division per symbol for a decision that needs one comparison. The ratio gain is secondary; the speed gain is the point.

`bin_start`'s shift is the model's total-probability bit count, so `range >> shift` is the same truncating division the general path performs, written as a shift because the denominator is a power of two here.
