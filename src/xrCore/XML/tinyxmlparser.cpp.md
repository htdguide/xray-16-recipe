# src/xrCore/XML/tinyxmlparser.cpp

> The grammar: how a byte stream becomes the tree — including the encoding decision, the entity table, and every leniency the shipped data needs.

**Needs** — [`tinyxml.h`](tinyxml.h.md) · [`tinystr.h`](tinystr.h.md)
**Used by** — [`tinyxml.h`](tinyxml.h.md)
**Tier floor** — T1: byte-level lead-byte dispatch, a byte-order-mark probe, and a hand-rolled character-reference conversion.

## Purpose

The read side of the engine's XML, and therefore part of what §5 of the system requirements calls frozen: the interface layouts and the string tables must parse the way they parse today, including the several places where this parser is *more permissive* than the standard. A strict parser will reject shipped files. Everything below that is marked as leniency is a requirement, not a defect to correct.

## State

Parsing is a cursor walking a zero-terminated byte image, plus a position tracker maintaining row and column for diagnostics. There is no push-back and no lookahead beyond a few bytes; each production consumes and returns the new cursor, or nothing on failure.

```text
RECORD Position
  row, col : int      # 1-based, in CHARACTERS not bytes; a tab advances col to
                      # the next multiple of the configured tab width
```

**Invariant** — the column advances by *characters*, so under multi-byte decoding it consults the lead-byte length table. Under byte-per-character it advances by one per byte. The row/column pair is diagnostic only; the engine reports the description and not the position.

## The encoding decision

This happens once, at the very start of the document, and governs every production afterwards.

```text
FUNCTION decide_encoding(bytes, requested) -> Encoding
  IF requested is not UNKNOWN
    RETURN requested                  # the caller forced it
  IF the first three bytes are the byte-order mark (0xEF 0xBB 0xBF)
    skip them
    RETURN MULTI_BYTE
  IF a declaration follows and its stated encoding name begins with the
     multi-byte encoding's name, compared case-insensitively
    RETURN MULTI_BYTE
  RETURN BYTE_PER_CHARACTER
```

**Invariants** — the default is byte-per-character. That is what makes the shipped single-byte localized files decode with their high bytes intact — see [`tinyxml.h`](tinyxml.h.md). A rebuild that defaults the other way corrupts every non-Latin localization.

Under multi-byte decoding, a lead byte's sequence length comes from a 256-entry table; an invalid lead byte is assigned length **1** rather than being rejected, so malformed sequences pass through as their own bytes instead of failing the parse. That is a deliberate leniency.

Three byte sequences are additionally recognized and **skipped as whitespace** wherever whitespace is skipped: the byte-order mark itself, and two other non-characters in the same block. Encountering a mark mid-document is not an error.

## Character classes

- **Whitespace** — space, horizontal tab, carriage return, line feed. Nothing else, and notably not the vertical tab or form feed.
- **A name's first character** — a Latin letter or underscore. A byte at or above 128 is *also* accepted as a letter, unconditionally and regardless of encoding, so a tag or attribute name may contain any high byte.
- **A name's subsequent characters** — the above, plus digits, hyphen, period and colon.

**Notes** — accepting every high byte as a letter is the second deliberate leniency. It means the classifier never needs to know the encoding, and it lets names in a single-byte code page parse. The standard's actual name rules are far narrower and would reject shipped files.

## Entities and character references

The five named entities in [`tinyxml.h`](tinyxml.h.md) are matched by prefix against a fixed table, in table order. A numeric reference is recognized in decimal (`&#` digits `;`) or hexadecimal (`&#x` digits `;`), and is parsed **backwards** from the terminating semicolon, accumulating with a running place multiplier — an implementation detail, but it means a reference with no terminating semicolon is rejected rather than run away.

```text
FUNCTION read_entity(cursor) -> (character_or_sequence, consumed)
  IF cursor is "&#"
    locate the terminating ';'; if absent, FAIL
    accumulate the digits from the last one backwards, base 10 or 16
    IF encoding is MULTI_BYTE
      emit the code point as a multi-byte sequence
    ELSE
      emit the code point's low byte only          # truncation, deliberate
    RETURN
  FOR EACH e IN the five-entry table
    IF cursor begins with e.text
      emit e.character; RETURN
  # no match: emit the ampersand itself, consume one byte, and carry on
  emit '&'; consume 1
```

**Invariants** — an unrecognized entity is **not an error**. The ampersand is emitted literally and parsing continues with the next byte. This is the third deliberate leniency and the shipped data relies on it: stray ampersands appear in interface captions.

Under byte-per-character, a numeric reference above 255 is truncated to its low byte, silently. Nothing in the shipped data does this.

## The productions

**Document** — skip any byte-order mark, then skip whitespace, then repeatedly identify and parse a top-level node until the input is exhausted. An input that is empty, or that is only whitespace, is the *document empty* error. A nested document node is the *top only* error.

**Node identification** — from the first bytes after a `<`:

| Prefix | Node |
|---|---|
| `<?xml` (case-insensitive) | declaration |
| `<!--` | comment |
| `<!` | unknown (a doctype or entity declaration; captured raw) |
| `<?` | unknown (a processing instruction; captured raw) |
| `<` followed by a name character or underscore | element |
| anything else | the *parsing unknown* error |

**Element** — read the tag name; then loop: skip whitespace and read attributes until one of three terminators. `>` opens the content; `/>` closes an empty element; anything else is the *reading attributes* error. Inside the content, text runs, child elements, comments and unknowns are read until the matching `</name>`; a mismatched or missing close is the **end-tag** error — the one the engine selectively forgives (see [`XMLDocument.cpp`](XMLDocument.cpp.md)).

An element may contain a `<![CDATA[ ... ]]>` section, in which case the text is taken verbatim with **no entity expansion and no whitespace condensing**, and the resulting text node remembers that it came from such a section.

**Attribute** — a name, optional whitespace, `=`, optional whitespace, then a value in single or double quotes. Entities inside the value *are* expanded. An unquoted value is read to the next whitespace or `>` — a fourth leniency, since the standard requires quotes.

**Text** — bytes up to the next `<`, with entities expanded. Under the condensing rule (on by default, see [`tinyxml.h`](tinyxml.h.md)), leading whitespace is skipped and every internal run of whitespace collapses to one space; with condensing off the bytes are taken as they lie.

**Comment** — bytes between `<!--` and `-->`, verbatim, no entity expansion.

**Declaration** — the three optional pseudo-attributes `version`, `encoding` and `standalone`, in any order, each read as a quoted value without entity expansion.

**Unknown** — everything from `<` to the matching `>`, stored raw. The parser makes no attempt to handle a `>` inside a quoted string here, so a doctype containing one truncates. No shipped file does.

## Errors and recovery

**Contract** — there is none. The first failing production aborts the whole parse; the document keeps whatever tree was built up to that point, plus the error identifier, description and position. The engine reads the identifier, forgives exactly one value of it, and otherwise reports the description with the filename.

An embedded zero byte anywhere in the input is its own error identifier, because the whole parser walks a terminated buffer and a zero byte is indistinguishable from the end.

## What a rebuild must reproduce

A standard-conforming parser is *not* a drop-in replacement. The shipped data requires all of:

1. Default to **byte-per-character** decoding when there is no mark and no declaration.
2. Accept **any byte at or above 128 as a name character**.
3. Pass an **unrecognized entity through as literal text**, ampersand included.
4. Accept an **unquoted attribute value**.
5. Condense whitespace runs in text by default.
6. Distinguish the **missing-end-tag** failure from every other failure, so it can be forgiven.
7. Treat a **stray byte-order mark as whitespace** anywhere it appears.
