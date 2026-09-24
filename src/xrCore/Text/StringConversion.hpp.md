# src/xrCore/Text/StringConversion.hpp

> Declares the text conversions the localization layer needs, plus the character classes the line-breaker asks about.

**Needs** — [`StringConversion.cpp`](StringConversion.cpp.md) · [`xrCore.h`](../xrCore.h.md)
**Used by** — [`StringConversion.cpp`](StringConversion.cpp.md) · [`os_clipboard.cpp`](../os_clipboard.cpp.md) · [`IGameFont.hpp`](../../xrEngine/IGameFont.hpp.md) · [`xr_input.cpp`](../../xrEngine/xr_input.cpp.md)
**Tier floor** — T2: byte-level decoding of a text encoding and classification by code point.

## Purpose

Declares the surface implemented in [`StringConversion.cpp`](StringConversion.cpp.md), and carries inline the character predicates the user-interface text layout uses. Those predicates live in the header because they are called once per character during line breaking and must inline; they are the only substance this header holds.

## Exported units

- **`xr_wide_char`** — the engine's wide character: a 16-bit code unit. It is *not* the language's own wide character type, because that type's width varies by platform and this one is pinned by the layout code that indexes arrays of it.
- **`MAX_MB_CHARS`** — 4096; the largest string the conversion will produce, and therefore the largest single localized string the interface can display.
- **`mbhMulti2Wide`** — decode a byte string into wide characters, optionally recording where each character started in the source; see [`StringConversion.cpp`](StringConversion.cpp.md).
- **`StringFromUTF8`**, **`StringToUTF8`** — convert between the engine's single-byte representation in a given locale and a portable byte encoding.
- **`IsNeedSpaceCharacter`** — a character after which a line may break *and* which is followed by a space in the rendered text: the ordinary space, the ideographic space, and the full-width comma, period, colon, semicolon, exclamation and question marks, plus the ellipsis and the two ideographic stops. This is the set that makes East Asian text lay out correctly without a word dictionary.
- **`IsBadStartCharacter`** — a character that may not begin a line: everything in the set above, plus the half-width comma, period, colon, semicolon, exclamation and question marks and both closing parentheses. A break that would place one of these at the start of a line must be moved back.
- **`IsBadEndCharacter`** — a character that may not end a line: both opening parentheses, and the ideographic digit one. The last is startling and is not explained anywhere in the source; it is the character used as a full-width dash in some of the shipped Chinese text, which is a plausible but unconfirmed reason.
- **`IsAlphaCharacter`** — digits and Latin letters, in both their half-width and full-width forms. A run of these is a word and may not be broken inside.
