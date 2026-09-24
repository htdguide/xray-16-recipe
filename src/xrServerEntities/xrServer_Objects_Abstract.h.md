# src/xrServerEntities/xrServer_Objects_Abstract.h

> The interfaces an entity record must satisfy to be placed, drawn in the editor, shaped, skinned and animated — and the two mixins that give a record a visual and a motion.

**Needs** — [`xrServer_Space.h`](xrServer_Space.h.md) · [`ShapeData.h`](ShapeData.h.md) · [`gametype_chooser.h`](gametype_chooser.h.md) · [`xrEProps.h`](xrEProps.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [`Include/xrRender/DrawUtils.h`](../Include/xrRender/DrawUtils.h.md)
**Used by** — [`gametype_chooser.cpp`](gametype_chooser.cpp.md) · [`xrServer_Object_Base.h`](xrServer_Object_Base.h.md) · [`xrServer_Objects_Abstract.cpp`](xrServer_Objects_Abstract.cpp.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md)
**Tier floor** — T2: an interface list; nothing here touches bytes directly.

## Purpose

Three abstract interfaces and two mixins. The interfaces are the contract a rebuild must
satisfy for *anything* that wants to be an entity record; the mixins are the two most common
optional capabilities — having a renderable model, and having a named animation clip. The
file exists separately from the base record because the editor links against these
interfaces without linking against the records.

## `IServerEntity`

**Contract** — what an entity record owes its two consumers, the level editor and the level
loader.

```text
INTERFACE IServerEntity
  spawn_write(packet, is_local)          # emit the record
  spawn_read(packet) -> bool             # parse it back

  name() -> text                         # the configuration section
  set_name(text)
  name_replace() -> text                 # the designer's instance name
  set_name_replace(text)

  position() -> vector (mutable)
  angle()    -> vector (mutable)
  flags()    -> int (16-bit, mutable)

  shape()  -> optional<IServerEntityShape>
  visual() -> optional<Visual>
  motion() -> optional<Motion>

  validate() -> bool                     # is this record internally coherent
```

**Invariants** — the accessors return *mutable* references because the editor's property
grid writes through them. The capability accessors return nothing when the record does not
have that capability; a caller must not assume.

**Notes** — the interface also carries an editor-change flag set (properties changed, visual
changed, animation changed, motion changed, animation-pause changed) that an editor polls to
know what to rebuild. Those flags are not serialized and exist only while a tool is running.

**The tools-versus-game split lives here.** Everything editor-facing — the property-grid
fill, the debug render callback, the visual collection a tool draws — is compiled out of the
shipping build. A rebuild should make this a compile-time or link-time capability rather
than reproducing the macro: the same record type is compiled into the game with a narrow
surface and into the tools with a wide one, which is the cycle noted in the build order.

## `IServerEntityShape`

**Contract** — a record that occupies a volume rather than a point. The only demand is that
it accept an array of shape definitions (spheres in a local frame, or oriented boxes) as one
assignment. Used by restrictors, zones, climbables and smart covers.

## `IServerEntityLEOwner`

**Contract** — the *other* direction: what the editor owes a record that wants to draw
itself. One demand — resolve a bone name on this record's model to a transform — which is
what a lamp needs to draw its light at the right place and a smart cover needs to place its
loophole markers.

## `CSE_Visual`

**Contract** — the mixin for "this record has a model". Holds the model path, a startup
animation name and a flag set (currently one flag: the model obstructs navigation). Contracts
and the serialized layout are in
[`xrServer_Objects_Abstract.cpp`](xrServer_Objects_Abstract.cpp.md).

## `CSE_Motion`

**Contract** — the mixin for "this record has one named animation clip", separate from the
visual's startup animation because a helicopter's flight path and a torrid zone's movement
are authored as motions in a shared bank rather than as model animations.

## `visual_data`

**Contract** — a (transform, visual) pair. A record that wants to show more than one model in
the editor — a smart cover shows one posed figure per loophole — returns an array of these.
The packing is declared tight because the array is handed across a module boundary to the
editor.
