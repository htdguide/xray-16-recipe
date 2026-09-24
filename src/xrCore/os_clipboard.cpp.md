# src/xrCore/os_clipboard.cpp

> Clipboard read, write and append, with the codepage translation the engine's single-byte text needs to survive a round trip through a Unicode clipboard.

**Needs** — [`os_clipboard.h`](os_clipboard.h.md) · [`Text/StringConversion.hpp`](Text/StringConversion.hpp.md) · [`log.h`](log.h.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`os_clipboard.h`](os_clipboard.h.md)
**Tier floor** — T2: it translates between the engine's single-byte text and the clipboard's Unicode, and the translation is the whole content.

## Purpose

Two callers need the clipboard and they need opposite things. The console needs paste, so a player can put a command in. The failure path needs copy and append, so a crash report reaches a bug tracker without anyone finding the log file. Both cross the same boundary: **the engine's text is single-byte in one of several codepages, and the clipboard is Unicode.**

## State

Stateless, except for one locale object cached per function — the platform's current locale, which is what decides which codepage the engine's bytes are in.

## `copy_to_clipboard`

**Contract** — Replaces the clipboard's contents with the given text. Translates from the current locale's codepage to Unicode unless told the text is already Unicode. On failure, reports it and **logs the text instead**, so the content is not lost.

**Notes** — The fallback is the useful decision: a copy that fails in the failure path would otherwise silently lose the crash report.

## `paste_from_clipboard`

**Contract** — Reads the clipboard into a caller-supplied buffer of stated size, translating Unicode back into the current locale's codepage, truncating to fit. Does nothing when the clipboard holds no text. Then **sanitizes**: every character that is not printable in the current locale is replaced by a space, and tabs and line breaks are replaced by spaces too.

**Invariants** — One character is exempted from the printability test by its byte value. The source names it: the Cyrillic letter that lands on the byte the test would otherwise reject. This is the shape of the problem — the printability test is locale-dependent and the engine's text is not reliably in the locale the platform reports — and the exemption is a patch over it, not a rule. A rebuild working in Unicode throughout deletes the whole sanitization except the tab and line-break replacement, which exists because the console is a single-line field.

## `update_clipboard`

**Contract** — Appends to the clipboard rather than replacing it: reads what is there, concatenates the new text, writes the result back. When the clipboard is empty, degrades to a plain copy. A null argument is logged and ignored.

**Invariants** — The existing content is already Unicode and the new text is not, so the new text is translated and the two are joined *after* translation — joining before it would translate the existing content a second time. The joined result is then written with the "already Unicode" flag set.

**Notes** — This is how the failure path accumulates a stack trace line by line onto the clipboard. It is quadratic in the number of appends, which is acceptable for a few dozen stack frames and would not be for anything else.
