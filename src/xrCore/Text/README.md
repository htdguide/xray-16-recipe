# src/xrCore/Text — getting shipped text into characters

Part of chapter 6, [`src/xrCore`](../README.md). Two files, and one of the sharper
compatibility hazards in the whole recipe.

## What this module is responsible for

The game's text ships in XML files encoded in **one of several single-byte codepages
depending on the localization** — not in a portable encoding
([`SYSTEM-REQUIREMENTS.md` §4](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)). This
directory turns those bytes into the engine's internal 16-bit characters, and back.

It also owns the character classes the user-interface line breaker consults: which characters
may end a line, which may not begin one, which are followed by a space, and which form a word
that must not be split. Those classes are what make East Asian localizations lay out
correctly without a word dictionary, and they are the only substance in the header.

## Where it sits

It rests on the scalar types and nothing else. Chapter 15's text layout consumes the
character classes; chapter 23's string tables consume the decoding.

## The load-bearing ideas

**The engine's wide character is pinned at 16 bits and is not the host language's.** The
layout code indexes arrays of them, so the width is a layout decision, not a portability
convenience.

**The decode falls back to treating bytes as characters.** When the codepage decode fails,
the bytes are reinterpreted one-for-one rather than rejected — because the shipped text is
*not all one encoding*, and rejecting would drop strings the original engine displays. This
is a leniency the recipe must keep: a strict decoder produces a game with missing text.

**Decoding records where each character started.** The line breaker needs to map a position
in the decoded text back to a byte offset in the source, so the conversion optionally yields
that index alongside the characters. A rebuild whose strings carry their own indexing gets
this for free and can drop the second output.

**4096 characters is the ceiling** on a single converted string, and therefore on a single
localized string the interface can display.

## The twins

| File | Role |
|---|---|
| [`StringConversion.hpp`](StringConversion.hpp.md) | Declares the conversions, and carries **the four character classes** the line breaker asks about. Substantive. |
| [`StringConversion.cpp`](StringConversion.cpp.md) | **The decode**, the per-character start index it records, and the byte-for-byte fallback the shipped text requires. Substantive. |
