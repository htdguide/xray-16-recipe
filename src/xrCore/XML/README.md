# src/xrCore/XML — the second configuration format

Part of chapter 6, [`src/xrCore`](../README.md). Eight files: an adopted lightweight XML
parser, and the engine's own layer on top of it.

## What this module is responsible for

The engine has **two** configuration formats and they are not interchangeable. The INI-like
`ltx` format ([`../xr_ini.cpp`](../xr_ini.cpp.md)) carries every tunable number: weapon
damage, AI parameters, graphics presets. XML carries the two things `ltx` cannot express —
**trees and localized text**: the user-interface window hierarchies with per-element geometry,
textures, fonts and handler names, and the per-language string tables.

Both are frozen on the read side
([`SYSTEM-REQUIREMENTS.md` §5](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)).

The directory owns a complete small XML parser and document model — adopted, not written here
— plus one engine-specific layer that does three things the parser does not: **flattens
include directives** before parsing, **addresses nodes by a colon-separated path**, and
**returns a default for every miss** so that a malformed or older data file degrades rather
than failing.

## Where it sits

It rests on the virtual filesystem ([`../LocatorAPI.h`](../LocatorAPI.h.md)) for reading and
on the string interner. Chapter 15's widget construction and chapter 23's string tables are
the consumers; the parser itself is consumed by nothing else.

## The load-bearing ideas

**Includes are flattened before parsing, not resolved during it.** The engine's layer reads a
file, textually substitutes every include directive, and hands one image to the parser. That
means an include may appear anywhere a byte may, including mid-element, and the parser never
learns includes exist. It also means a cycle is an infinite expansion rather than an error.

**Every read has a default and no read fails.** Asking for a path that is not there yields the
caller's default. That is what makes the shipped data — which spans three games with different
schemas — loadable by one engine, and it is also why a typo in a data file is invisible. A
rebuild that reports misses will find real bugs and will also refuse to load the shipped data;
the twin says where the line is.

**A node is addressed by a colon-separated path with an occurrence index.** Not XPath, not a
walk: a small fixed grammar that the widget loader is written against.

**The parser's encoding decision is its own and is lenient.** It sniffs a byte-order mark,
honours a declaration, and otherwise guesses — and its guess is what the shipped single-byte
localizations depend on. [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md) records every leniency
the data requires; none of them is optional.

**The parser's private string type exists for one reason: growth.** A parser that appends
character by character is quadratic with a naive string. The growth rule in
[`tinystr.cpp`](tinystr.cpp.md) is the only interesting thing about the type, and a rebuild
with a decent string deletes both files.

## The twins

| File | Role |
|---|---|
| [`XMLDocument.hpp`](XMLDocument.hpp.md) | Declares the engine's view of an XML file: a parsed tree plus path-addressed readers with defaults. |
| [`XMLDocument.cpp`](XMLDocument.cpp.md) | **Include flattening, parsing, and colon-path queries with a default for every miss.** The engine's own layer. Substantive. |
| [`tinyxml.h`](tinyxml.h.md) | **The document model**: six node kinds, an attribute list, an encoding decision, a fixed error vocabulary. Substantive. |
| [`tinyxml.cpp`](tinyxml.cpp.md) | The tree: linking children, walking siblings by name, reading attributes. |
| [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md) | **The grammar**: bytes to tree, the encoding decision, the entity table, and every leniency the shipped data needs. Substantive. |
| [`tinyxmlerror.cpp`](tinyxmlerror.cpp.md) | The error identifiers' text, in one table, separated so it could be translated. |
| [`tinystr.h`](tinystr.h.md) | The parser's private string: one pointer, with length, capacity and bytes behind it. |
| [`tinystr.cpp`](tinystr.cpp.md) | The three operations that allocate, and the growth rule that keeps repeated appends from being quadratic. |
