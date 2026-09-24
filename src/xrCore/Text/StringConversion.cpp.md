# src/xrCore/Text/StringConversion.cpp

> Decodes a byte string into 16-bit characters while recording where each one started — and falls back to treating the bytes as characters when the decode fails, because the shipped text is not all one encoding.

**Needs** — [`StringConversion.hpp`](StringConversion.hpp.md) · [`xrDebug.h`](../xrDebug.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`StringConversion.hpp`](StringConversion.hpp.md)
**Tier floor** — T2: byte-level decoding with a byte-to-character index map.

## Purpose

The user interface lays text out character by character but stores and edits it as bytes. Every caret position, every selection range and every line break is a byte offset that must be converted to a character index and back. This file produces both at once: the decoded characters, and the parallel table mapping character index to byte offset.

The second half of the file is the localization bridge: the shipped string tables are in one of several single-byte code pages depending on language (system requirements §4), and anything the engine exchanges with the outside world — a saved name, a console line — must be converted.

## State

Stateless.

## `mbhMulti2Wide` — decode with a position map

**Contract** — takes a byte string and up to two output arrays: the decoded characters, and the byte offset at which each decoded character started. Either output may be omitted, so the function serves both "how long is this in characters" and "give me the characters". Returns the character count. An empty input returns zero and writes nothing. When an output array is supplied its capacity is given and overrunning it is an assertion, not a truncation.

```text
FUNCTION decode(src: bytes, want_chars: bool, want_positions: bool) -> int
  IF src is empty
    RETURN 0
  n <- 0
  cursor <- 0
  WHILE src[cursor] is not the terminator
    IF want_positions: positions[n] <- cursor
    lead <- src[cursor]; advance cursor
    IF lead has the 1-byte form                 # high bit clear
      ch <- lead
    ELSE IF lead has the 2-byte lead form       # top three bits are 110
      read one continuation byte
      IF it is not a continuation: RESTART IN FALLBACK MODE
      ch <- (low 5 bits of lead) << 6 OR (low 6 bits of continuation)
    ELSE IF lead has the 3-byte lead form       # top four bits are 1110
      read two continuation bytes, same check on each
      ch <- (low 4 bits of lead) << 12 OR (low 6 of first) << 6 OR (low 6 of second)
    ELSE
      RESTART IN FALLBACK MODE
    n <- n + 1
    IF want_chars: chars[n] <- ch               # note the index — see below
  IF want_positions: positions[n] <- cursor     # one past the last, so a range is
                                                # always [positions[i], positions[i+1])
  IF want_chars
    chars[n + 1] <- 0                           # terminator
    chars[0] <- n                               # the LENGTH lives in slot zero
  RETURN n
```

**Invariants** — the character array is **one-based with the length stored in slot zero**. That is the single most surprising thing in this file and it propagates into every caller: element `i` of the text is at index `i`, not `i - 1`, and the array must be allocated with two slots of slack beyond the character count. A rebuild should use an ordinary array plus a separate length and fix the call sites; keeping the convention is not worth it.

The position array is one longer than the character count, with the extra entry holding the offset one past the end. That is what makes "the bytes of character *i*" a half-open range without a special case at the end.

**Notes** — sequences of four bytes or more are not decoded at all; they fall into the mismatch branch. The engine's characters are sixteen bits wide, so anything outside the basic multilingual plane has nowhere to go regardless.

## The fallback

**Contract** — when any byte fails the multi-byte structure check, the whole decode restarts from the beginning in a mode that maps **each byte to one character of the same value**. The already-decoded prefix is discarded and redone. The result is the same shape — one-based array, length in slot zero, position table one longer.

**Notes** — this is not error recovery, it is *encoding detection by failure*. The shipped string tables are single-byte code pages, and a byte in the upper half of such a page is very likely to look like a malformed multi-byte lead. Falling back to byte-per-character means those tables decode to the code page's own byte values, which is exactly what the font atlas is indexed by. A file that really is multi-byte-encoded decodes correctly on the first pass.

The cost is a full restart on the first high byte, which for a Cyrillic or Central European string is the first non-Latin character. That is a per-string cost on a path called per frame in the worst case, and it is the reason a rebuild should detect the encoding once at load time — the localization's code page is known from configuration — rather than per string.

The strict mode, where a malformed sequence is an assertion instead, is still in the source behind a compile-time switch and is not the shipping behaviour. A rebuild reproducing the shipped engine must take the lenient path.

## `StringFromUTF8` and `StringToUTF8`

**Contract** — convert a byte string between a portable multi-byte encoding and the single-byte representation of a supplied locale. A character with no representation in the target single-byte encoding becomes a question mark; the reverse direction has no lossy case. Both allocate their result.

**Notes** — the round trip is not the identity for any text outside the locale's code page, which is expected and is why the engine stores text in its own encoding and converts only at the edges.

Both routes go through the language's wide string as an intermediate, which on some platforms is 32 bits wide and on others 16 — irrelevant here, because the values involved all fit in the smaller. A rebuild should convert directly between the two byte encodings.

The reverse conversion sizes its intermediate from the source's *byte* length, which is correct only because the source is single-byte by construction at every call site.
