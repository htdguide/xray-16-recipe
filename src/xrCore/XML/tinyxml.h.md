# src/xrCore/XML/tinyxml.h

> The document object model the engine's XML rests on: six node kinds, an attribute list, an encoding decision, and a fixed error vocabulary.

**Needs** — [`tinyxml.cpp`](tinyxml.cpp.md) · [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md) · [`tinyxmlerror.cpp`](tinyxmlerror.cpp.md) · [`tinystr.h`](tinystr.h.md) · [`xrMemory.h`](../xrMemory.h.md)
**Used by** — [`XMLDocument.cpp`](XMLDocument.cpp.md) · [`XMLDocument.hpp`](XMLDocument.hpp.md) · [`tinyxml.cpp`](tinyxml.cpp.md) · [`tinyxmlerror.cpp`](tinyxmlerror.cpp.md) · [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md)
**Tier floor** — T2: a linked tree with parent and sibling links, and a byte-level encoding decision at parse time.

## Purpose

A vendored XML library, adapted to allocate through the engine's allocator and to use the engine's string type. A rebuild should replace it with whatever its language already has — but must reproduce three things this header fixes, because the engine's own code and the shipped data both depend on them:

- the **node kinds** and what each one's "value" means, since [`XMLDocument.cpp`](XMLDocument.cpp.md) navigates by them;
- the **encoding decision**, because the shipped interface files are not all in one encoding;
- the **error identifiers**, because one of them is selectively forgiven.

Everything else here — the visitor, the printing, the streaming operators, the const/non-const pairs — is library surface the engine does not use and a rebuild need not provide.

## The node model

```text
ENUM NodeType
  DOCUMENT      # the whole file; its "value" is the filename
  ELEMENT       # a tag; its "value" is the tag name
  COMMENT       # its "value" is the comment body
  UNKNOWN       # anything the parser does not model — a processing instruction,
                # a doctype, an entity declaration; its "value" is the raw tag body
  TEXT          # character data; its "value" is the text
  DECLARATION   # the leading <?xml ...?>; carries version, encoding, standalone

RECORD Node
  type     : NodeType
  value    : text                 # meaning depends on type, per the list above
  parent   : optional<Node>
  first, last   : optional<Node>  # children
  prev, next    : optional<Node>  # siblings
```

**Invariant** — an element's *text content* is a separate child node of kind TEXT, not a field of the element. That is why the engine's readers fetch a node's first child and ask whether it is text; an element with a leading comment has no readable text by that rule.

**Invariant** — a node belongs to exactly one parent and is owned by it; destroying a node destroys its subtree. Inserting a node that already has a parent is not supported — the library copies instead.

An **element** additionally owns an ordered attribute list, looked up by name with a linear scan. Attribute order is preserved, which matters only for output.

A **declaration** is parsed and preserved but its stated encoding is *not* what decides how the document is decoded — see below.

## The encoding decision

```text
ENUM Encoding
  UNKNOWN     # decide from the byte-order mark, then from the declaration
  UTF8        # force multi-byte decoding
  LEGACY      # force byte-per-character
```

The default is UNKNOWN, and under it the parser decides as follows: a leading three-byte mark forces multi-byte; otherwise the declaration's stated encoding is consulted, and only a name beginning with the multi-byte encoding's name selects it; anything else, including no declaration at all, selects byte-per-character.

**This is load-bearing.** The shipped localized files are in single-byte code pages with no byte-order mark and frequently no declaration, so they must land in the byte-per-character branch, where a high byte passes through untouched and reaches the font atlas as its own index. A rebuild whose parser assumes a single modern encoding will mangle every non-Latin localization. See also [`Text/StringConversion.cpp`](../Text/StringConversion.cpp.md), which makes the same decision the other way round — by failing over to byte-per-character.

Under the multi-byte branch, a numeric character reference is converted to a multi-byte sequence; under the byte branch it is truncated to a single byte. That asymmetry is part of the same decision.

## Entities

Exactly five named entities are recognized, and the set is closed:

| Entity | Character |
|---|---|
| `&amp;` | ampersand |
| `&lt;` | less-than |
| `&gt;` | greater-than |
| `&quot;` | double quote |
| `&apos;` | apostrophe |

Numeric references are recognized in both decimal and hexadecimal form. Any other named entity is **not** an error: it is passed through as literal text, ampersand and all. A rebuild must match that leniency, because the shipped files contain stray ampersands.

## Errors

```text
ENUM ErrorId
  NONE, GENERIC, OPENING_FILE, OUT_OF_MEMORY,
  PARSING_ELEMENT, FAILED_TO_READ_ELEMENT_NAME, READING_ELEMENT_VALUE,
  READING_ATTRIBUTES, PARSING_EMPTY, READING_END_TAG,
  PARSING_UNKNOWN, PARSING_COMMENT, PARSING_DECLARATION,
  DOCUMENT_EMPTY, EMBEDDED_NULL, PARSING_CDATA, DOCUMENT_TOP_ONLY
```

**`READING_END_TAG` is the one the engine selectively forgives** — see [`XMLDocument.cpp`](XMLDocument.cpp.md). A rebuild must be able to distinguish "this document is missing a closing tag" from every other malformedness, or it cannot load the shipped files that are broken in exactly that way. The other identifiers only need to exist as distinct values.

A parse error also records the row and column at which it occurred, counted in characters with a configurable tab width; the engine reports the description but not the position.

## Whitespace

A global flag decides whether runs of whitespace inside text are condensed to a single space. It defaults to **on**, and the engine never changes it. That means the shipped layout files' indentation does not appear in the text the interface displays — which is what makes them readable as source. A rebuild that preserves whitespace will render every caption with its surrounding indentation.

## Exported units

- **`TiXmlNode`** — the tree node above, with child, sibling and typed accessors (`FirstChild`, `FirstChildElement`, `NextSibling`, each with an optional name filter), linking and removal, and the type query.
- **`TiXmlElement`** — a node plus attributes: fetch an attribute by name as text or converted, set one, remove one, and fetch the element's first text child directly.
- **`TiXmlText`**, **`TiXmlComment`**, **`TiXmlUnknown`**, **`TiXmlDeclaration`** — the leaf kinds.
- **`TiXmlDocument`** — the root: parse from text, report the error identifier, description and position, and clear.
- **`TiXmlAttribute`**, **`TiXmlAttributeSet`** — one attribute and the ordered list; lookup is a linear scan by name.
- **`TiXmlVisitor`** — a depth-first callback interface over the tree. Unused by the engine.
- **`TiXmlBase`** — the shared base carrying the entity table, the whitespace flag, the encoding helpers and the error strings.
