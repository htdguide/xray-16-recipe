# src/xrCore/XML/tinyxmlerror.cpp

> The error identifiers' human-readable text, in one table, separated so it could be translated.

**Needs** — [`tinyxml.h`](tinyxml.h.md)
**Used by** — [`tinyxml.h`](tinyxml.h.md)
**Tier floor** — T4: a constant table of strings.

## Purpose

Maps each parse error identifier to a description. It is a separate file for one stated reason — so that the strings could be swapped for a localized set without touching the parser — and that reason never materialized: the table is English and there is no mechanism to replace it.

## The table

One string per identifier declared in [`tinyxml.h`](tinyxml.h.md), in the same order, indexed directly by the identifier's value. The engine surfaces these verbatim in its parse-failure diagnostic alongside the filename, so they are what a modder sees when a layout file is broken.

**Notes** — the correspondence between the enumeration's order and this table's order is unchecked. Inserting an identifier without inserting a string shifts every message after it, producing a confidently wrong diagnostic. A rebuild should attach the text to the identifier rather than to its position.

The entry for the identifier the engine forgives reads simply "Error reading end tag", which is the string a rebuilder will see while working out why a shipped file loads anyway — see [`XMLDocument.cpp`](XMLDocument.cpp.md).
