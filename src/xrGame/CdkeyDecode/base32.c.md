# src/xrGame/CdkeyDecode/base32.c

> A base-32 alphabet chosen so a human can read a key aloud without ambiguity, and the
> bit-packing that turns groups of five characters into five bytes — with the byte order
> reversed inside each group.

**Needs** — [`base32.h`](base32.h.md)
**Used by** — reached through its declarations in [`base32.h`](base32.h.md); callers name that, not this file.
**Tier floor** — T2. The packing is a shift register over a fixed-width byte array; the
widths are load-bearing, the storage is not. A rebuild may hold the register as a single
40-bit integer and get identical results.

## Purpose

Converts between a byte string and a text encoding using thirty-two printable characters.
This is not RFC 4648 base32 and must not be replaced by a standard implementation: the
alphabet differs, the padding rule differs, and the byte order within a group differs.
The shipped product keys are frozen text, so all three choices are frozen with them.

## State

```text
CONSTANT alphabet : text = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"
```

Thirty-two characters: the twenty-six letters minus `I` and `O`, plus the digits `2`–`9`.
`I`/`1` and `O`/`0` are dropped because keys are printed on a box and read by a person —
the alphabet's whole point is that no two members look alike in any font. The character's
value is its index in that string; anything outside it is a decoding error. Comparison is
case-sensitive against the upper-case alphabet, which is why a cleaning pass exists.

## `ConvertFromBase32`

**Contract** — takes encoded text and its length, writes the decoded bytes into a
caller-supplied buffer, and returns the number of bytes written. Returns the *negation of
the offending character's code* if a character is not in the alphabet — a negative return
is the only error signal, and callers distinguish it from success by testing for
non-positive. Writes no more than `floor(length/8)*5 + floor((length mod 8)*5/8)` bytes.
Does not allocate, does not validate that the output buffer is large enough.

**Invariants** — input is already cleaned (upper-case, no separators); the caller sized
the output buffer from the maximum key length.

```text
FUNCTION decode_base32(input : text) -> result<bytes, int>
  out <- empty bytes
  cursor <- 0
  WHILE cursor < length(input)
    take <- min(8, length(input) - cursor)      # 8 characters = 40 bits = 5 bytes
    produced <- (take * 5) / 8                  # integer division: a partial group
                                                # yields only whole bytes, the rest is
                                                # discarded, not padded
    register <- 40-bit accumulator, zero

    # The characters of a group are consumed back-to-front. This is the one thing a
    # rebuilder will get wrong: the group's LAST character contributes the HIGHEST
    # five bits, so the group reads as a big-endian number while the register is then
    # copied out little-endian.
    FOR i FROM take - 1 DOWN TO 0
      v <- index of input[cursor + i] IN alphabet
      IF v is absent
        FAIL WITH negative character code
      register <- (register SHIFTED LEFT 5) WITH v placed in the low five bits

    append the low `produced` bytes of register to out, least significant first
    cursor <- cursor + take
  RETURN out
```

**Notes** — the left shift is written as a walk over a five-byte array carrying bits
between neighbours. That is an artifact of a 1990s C target with no 64-bit integer; the
decision it encodes is only *the register is 40 bits wide and shifts left by five per
character*. A rebuild uses whatever integer it has.

A group shorter than eight characters produces only whole bytes and silently drops the
leftover bits. That is not padding in the RFC sense — there is no pad character and no
way to distinguish a truncated key from a short one. The key format sidesteps this by
having a length that is a whole number of five-byte groups.

## `ConvertToBase32`

**Contract** — takes a byte buffer and its length, writes encoded characters into a
caller-supplied buffer, returns the count written. Never fails. The inverse of the
decoder, and the direction the game never uses — only a key *generator* would.

```text
FUNCTION encode_base32(input : bytes) -> text
  out <- empty text
  cursor <- 0
  WHILE cursor < length(input)
    take <- min(5, length(input) - cursor)      # 5 bytes = 40 bits = 8 characters
    produced <- (take * 8 + 4) / 5              # rounds UP: every bit gets a character
    register <- the `take` bytes, least significant byte first, zero-extended to 40 bits
    REPEAT produced TIMES
      out <- out + alphabet[low five bits of register]
      register <- register SHIFTED RIGHT 5
    cursor <- cursor + take
  RETURN out
```

**Notes** — the rounding differs between the two directions: encoding rounds up (`+4`
before dividing by five) so no input bit is lost, decoding rounds down so no output byte
is fabricated. They are inverses only on inputs whose length is a multiple of five bytes.

Note also that encoding emits characters least-significant-group-first while decoding
consumes them last-character-first. The two conventions cancel, which is the only reason
the pair round-trips at all; changing either one alone breaks every shipped key.

## `CleanForBase32`

**Contract** — copies text, dropping `-` and upper-casing anything lower-case, into a
buffer of a stated capacity. Returns success, or failure if the result including its
terminator would not fit — in which case the output is left partially written and must not
be used. Does not allocate.

**Notes** — the separator is dropped unconditionally rather than validated, so a key with
groups of the wrong size still decodes. That is deliberate: the group separators are
presentation, and users retype them inconsistently. Case folding is plain ASCII — the
alphabet is ASCII-only, so a locale-aware fold would be wrong here, matching the rule in
[`SYSTEM-REQUIREMENTS.md` §4](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions).

## `MakeBase32Pretty`

**Contract** — copies encoded text into an output buffer inserting `-` every four
characters, and terminates it. Pure presentation; no validation. The odd part is where the
short group lands: the run length is `4` when the remaining count is a multiple of four
and `remaining mod 4` otherwise, so the *first* group is the short one and every group
after it is exactly four. Writes up to `length + length/4` characters.

**Notes** — leading rather than trailing short group is what makes a key of any length
display with a stable right-hand alignment. The game never calls this; only a key
generator or an account tool would.
