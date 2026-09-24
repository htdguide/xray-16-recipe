# src/xrEngine/Properties.h

> A self-describing stream of named, typed editor properties — a tag-length-value dialect used by the shader and effect tools.

**Needs** — [`xrCore/FS.h`](../xrCore/FS.h.md)
**Used by** — [`Blender.h`](../Layers/xrRender/Blender.h.md)
**Tier floor** — T1: a frozen on-disk record layout read by fixed-size copies, with padding that is part of the format

## Purpose

The material and effect editors need to write a parameter block whose *shape* is not known
to the reader in advance — one blender has three floats and a token, another has a class
identifier and two textures. This file defines the stream dialect for that: a sequence of
(type, name, payload) records where the payload's size is implied by the type, plus the
per-type payload records themselves.

It is a header with no implementation because the whole dialect is a handful of record
layouts and four inline read/write helpers. It is declared in the engine rather than in an
editor because the runtime reads these blocks when loading material descriptions.

**Frozen (read side).** The layouts are what shipped tooling wrote.

## State

The stream. Every entry is the same three-part shape:

```text
ENTRY
  type    : int (32-bit)     # one of the property kinds below
  name    : stringZ
  payload : bytes            # length implied by type; absent for name-only kinds
```

The property kinds, in their frozen numeric order:

```text
ENUM PropertyKind
  MARKER = 0        # section boundary; no payload
  MATRIX            # name only: the value lives elsewhere, this records the binding
  CONSTANT          # name only
  TEXTURE           # name only
  INTEGER           # payload: Integer
  FLOAT             # payload: Float
  BOOL              # payload: Bool
  TOKEN             # payload: Token, then Count entries of (id, 64-byte name)
  CLSID             # payload: ClassId, then Count class identifiers
  OBJECT            # name only
  STRING            # name only
  MARKER_TEMPLATE   # section boundary carrying a type and a count limit
```

Four kinds carry only a name: the editor records *that this parameter exists and what it
is called*, and the value is resolved elsewhere — a texture by name through the resource
system, a matrix or constant by name through the renderer's binding table. That is why
the comment in the source says "really only name is written".

The payload records. Each numeric property carries its authored value together with the
bounds the editor enforced, because the bounds are authored data, not code:

```text
RECORD Integer   value : int,  min : int  (default 0),  max : int  (default 255)
RECORD Float     value : real, min : real (default 0),  max : real (default 1)
RECORD Bool      value : bool                      # stored as a 32-bit word
RECORD Token     selected_id : int, count : int    # followed by count (id, name[64]) pairs
RECORD ClassId   selected : class_id, count : int  # followed by count class identifiers
RECORD Template  type : int, limit : int           # a repeatable section and how many may repeat
```

**Invariants** — the whole stream is packed on a 4-byte boundary; the token name field is
a fixed 64 bytes including its terminator, not a length-prefixed string. Both are part of
the format and a rebuild that packs differently reads garbage.

## Writing

**Contract** — one call emits type, name and an optional fixed-size payload. A marker is
the same call with no payload. The writer never emits the variable-length tails itself —
the caller that wrote a token or class-identifier header is responsible for the entries
that follow it.

```text
FUNCTION write_entry(stream, kind, name, payload : optional<bytes>)
  stream.write int (32-bit) kind
  stream.write stringZ name
  IF payload IS present
    stream.write payload
```

## Reading

**Contract** — reading is *type-checked but not type-driven*: the reader asserts the kind
it expected and then copies exactly the payload record it expected. There is no way to
skip an entry of an unknown kind, because the payload length is not in the stream. The
consequence is that reader and writer must agree on the property order, and a stream
cannot be extended without breaking old readers.

```text
FUNCTION read_kind(stream) -> int
  kind = stream.read int (32-bit)
  stream.skip stringZ               # the name is not returned; the caller knows it
  RETURN kind

FUNCTION read_entry(stream, expected_kind, out payload)
  ASSERT read_kind(stream) == expected_kind
  stream.read payload                       # fixed size for this kind
  IF expected_kind == TOKEN
    stream.skip payload.count * size of (id, name[64])
  ELSE IF expected_kind == CLSID
    stream.skip payload.count * size of class_id
```

**Notes** — the name is read and discarded. It exists in the stream for the editor's
benefit (and for a human reading a hex dump); the runtime identifies a property by its
position in the expected sequence. That is a design decision worth reversing in a rebuild —
keying by name would make the format extensible — but only if the tools are rebuilt too,
since the shipped blocks must still be read positionally.
