# src/xrUICore/XML/xrUIXmlParser.h

> Declares the layout-flavoured XML document whose one behaviour lives in [`xrUIXmlParser.cpp`](xrUIXmlParser.cpp.md).

**Needs** — [`xrUIXmlParser.cpp`](xrUIXmlParser.cpp.md) · [`xrCore/XML/XMLDocument.hpp`](../../xrCore/XML/XMLDocument.hpp.md)
**Used by** — [`PhraseScript.cpp`](../../xrGame/PhraseScript.cpp.md) · [`ScriptXMLInit.cpp`](../../xrGame/ScriptXMLInit.cpp.md) · [`ScriptXMLInit.h`](../../xrGame/ScriptXMLInit.h.md) · [`UIGameDM.cpp`](../../xrGame/UIGameDM.cpp.md) · [`encyclopedia_article.cpp`](../../xrGame/encyclopedia_article.cpp.md) · [`ArtefactDetectorUI.cpp`](../../xrGame/ui/ArtefactDetectorUI.cpp.md) · [`MMSound.cpp`](../../xrGame/ui/MMSound.cpp.md) · [`UIKeyBinding.cpp`](../../xrGame/ui/UIKeyBinding.cpp.md) · [`UINewsItemWnd.h`](../../xrGame/ui/UINewsItemWnd.h.md) · [`xrGame.cpp`](../../xrGame/xrGame.cpp.md) · [`character_info.cpp`](../../xrServerEntities/character_info.cpp.md) · [`xml_str_id_loader.h`](../../xrServerEntities/xml_str_id_loader.h.md) · [`UIBtnHint.cpp`](../Buttons/UIBtnHint.cpp.md) · [`UIMessageBox.cpp`](../MessageBox/UIMessageBox.cpp.md) · _and 5 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`xrUIXmlParser.cpp`](xrUIXmlParser.cpp.md). It adds exactly
one behaviour to the engine's document type — widescreen filename resolution — and nothing
else. Every layout reader in the chapter takes this type rather than the base document, which
is how the rule is guaranteed to apply: you cannot open a layout through a document that does
not know about it.

That a whole type exists for one string rewrite is worth noting to a rebuilder: the rewrite
could equally be a parameter to the open call, and then this file disappears.

## Exported units

- `CUIXml` — an XML document with layout-aware filename resolution
- `correct_file_name(path, name)` — the resolution hook, overridable again by a further
  subclass (the game layer does not, in practice)
