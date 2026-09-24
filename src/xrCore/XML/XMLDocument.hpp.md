# src/xrCore/XML/XMLDocument.hpp

> Declares the engine's view of an XML file: a parsed tree plus path-addressed readers with defaults.

**Needs** — [`XMLDocument.cpp`](XMLDocument.cpp.md) · [`tinyxml.h`](tinyxml.h.md) · [`xrstring.h`](../xrstring.h.md)
**Used by** — [`XMLDocument.cpp`](XMLDocument.cpp.md) · [`StringTable.cpp`](../../xrEngine/StringTable/StringTable.cpp.md) · [`xrUIXmlParser.cpp`](../../xrUICore/XML/xrUIXmlParser.cpp.md) · [`xrUIXmlParser.h`](../../xrUICore/XML/xrUIXmlParser.h.md) · [`ui_base.cpp`](../../xrUICore/ui_base.cpp.md) · [`ui_styles.cpp`](../../xrUICore/ui_styles.cpp.md)
**Tier floor** — T2: a document object model and string-keyed navigation.

## Purpose

Declares the surface implemented in [`XMLDocument.cpp`](XMLDocument.cpp.md). The user-interface layout files and the localized string tables are XML (system requirements §5), and this is the only way the engine reads them.

Two constants declared here are part of the data contract: the configuration directory the loader resolves relative paths against, and the *interface directory* — a mutable global that a localization or a screen-resolution variant overrides so that the same logical path resolves to a different set of layout files. Both the default and the override are consulted, in that order, when resolving an include.

## Exported units

- **`XMLDocument`** — one parsed file: the tree, its name, an optional *local root* that re-bases every subsequent path lookup, and a flag admitting one specific malformedness.
- **`XMLDocument.Load`** — parse a file, in three forms differing only in how many directory components are supplied and whether a second directory is tried on failure.
- **`XMLDocument.Set`** — parse text the caller already has. Does **not** honour include directives — those are resolved during file loading, not during parsing.
- **`XMLDocument.IsErrored`**, **`GetErrorDesc`** — whether the parse failed and the description.
- **`XMLDocument.Read`, `ReadInt`, `ReadFlt`** — a node's text content as text, integer or real, with a default when the node or the text is absent. Each comes in three forms: by path from the document root, by path from a given node, and from a node directly.
- **`XMLDocument.ReadAttrib`, `ReadAttribInt`, `ReadAttribFlt`** — the same three forms for an attribute.
- **`XMLDocument.NavigateToNode`** — resolve a colon-separated path to a node, with an index selecting among same-named siblings.
- **`XMLDocument.NavigateToNodeWithAttribute`** — the first child with a given tag whose named attribute equals a value.
- **`XMLDocument.SearchForAttribute`** — the same search, recursive.
- **`XMLDocument.GetNodesNum`** — how many children carry a given tag, optionally excluding comments.
- **`XMLDocument.CheckUniqueAttrib`** — the first duplicated attribute value among same-named children, or nothing. A data-validation helper.
- **`XMLDocument.SetLocalRoot`, `GetLocalRoot`, `GetRoot`** — the re-basing hook and the two roots.
- **`XMLDocument.IgnoreMissingEndTagError`** — admit a document whose only fault is an unclosed tag; see [`XMLDocument.cpp`](XMLDocument.cpp.md).
- **`XMLDocument.correct_file_name`** — an overridable hook letting a subclass rewrite a filename before it is resolved. The base returns the name unchanged; the localization layer uses it to pick a language-specific variant.
