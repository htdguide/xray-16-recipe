# src/xrEngine/StringTable/StringTable.h

> Declares the localized string table and the language selection it performs at startup.

**Needs** — [`StringTable.cpp`](StringTable.cpp.md) · [`xrCore/xrstring.h`](../../xrCore/xrstring.h.md) · [`xrCore/xr_token.h`](../../xrCore/xr_token.h.md)
**Used by** — [`GameFont.cpp`](../GameFont.cpp.md) · [`IGame_Persistent.cpp`](../IGame_Persistent.cpp.md) · [`StringTable.cpp`](StringTable.cpp.md) · [`xr_level_controller.cpp`](../xr_level_controller.cpp.md) · [`UIDebugFonts.cpp`](../../xrGame/ui/UIDebugFonts.cpp.md) · [`UIHint.cpp`](../../xrUICore/Hint/UIHint.cpp.md) · [`UILines.cpp`](../../xrUICore/Lines/UILines.cpp.md) · [`UIListBox.cpp`](../../xrUICore/ListBox/UIListBox.cpp.md) · [`UIXmlInitBase.cpp`](../../xrUICore/XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3: a name-to-text map built from XML at startup

## Purpose

Declares the surface implemented in [`StringTable.cpp`](StringTable.cpp.md).

Exported units:

- `CStringTable` — the table. One process-wide instance reached through a free function;
  its data is a separately owned block so the whole table can be dropped and rebuilt when
  the language changes without invalidating the instance.
- `Init` / `Destroy` / `rescan` / `ReloadLanguage` — the lifecycle.
- `translate` — three forms: return the translation or the identifier itself; try two
  identifiers in turn; or report whether a translation existed and write it out.
- `has_translation` — the query without the fallback.
- `GetCurrentLanguage`, `GetCurrentFontPrefix`, `GetCurrency`, `GetLanguagesToken` — what
  the interface layer needs to pick fonts, format money and offer a language menu.
- `LanguageID`, `LanguageIDInLTX` — the selected language's index and the value the
  configuration last declared, kept so a configuration edit can be told apart from a
  console-driven change.

Internal: `STRING_TABLE_DATA` (the language, its font prefix, its currency symbol and the
map), `Load`, `FillLanguageToken`, `SetLanguage`, `ParseLine`.
