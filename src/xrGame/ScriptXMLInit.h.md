# src/xrGame/ScriptXMLInit.h

> Declares the script-visible widget factory implemented in [`ScriptXMLInit.cpp`](ScriptXMLInit.cpp.md).

**Needs** — [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ScriptXMLInit.cpp`](ScriptXMLInit.cpp.md) · [`ui_export_script.cpp`](ui_export_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CScriptXmlInit`, the object a Lua script holds to turn nodes of a UI layout
document into live widgets. It owns one parsed document. Substance is in
[`ScriptXMLInit.cpp`](ScriptXMLInit.cpp.md).

Exported units:

- `CScriptXmlInit` — the factory.
- `ParseFile` — load a layout document by name, through the localization fallback chain.
- `ParseShTexInfo` — register a texture-atlas description globally.
- `InitWindow` — apply a node to an existing window, at a chosen occurrence index.
- One factory per widget type: `InitHint`, `InitFrame`, `InitFrameLine`, `InitEditBox`,
  `InitStatic`, `InitAnimStatic`, `InitSleepStatic`, `InitTextWnd`, `InitCheck`,
  `InitSpinNum`, `InitSpinFlt`, `InitSpinText`, `InitComboBox`, `Init3tButton`, `InitTab`,
  `InitServerList`, `InitMapList`, `InitMapInfo`, `InitTrackBar`, `InitCDkey`,
  `InitMPPlayerName`, `InitMMShniaga`, `InitKeyBinding`, `InitScrollView`, `InitListWnd`,
  `InitListBox`, `InitProgressBar`.
- `GetXml` — the parsed document, shared so that engine and script code can build parts of
  the same screen.
- `script_register` — declares all of the above to Lua, under script names that do not
  always match the internal ones.

## Notes

The declaration names roughly thirty widget classes it never touches, purely to avoid
including their headers. That is an artifact of compilation, not a dependency; a rebuild's
factory depends on the widget types it constructs and on nothing else.
