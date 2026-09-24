# src/xrCore/LzHuf.cpp

> The engine's own compressed-blob codec: LZSS over a 4096-byte ring buffer, with the literal/length symbol and the high position bits coded by an adaptive Huffman tree. **Frozen** — shipped archives and shipped chunked files are encoded with it.

**Needs** — [`lzhuf.h`](lzhuf.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`lzhuf.h`](lzhuf.h.md)
**Tier floor** — T1: a bit-exact wire format, byte-order-explicit header, and a hot inner loop over a ring buffer. The *format* is reproducible at any tier; the throughput expectation is not.

## Purpose

This is the older of the two compression schemes the game data uses. It is not the LZO or DEFLATE of [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) — those cover the archive entry payloads; this one covers whole *files* that were stored compressed, and the compressed chunk bodies of the chunked container (see [`FS.cpp`](FS.cpp.md)). A rebuild cannot substitute a different algorithm here: the bytes are on the retail disc.

The algorithm is the 1989 LZHUF of Haruyasu Yoshizaki (by way of Haruhiko Okumura's LZSS), unchanged in substance. Everything below is therefore *specification*, not design: reproduce it exactly.

## State

The codec is a single global machine with no reentrancy: one input cursor, one output buffer, one ring buffer, one Huffman tree. Two threads must not compress or decompress concurrently, and a rebuild that wants concurrency must make every field below per-invocation.

```text
RECORD Codec
  # bit-level I/O
  get_window   : int (16-bit window, MSB first)
  get_bits     : int          # how many bits of get_window are live
  put_window   : int
  put_bits     : int

  # LZSS
  ring         : bytes[N + F]           # N = 4096, F = 60
  # invariant: positions [0, F) are mirrored at [N, N+F) so a match that wraps
  # the ring can be read as one contiguous run without a modulo per byte.

  # match search: a binary search tree over ring contents, keyed by the
  # F-byte string starting at each ring position
  left, right, parent : int[...]        # NIL = N marks an empty link
  match_position : int
  match_length   : int

  # adaptive Huffman over N_CHAR symbols
  freq   : int[T + 1]         # invariant: non-decreasing across [0, T); freq[T] is a
                              # sentinel of 0xffff so the sift loop always terminates
  son    : int[T]             # child index; a value >= T means "leaf for symbol son-T"
  parent_of : int[T + N_CHAR + 1]
```

### Constants — all load-bearing

| Name | Value | Meaning |
|---|---|---|
| `N` | 4096 | ring buffer size; a match distance is 12 bits |
| `F` | 60 | lookahead: the longest match, and the ring's mirror length |
| `THRESHOLD` | 2 | a match shorter than 3 bytes is emitted as literals instead |
| `N_CHAR` | `256 - THRESHOLD + F` = 314 | alphabet: 256 literals plus 58 length codes |
| `T` | `N_CHAR * 2 - 1` = 627 | nodes in the Huffman tree |
| `R` | `T - 1` = 626 | index of the root |
| `MAX_FREQ` | 0x4000 | root frequency at which the tree is rebuilt with halved counts |

Symbol `c < 256` is the literal byte `c`. Symbol `c >= 256` is a match of length `c - 255 + THRESHOLD` — that is, `c - 253` — so lengths 3..60 map to symbols 256..313. The encoder writes `255 - THRESHOLD + length`, which is the same mapping read the other way.

## Stream format

```text
byte 0..3 : uncompressed length, 32-bit, LITTLE-ENDIAN (low byte first)
byte 4..  : the bit stream, most-significant bit of each byte first
```

If the uncompressed length is zero the stream is exactly four bytes and there is no bit stream. A decoder that reads a length of zero must report failure rather than produce an empty result — the engine treats a zero-length compressed blob as a corrupt one.

The bit stream is a sequence of symbols. Each symbol is the adaptive-Huffman code for one alphabet entry, emitted from leaf to root and therefore read root-to-leaf on the other side, bit by bit, taking the smaller child on 0 and the larger on 1. A symbol below 256 stands alone. A symbol at or above 256 is followed by a 12-bit *position*, coded in two halves:

- **High 6 bits** — a static prefix code. The encoder looks the 6-bit value up in a 64-entry pair of tables giving a code length of 3 to 8 bits and the code itself, left-aligned in a byte. The decoder reads one whole byte, looks its value up in a 256-entry pair of tables giving the recovered 6 bits and the total code length, then discards the surplus.
- **Low 6 bits** — verbatim, most-significant first.

The decoder's low-6 recovery is worth spelling out because it is not obvious: after the byte lookup it has `len` bits of prefix consumed out of the 8 it read, so `len - 2` further bits must be pulled in and shifted into the same accumulator; the final 6 low bits of that accumulator are the answer. The distance from the current ring position is `position + 1`.

Both static tables are the LZHUF originals and must be copied verbatim; they are a canonical prefix code chosen so that near distances cost 3 bits of prefix and far ones cost 8. They are printed in the source and are not derivable from a rule.

## The adaptive Huffman tree

The tree is a *sorted array*, not a pointer structure, and that is what makes the update cheap. Nodes `0 .. T-1` are held in non-decreasing frequency order; `son[i]` is either the index of the left child of an internal node (the right child is implicitly `son[i] + 1`) or a leaf marker `>= T`. The root is at `R`.

```text
FUNCTION start_tree()
  FOR i IN 0 .. N_CHAR-1                 # every symbol starts with weight 1
    freq[i] <- 1
    son[i] <- i + T                      # leaf marker
    parent_of[i + T] <- i
  i <- 0
  j <- N_CHAR
  WHILE j <= R                           # pair up, bottom to top
    freq[j] <- freq[i] + freq[i+1]
    son[j] <- i
    parent_of[i] <- j
    parent_of[i+1] <- j
    i <- i + 2
    j <- j + 1
  freq[T] <- 0xffff                      # sentinel: sifting never runs off the end
  parent_of[R] <- 0                      # terminates the walk to the root

FUNCTION bump(symbol)
  IF freq[R] == MAX_FREQ
    rebuild()
  node <- parent_of[symbol + T]
  REPEAT
    freq[node] <- freq[node] + 1
    IF freq[node] > freq[node + 1]       # order violated: swap this node with the
      find the last position l whose freq is still below the new value
      exchange freq[node] and freq[l], and exchange son[node] and son[l],
      repairing parent_of for both subtrees (and for the implicit right child
      of an internal node, which shares the parent slot)
      node <- l
    node <- parent_of[node]
  UNTIL node == 0                        # 0 is the sentinel parent of the root

FUNCTION rebuild()
  # Halve every leaf weight, rounding up, and re-link the tree bottom-up.
  compact all leaves into the front of the array, setting freq <- (freq + 1) / 2
  FOR each internal node j, in order
    f <- freq[2k] + freq[2k+1]           # k walks the pairs below
    insert f into the already-sorted tail by shifting the run that exceeds it
    son[that position] <- 2k
  recompute parent_of for every node
```

**Invariants** — `freq` is non-decreasing over `[0, T)`; the encoder and decoder must apply `bump` to the *same* symbol at the *same* point in the stream, or the trees diverge and every subsequent symbol is garbage. Rounding *up* in `rebuild` matters: it keeps every symbol's weight at least 1 so no symbol becomes uncodeable, and it is what makes the halving reproducible on both sides.

## `_compressLZ` / `_writeLZ` — encode

**Contract** — `_compressLZ` takes a source buffer and its length and returns a newly allocated output buffer and its length; ownership of the buffer passes to the caller, who releases it with the allocator. `_writeLZ` does the same but streams the result to an already-open file handle and returns the number of bytes written, releasing the buffer itself. Neither is thread-safe; neither can fail except by exhausting memory.

```text
FUNCTION encode(src: bytes) -> bytes
  emit 4 bytes: length(src), little-endian
  IF length(src) == 0
    RETURN the 4 bytes
  start_tree()
  clear the match tree
  fill ring[0 .. N-F) with the space character (0x20)
  r <- N - F                            # the insertion point
  read up to F bytes into ring[r ..]; len <- how many arrived
  insert the F overlapping positions ending at r into the match tree
  REPEAT
    clamp match_length to len
    IF match_length <= THRESHOLD
      match_length <- 1
      emit symbol ring[r]               # literal
    ELSE
      emit symbol 255 - THRESHOLD + match_length
      emit position match_position
    slide the window forward by match_length bytes, deleting each departing
      position from the match tree and inserting each arriving one, refilling
      ring from the source; when the source runs dry, keep sliding but stop
      inserting, shrinking len toward zero
  UNTIL len == 0
  flush the remaining partial output byte
```

**Notes** — the output buffer is first sized to the *uncompressed* length and then grown 1024 bytes at a time if the encoder overshoots, which it does on incompressible input. That growth path exists because the format has no expansion bound: worst case each literal costs more than 8 bits.

The initial fill with the space character is part of the format, not an optimization — a match that reaches back before the start of the data decodes as spaces, and both sides must agree. Likewise the starting insertion point `N - F` is fixed.

The match search is a binary search tree over the ring keyed by the F-byte string at each position, with a tie-break that prefers the *nearest* of two equally long matches. The tree gives the longest match in roughly logarithmic time; any structure that finds the same match works, but "longest, nearest on a tie" is the rule the encoder must follow to reproduce a given file byte for byte. Reproducing a byte-identical *compression* is not required for correctness — only decoding is frozen — so a rebuild is free to use a simpler search and accept a different, still-valid encoding.

## `_decompressLZ` / `_readLZ` — decode

**Contract** — `_decompressLZ` takes a source buffer, its length, and an optional upper bound on the decompressed size. It returns success plus a newly allocated output buffer and its length; on failure it returns false and allocates nothing. `_readLZ` reads the compressed bytes from a file handle first, then decodes with no bound. Ownership of the output passes to the caller.

Failure cases, both of which a rebuild must keep because the engine relies on them to reject corrupt data: the recorded length is zero, or the recorded length exceeds the caller's bound. The bound is how the archive reader refuses a blob that claims to expand past the space it has.

```text
FUNCTION decode(src: bytes, limit: optional<int>) -> result<bytes, Corrupt>
  n <- read 4 bytes little-endian
  IF n == 0 OR (limit present AND n > limit)
    FAIL WITH Corrupt
  allocate output of exactly n bytes
  start_tree()
  fill ring[0 .. N-F) with the space character
  r <- N - F
  produced <- 0
  WHILE produced < n
    c <- decode one symbol                # root to leaf; bump(c) afterwards
    IF c < 256
      emit byte c; ring[r] <- c; r <- (r + 1) mod N; produced <- produced + 1
    ELSE
      back <- decode position             # distance is back + 1
      run  <- c - 255 + THRESHOLD
      FOR k IN 0 .. run-1
        b <- ring[(r - back - 1 + k) mod N]
        emit byte b; ring[r] <- b; r <- (r + 1) mod N; produced <- produced + 1
  RETURN output
```

**Notes** — the copy reads from the ring one byte at a time and writes back into the ring as it goes, so a run may legitimately overlap itself (distance 1, length 40 is a valid run of one repeated byte). A block-copy shortcut is therefore wrong unless it is written to handle overlap forward.

The output buffer is sized from the header and the loop is bounded by it, so a stream whose symbols would produce more than the declared length simply stops — but the growth path in the shared output writer means an over-long stream silently reallocates instead of being rejected. A rebuild should treat "the stream produced more than the header promised" as corruption; the original does not.

Nothing in the bit reader distinguishes end-of-input from a zero bit: past the end it feeds zeros forever. Termination is entirely by the declared length. A truncated stream therefore decodes to garbage of the right size rather than to an error, which is worth knowing when diagnosing a bad install.
