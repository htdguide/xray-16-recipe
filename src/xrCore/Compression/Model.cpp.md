# src/xrCore/Compression/Model.cpp

> The statistical model itself: a suffix tree of byte contexts with information-inheritance, secondary escape estimation and a memory-exhaustion policy. **Frozen** — it is the model half of the format every compressed save is written in.

**Needs** — [`PPMd.h`](PPMd.h.md) · [`PPMdType.h`](PPMdType.h.md) · [`Coder.hpp`](Coder.hpp.md) · [`SubAlloc.hpp`](SubAlloc.hpp.md) · [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Used by** — [`PPMd.h`](PPMd.h.md)
**Tier floor** — T1: the tree is a graph of raw pointers into one pool, and the model asks structural questions by comparing those pointers against pool landmarks. Several counters rely on exact byte and 16-bit widths.

## Purpose

This is an adopted public-domain compressor (PPMd variant I / PPMII), reproduced with almost no change. It is not a design this project made, and the recipe's job here is not to justify it but to describe it precisely enough that a rebuild produces byte-identical output — because the engine must read save files and archives it did not write.

Read [`Coder.hpp`](Coder.hpp.md) and [`SubAlloc.hpp`](SubAlloc.hpp.md) first: this file decides *probabilities*, the coder turns them into bits, and the pool is where the structure lives and, crucially, where it runs out.

The one-paragraph summary of the algorithm: predict the next byte from the longest recent context that has seen anything; if that context has not seen this byte, emit an *escape* and fall back to the next shorter context, masking off the symbols already ruled out; repeat down to an order-(-1) uniform model. Everything else in the file is about estimating escape probabilities well, updating the tree cheaply, and surviving when the pool fills.

## State

The whole model is file-scope. There is exactly one of it per process, which is why the compressor is serialized behind a lock.

```text
RECORD Context                          # one node of the suffix tree
  num_stats  : int (8-bit)              # number of symbols MINUS ONE
  flags      : int (8-bit)              # see below
  summ_freq  : int (16-bit)             # total of the symbol frequencies
  stats      : list<State>              # num_stats + 1 entries, kept sorted by
                                        #   descending frequency
  suffix     : optional<Context>        # this context with its first byte dropped

RECORD State
  symbol    : int (8-bit)
  freq      : int (8-bit)               # invariant: 1 <= freq <= MAX_FREQ
  successor : optional<Context>         # this context extended by this symbol,
                                        #   OR a pointer into the pool's text area
                                        #   meaning "not built yet; bytes start here"

RECORD SeeContext                       # secondary escape estimation
  summ  : int (16-bit)                  # scaled running escape estimate
  shift : int (8-bit)                   # its current scale
  count : int (8-bit)                   # countdown to the next rescale
```

**Invariants** — the ones that are enforced by scattered code and written down nowhere else:

- `num_stats` is a count **minus one**, so a value of zero means a context with exactly one symbol. That is not a saving of one byte; it is what makes "binary context" testable as `num_stats == 0`, and the binary path is the hot one.
- When `num_stats == 0` the statistics array pointer is *not used*; the single state is stored **overlapping the `summ_freq` field**. The record is deliberately unioned that way, so a rebuild must either mirror it or, better, make the one-symbol case a separate variant — it costs nothing and the packing is not part of the format.
- `stats` is maintained in descending frequency order at all times. Every update that raises a frequency immediately bubbles the entry up. The search loops scan linearly and stop at the first match, so the ordering is what makes the common case one or two comparisons.
- A `successor` that is numerically **below** the pool's units boundary points into the text area and denotes a context that has not been materialized. Above it, the successor is a real context. The model asks this question constantly; it is the single most important thing to understand about the data structure, and it is why the pool's layout is not an implementation detail.
- A context's `suffix` chain always terminates at the order-0 root, whose suffix is absent.

### Context flags

| Bit | Meaning |
|---|---|
| `0x10` | the byte *preceding* this context was ≥ 0x40 |
| `0x08` | at least one symbol in this context is ≥ 0x40 |
| `0x04` | this context has been rescaled, or was loaded already scaled |

0x40 is the boundary between punctuation/digits and letters in the encodings this was tuned for. The flags feed the escape-estimation context index, so they steer *how much probability escape gets* — they change the output bytes and must be reproduced exactly, including the way they are recomputed on every rescale and refresh.

### Constants — all load-bearing

| Name | Value | Meaning |
|---|---|---|
| `MAX_FREQ` | 124 | a symbol frequency above this forces a rescale |
| `UP_FREQ` | 5 | below this, the frequency-quantization table is the identity |
| `INT_BITS` | 7 | `INTERVAL = 128`, the binary-estimate step |
| `PERIOD_BITS` | 7 | the binary-estimate adaptation rate |
| `TOT_BITS` | 14 | `BIN_SCALE = 16384`, the binary probability denominator |
| `O_BOUND` | 9 | orders above this may be dropped entirely during a prune |
| `MAX_O` | 16 | from [`PPMdType.h`](PPMdType.h.md); the depth of the scratch stack |

`BIN_SCALE` is 2¹⁴ and the coder's carry floor is 2¹⁵ — the binary probability therefore always fits below the floor, which is what makes the binary path's single shift a legal substitute for the general path's division.

### Startup tables

Four tables are built once and never change. Two of them belong to the pool ([`SubAlloc.hpp`](SubAlloc.hpp.md)); the other two are the model's own.

```text
ns_to_binary_index:  0 -> 0,  1 -> 2,  2..10 -> 4,  11..255 -> 6
    # how many symbols the *suffix* context holds, coarsely bucketed; it is the
    # first term of the binary-context index

quantize_freq (260 entries):
    identity below UP_FREQ; above it, a staircase whose tread widens by one at
    each step — 1 value maps to 5, then 2 map to 6, then 3 map to 7, and so on
    # this is a logarithmic-ish bucketing of frequency, and it is what lets
    # 25 binary-estimate rows and 24 escape-estimate rows cover the whole range
```

The quantizer is the reason the estimate tables are small enough to adapt quickly: a model that kept one estimate per exact frequency would learn nothing before the data ran out.

## Escape estimation

Escape probability is the whole difficulty of a PPM model, and this one estimates it two different ways.

**Binary contexts** — a context with one symbol has no statistics to average, so the estimate is looked up in a 25 × 64 table of 16-bit probabilities, indexed by the quantized frequency of the one symbol and by a composite of: how many symbols the suffix context holds, whether the last prediction in this context succeeded, the context's two character-class flags, and **the sign of a run-length counter**. On a hit the entry moves toward certainty by `INTERVAL` minus a rounded mean; on a miss it moves down by that mean. The initial values come from an eight-entry table of tuned constants:

```text
initial_binary_escape = [0x3CDD, 0x1F3F, 0x59BF, 0x48F3, 0x64A1, 0x5ABC, 0x6632, 0x6051]
```

Row *m* of the table is seeded as `BIN_SCALE - initial_binary_escape[k] / (i + 1)`, where *i* counts how many raw frequencies quantize into row *m*, and the eight seeded columns are replicated across all 64. **These eight numbers are not derivable.** They are fitted constants from the original author's corpus and must be copied verbatim.

**Multi-symbol contexts after an escape** — a 24 × 32 grid of adaptive estimators, indexed by the quantized symbol count, by whether the context's total frequency is more than eleven times its symbol count, by whether this context has many more symbols than its suffix plus the masked set, and by the character-class flags. Each estimator holds a scaled sum and reports a mean by subtracting one mean's worth from itself; on use it either doubles its scale (slowing adaptation) or accumulates the observed escape weight. A context holding all 256 symbols cannot escape at all and uses a dummy estimator with scale 1.

A third, cruder estimate is used when a *binary* context escapes and its parent has to start coding: the current binary estimate's top bits index a sixteen-entry table

```text
exponential_escape = [25, 14, 9, 7, 5, 5, 4, 4, 4, 3, 3, 3, 2, 2, 2, 2]
```

which supplies the escape frequency the parent starts from. Also fitted; also copy verbatim.

## `EncodeFile`

**Contract** — read the decoded stream to exhaustion and write the compressed stream. Initializes the coder and the model, runs to end of input, flushes four bytes. Not reentrant, not thread-safe: one global model, one global pool, one set of coder registers. Allocates only from the pool; when the pool is exhausted the restoration policy fires rather than an error surfacing.

```text
FUNCTION encode_stream(out, in, max_order, restoration)
  init_encoder()
  start_model(max_order, restoration)
  LOOP
    ctx := max_context                       # the longest context we have
    c := next byte of in                     # end of input ends the loop below
    IF ctx HAS MORE THAN ONE SYMBOL
      encode_symbol_general(ctx, c); coder.encode_symbol()
    ELSE
      encode_symbol_binary(ctx, c)
    WHILE the symbol was not found
      coder.normalize(out)
      REPEAT                                 # walk down to a shorter context
        order_fall := order_fall + 1
        ctx := ctx.suffix
        IF ctx IS ABSENT THEN BREAK OUT OF EVERYTHING   # terminating escape
      UNTIL ctx HAS MORE SYMBOLS THAN THE MASKED SET
      encode_symbol_masked(ctx, c); coder.encode_symbol()
    IF order_fall == 0 AND the found state's successor is a real context
      max_context := that successor          # the cheap path: just descend
    ELSE
      update_model(ctx)                      # build what is missing
      IF the escape counter wrapped THEN clear the mask
    coder.normalize(out)
  flush_encoder(out)
```

**Invariants** — end of input is signalled by the stream returning a value the model treats as unmatchable, which drives the escape chain all the way past the order-(-1) context; that exhausted chain *is* the end-of-stream marker. There is no length field. A decoder therefore stops exactly where the encoder did, and a truncated stream is not detectable here.

The masked set is a 256-entry byte array compared against a rolling escape counter rather than cleared per symbol — clearing happens only when the counter wraps through zero, once every 256 escapes. That is a real optimization and it is observable: the model's behaviour at the wrap is part of the format.

## `DecodeFile`

**Contract** — the exact mirror. Same model construction, same update points, same restoration triggers. Writes the decoded bytes to the output cursor.

**Invariants** — the decoder must call the coder's count query exactly once per symbol, because that query narrows the range (see [`Coder.hpp`](Coder.hpp.md)). Every divergence between the two functions above is a stream that decodes to garbage from that byte onward, silently.

## Coding one symbol

### `encodeSymbol1` / `decodeSymbol1` — a context with several symbols, nothing masked

```text
FUNCTION code_general(ctx, c)
  sub.scale := ctx.summ_freq
  IF the first (most frequent) state matches
    remember whether it took at least half the total     # prev_success
    bump its frequency by 4, bump the total by 4
    extend the run length by prev_success
    IF the frequency now exceeds MAX_FREQ THEN rescale(ctx)
    sub.low := 0; sub.high := that frequency
    RETURN
  scan forward, accumulating a cumulative low
  IF found
    sub.low, sub.high := the cumulative bracket
    promote(ctx, the found state)
  ELSE
    sub.low := the whole accumulated total; sub.high := sub.scale   # escape
    mask every symbol of this context
    record how many were masked
```

**Notes** — the *first* state is special-cased with its own branch because in a well-trained model it is the answer most of the time and the general loop's cumulative-sum bookkeeping is pure cost on that path.

`prev_success` — "the top symbol held at least half the probability mass" — feeds the binary estimator's index and the run-length counter. It is a cheap proxy for *this context is confident*, and the model spends it on deciding how sharply to sharpen.

### `encodeSymbol2` / `decodeSymbol2` — after an escape, with symbols masked

```text
FUNCTION code_masked(ctx, c)
  see := pick_escape_estimator(ctx)            # also sets sub.scale to its mean
  walk the states, skipping masked ones, accumulating frequencies
  IF found
    sub.low, sub.high := its bracket
    sub.scale := sub.scale + the total of all unmasked frequencies
    tell the estimator it was used                  # adapt its scale
    promote_after_escape(ctx, the found state)
  ELSE
    sub.low := the unmasked total; sub.high := sub.scale
    add the whole scale into the estimator's running sum   # it escaped again
    mask everything in this context
```

**Invariants** — the escape estimator's mean is added to the scale *before* the symbol frequencies, so escape occupies the top of the interval. Both sides must accumulate in the same order, because the accumulation is integer and the scale is the coder's denominator.

The decoder collects pointers to the unmasked states into a stack as it goes, because it must walk them twice — once to total, once to locate the count. The encoder knows the symbol and walks once. This is the one place the two functions differ structurally while remaining arithmetically identical.

### `encodeBinSymbol` / `decodeBinSymbol` — a context with one symbol

```text
FUNCTION code_binary(ctx, c)
  index := ns_to_binary_index[ctx.suffix.symbol_count]
           + prev_success + ctx.flags + (32 IF run_length is negative)
  p := binary_table[quantize_freq[state.freq - 1]][index]
  threshold := coder.bin_start(p, TOT_BITS)
  IF the one symbol matches
    raise its frequency (capped at 196)
    coder.bin_correct_zero(threshold)
    p := p + INTERVAL - rounded_mean(p)
    prev_success := 1; run_length := run_length + 1
  ELSE
    coder.bin_correct_one(threshold, BIN_SCALE - p)
    p := p - rounded_mean(p)
    init_escape := exponential_escape[p >> 10]   # what the parent starts from
    mask this symbol; masked count := 0
    prev_success := 0
    report "not found"
```

**Invariants** — the frequency cap here is **196, not MAX_FREQ**. A binary context's single frequency is not a coding weight; it is the quantizer's input, and 196 is the top of the useful range of that table. Using MAX_FREQ instead changes which table row is consulted and therefore the output.

The run-length counter's *sign* selects between two halves of the binary table. It starts at `-(min(max_order, 12) + 1)` — so it is negative until the model has had a run of that many consecutive confident hits — and is reset to that value whenever a symbol is found only after an escape. So the two halves of the table are "the model is currently on a roll" and "it is not", and the model learns separate escape behaviour for each.

## Updating after a symbol

### `update1` / `update2` — promotion

```text
FUNCTION promote(ctx, state)
  state.freq := state.freq + 4
  ctx.summ_freq := ctx.summ_freq + 4
  IF state now outranks its predecessor
    swap them                               # keep the descending order
    IF the frequency exceeds MAX_FREQ THEN rescale(ctx)

FUNCTION promote_after_escape(ctx, state)
  state.freq := state.freq + 4
  ctx.summ_freq := ctx.summ_freq + 4
  IF the frequency exceeds MAX_FREQ THEN rescale(ctx)
  escape_counter := escape_counter + 1       # drives the periodic mask clear
  run_length := initial_run_length           # the roll is broken
```

The increment is 4, not 1, in both. Combined with `MAX_FREQ = 124` that gives a context roughly thirty promotions before it rescales, which sets the model's adaptation speed. A rebuild that changes either number changes every output byte.

### `rescale` — halving a saturated context

```text
FUNCTION rescale(ctx)
  move the just-found state to the front (bubble it all the way)
  give it +4 more, and the total +4
  adder := 1 IF the model is behind or frozen, ELSE 0     # round up or down
  FOR EACH state, front to back
    state.freq := (state.freq + adder) >> 1
    re-insert it into descending order as you go
  IF any frequencies fell to zero
    drop that whole tail; add its count back into the escape weight
    IF only one symbol survives
      collapse the context to the binary form, with its frequency scaled to
        (2*freq + escape - 1) / escape, capped at MAX_FREQ / 3
      release the statistics array back to the pool
      RETURN
    shrink the statistics array and recompute the character-class flag
  escape_weight := escape_weight - (escape_weight >> 1)   # halve it too
  add it back into the total; mark the context as rescaled
```

**Invariants** — the rounding direction is conditional: **round up while the model is still catching up (or frozen), round down when it is current**. Rounding up preserves rare symbols that the model has not yet had a chance to confirm; rounding down lets them die. Getting this backwards does not crash, it just compresses worse — and produces a different stream.

The collapse-to-binary path caps the surviving frequency at a third of `MAX_FREQ`, which keeps the newly-binary context from immediately looking maximally confident about a symbol it has only seen a few times.

### `refresh` — rescaling without a found symbol

Used by the restoration paths. Shrinks the statistics array to fit the current symbol count, halves every frequency (with the same conditional rounding, here passed in), recomputes the escape weight as the residue, and rebuilds the character-class flag from scratch. It preserves the "has been rescaled" flag only when the caller says the operation counts as a scaling.

## Growing the tree

This is the part that gives PPMII its name — *information inheritance*. A newly created context does not start from ignorance; it starts from an estimate derived from its parent's statistics.

### `CreateSuccessors`

**Contract** — materialize the chain of contexts that a deferred successor pointer stands for, and return the deepest one. Walks up the suffix chain collecting every state whose successor is still the same deferred pointer, then walks back down creating a real context for each, all sharing one inherited initial state. Returns nothing if the pool is exhausted, which signals the caller to restore.

```text
FUNCTION create_successors(skip_first, known_state, ctx) -> optional<Context>
  up := the found state's deferred successor       # a text-area address
  collect := empty stack
  IF NOT skip_first THEN push the found state
  WALK UP the suffix chain from ctx
    find (or fall into) the state for our symbol, nudging its frequency up
    IF its successor differs from `up`
      ctx := that successor                        # we have reached solid ground
      BREAK
    push that state
  IF nothing was collected THEN RETURN ctx

  # the inherited seed: one symbol, read from the text area at `up`
  seed.symbol    := the byte at `up`
  seed.successor := `up` + 1                       # still deferred, one deeper
  IF ctx has several symbols
    let cf be the found symbol's frequency minus one
    let s0 be ctx's total minus its symbol count minus cf
    seed.freq := 1 + (IF 2*cf <= s0 THEN (5*cf > s0) ELSE (cf + 2*s0 - 3) / s0)
  ELSE
    seed.freq := ctx's single frequency

  FOR EACH collected state, deepest first
    allocate a context; give it the seed and `ctx` as its suffix
    OR RETURN none IF the pool is exhausted
    point that state at it; ctx := it
  RETURN ctx
```

**Invariants** — the seed frequency formula is the heart of information inheritance and is not reconstructible from first principles; it is an empirical fit. What it expresses: a symbol that dominates its parent context (`2*cf > s0`) should enter the child already strong, and the two branches interpolate that with integer arithmetic chosen to avoid division in the common case. Reproduce it exactly.

The nudges applied while walking up — plus 2 if below `MAX_FREQ - 9` in a multi-symbol context, plus 1 up to 24 or 32 in a binary one — are updates to contexts the coder did *not* just use. They exist because those contexts did predict this symbol correctly, just at an order the coder never reached, and the model credits them. They change the output, so they are not optional.

### `ReduceOrder`

**Contract** — the other half of the growth path, taken when the found state has no successor at all. Points the found state and every state above it at the current text-area cursor, so that a future `create_successors` can build the chain, and returns the context the model should continue from. When the model is frozen it instead points the whole collected chain at a real context and resets the text cursor, which is what "frozen" means operationally: no more text accumulates.

### `UpdateModel`

**Contract** — run after every symbol that the cheap descend-only path could not handle. Credits the suffix context, materializes or defers the successor, and then adds the just-coded symbol to every context between the longest and the one that actually coded it. Allocates from the pool; on exhaustion jumps to restoration. This is the single most intricate function in the file and the one a rebuild will spend the most time on.

```text
FUNCTION update_model(min_ctx)
  IF the found frequency is below MAX_FREQ/4 AND a suffix exists
    credit the symbol in the suffix context (and re-sort if it outranks a peer)

  IF order_fall == 0 AND the found state has a successor
    build it out and descend; RETURN                   # nothing to append

  append the coded symbol to the text area
  successor := the new text cursor
  IF the text area has met the units area THEN RESTORE

  IF the found state had a successor
    IF it was deferred THEN materialize it
  ELSE
    defer it                                           # reduce_order
  IF that failed THEN RESTORE

  IF order_fall reaches zero
    successor := the real context; give back the text byte when the longest
      and coding contexts differ

  # inherit into every context between the longest and the coding one
  s0 := coding context's total, minus its symbol count, minus the found frequency
  FOR EACH context from the longest down to the coding one
    IF it has symbols
      grow its statistics array by one if the size class demands it  OR RESTORE
      nudge its total when it is much smaller than the coding context
    ELSE
      promote it from binary form to an array   OR RESTORE
      double the single frequency (or clamp near MAX_FREQ)
      seed the total from that frequency, the binary escape estimate, and
        whether the coding context held more than three symbols
    # the inherited frequency for the new entry:
    cf := 2 * found_frequency * (its total + 6)
    sf := s0 + its total
    IF cf < 6*sf
      cf := 1 + (cf > sf) + (cf >= 4*sf);   its total := its total + 4
    ELSE
      cf := 4 + (cf > 9*sf) + (cf > 12*sf) + (cf > 15*sf);  its total += cf
    append the symbol with frequency cf and the successor
    propagate the character-class flag
  max_context := the successor
```

**Invariants** — the two-branch `cf` formula is the second empirical fit and the same warning applies: the thresholds 6, 9, 12, 15 and the constants 4 and 6 are tuned, not derived. They map "how strongly did the deeper context believe this" onto a small integer frequency for the shallower one.

The statistics array is grown only when the new symbol count crosses a size-class boundary, which the pool answers in constant time. That is the reason the pool's size classes step the way they do.

The **order in which contexts are visited is load-bearing**: from the longest down to the one that coded, so that each one's total already reflects the changes below it when `sf` is computed.

## Running out of memory

### `RestoreModelRare`

**Contract** — the pool is exhausted. Undo the partial update that was in flight, then apply the configured restoration policy. Never fails; the restart branch always succeeds because it releases everything.

```text
FUNCTION restore_model(partial_ctx, coding_ctx, successor)
  # 1. roll back the half-finished append
  reset the text cursor to the start of the text area
  FOR EACH context from the longest down to where the update stopped
    remove the symbol just appended; collapse back to binary if it was the last,
      scaling the survivor's frequency by (freq + 11) >> 3
    otherwise refresh it without counting it as a scaling
  FOR EACH context from there down to the coding context
    decay it slightly: halve a binary frequency, or nudge the total by 4 and
      refresh (as a scaling) once it has grown past 128 + 4 * symbol_count

  # 2. apply the policy
  IF already past frozen
    continue from the successor; nudge the pool's glue budget
  ELSE IF frozen
    walk to the root, strip every binary-only branch, and enter the past-frozen
      state; reset the fall counter to the full order
  ELSE IF the policy is restart, OR less than half the pool is actually in use
    start the model over from scratch
  ELSE
    walk to the root and prune repeatedly, recovering text area after each pass,
      until under three quarters of the pool is in use
```

**Invariants** — "less than half the pool is in use" is the escape hatch that turns a *cut-off* policy into a *restart* when pruning cannot possibly help: the pool is not full of model, it is full of fragmentation. The three-quarters target for the prune loop is what stops it pruning forever. Neither fraction is derived; both are observable in the output stream, because they decide *when* the model changes shape.

The `(freq + 11) >> 3` rescale on collapse-to-binary appears in three places in this file with the same constants and is the model's standard "this symbol is now the only one; how confident should it be" rule.

### `cutOff` — pruning

**Contract** — recursive. Drops every successor beyond the maximum order, drops every state whose successor points into the text area (those are deferred and cheap to rebuild), compacts survivors toward the low end of the pool, and frees a context entirely when it has lost all its symbols and sits deeper than `O_BOUND`. Returns the surviving context or nothing.

**Notes** — `O_BOUND = 9` is why a prune never destroys the shallow part of the tree: orders up to nine are the ones that carry most of the model's value and are the most expensive to relearn. Deeper contexts are rebuilt quickly from the text that is still there.

### `removeBinConts` — the freeze preparation

**Contract** — recursive. Drops every deferred successor, then frees any binary context whose single symbol has no successor and whose suffix is itself binary or fully flagged. This runs once, at the transition into the frozen state, and its job is to leave a tree that is still useful but will never need to grow again.

## Starting and loading

### `StartModelRare`

**Contract** — initialize (or reinitialize) the model for one stream. Clears the mask, resets the counters, initializes the pool's layout, seeds both estimate tables, and creates the root context — either as a flat order-0 model over all 256 bytes with frequency 1 each, or from a caller-supplied **trained model**.

**Invariants** — the run-length counter's initial value is `-(min(max_order, 12) + 1)`, which is the sign trick the binary table's index depends on. The clamp at 12 means orders above 12 share one starting value.

**Notes** — the function has two branches guarded by a first-time flag, and **the assignment that would clear that flag is commented out in the source**, so the second branch is unreachable and the pool's layout is re-initialized on every stream. The dead branch would have preserved a trained model across streams. Whether the live behaviour is the intended one cannot be recovered from the source; a rebuild should implement only the live path and note the gap.

### `read` — the trained-model format

A caller may install a serialized model so that a short stream compresses well without having to learn anything first. The format is a pre-order walk of the context tree, byte-oriented, with no header beyond the first byte and no length fields — recursion is driven by a flag bit inside each frequency byte.

```text
per context:
  1 byte   symbol count minus one
  IF that is zero:                    # binary context
    1 byte  frequency, with bit 7 set IF a successor follows
    1 byte  symbol
    IF bit 7 was set: the successor context, recursively
  ELSE:
    (count + 1) pairs of:
      1 byte  frequency, with bit 7 set IF a successor follows
      1 byte  symbol
    then, for each pair in order, the successor context if its bit was set
```

The frequencies are **delta-coded against the previous entry**, not stored directly: the first symbol is assigned 64, and each subsequent one gets the difference between the previous stored byte and its own. The first entry's stored byte (with the flag stripped) is the *escape* weight, which also seeds the total.

If that escape weight exceeds 32, the whole context is rescaled on load: the total is halved and every frequency is reduced to a quarter of itself. That is the loader's way of saying "this trained context is confident enough that it should not dominate a fresh stream".

Character-class flags are recomputed on load from the symbols and from the byte that preceded the context, which the caller passes down the recursion; the top-level call passes 0xFF.

**Invariants** — the serialized form carries **no suffix links**. They are reconstructed afterwards by `makeSuffix`, which walks the tree and, for each state with a successor, finds the same symbol in the parent's suffix context and links the child to *that* state's successor. This works only because the tree is complete in the suffix sense — every context's suffix exists — which the writer guaranteed. A hand-built trained model that violates it makes the reconstruction search run off the end of a statistics array.

**Notes** — the writer for this format is present in the source **as a commented-out block**. The engine ships no trained model and never calls the trained entry points with one, so this whole path is dormant. It is documented here because the format is otherwise unrecoverable: the only description of it is the reader, and a rebuild that wants the capability needs both halves.

## What could not be recovered

- The eight binary-escape seeds, the sixteen exponential-escape values, the inheritance frequency formulas and their thresholds, the glue budget, and the 16 KiB compaction threshold are all fitted constants with no derivation anywhere in the source.
- The first-time flag whose clearing is commented out leaves the intended lifetime of a trained model ambiguous.
- The progress hook the model calls every 256 escapes is bound to an empty function; nothing indicates what it was meant to report.
