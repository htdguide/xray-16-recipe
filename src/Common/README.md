# src/Common — chapter 1 of the build order

Nothing in this directory depends on anything else in the repository. It is the floor: the
build-time decisions about what machine the engine is being built for, the vocabulary that
the entity layer's serialization is written in, the binary shapes of a compiled level, and
one vendored geometry tool. Everything in chapters 2 through 29 rests on some part of it.

## What this module is responsible for

Four unrelated responsibilities share this directory, and the sharing is historical rather
than principled. A rebuild is free to split them into four modules, and should:

1. **Build-time target detection and the per-platform fill-in.** Which operating system,
   architecture and compiler, what the compiler must be asked for (inlining, alignment,
   symbol visibility, a debugger trap), and one fill-in per platform family supplying the
   names the rest of the engine uses without ever branching on platform. In a language with
   one standard library across its targets, almost all of this disappears.

2. **The object broker** — five recursive traversals (save, load, clone, destroy, compare)
   that operate on *any* value by walking its structure. This is the serialization contract
   the whole entity layer rests on, and it is the part of this chapter a rebuilder must
   read carefully, because it is where the on-disk and on-wire format of every entity
   actually comes from.

3. **Level and identity structures** — the chunk vocabulary of a compiled level, the
   navigation-mesh node record in seven frozen generations, the vertex layouts of static
   geometry, the 128-bit build identity, and a portable restatement of the retired graphics
   enumerations that the shipped material files are written in.

4. **A vendored mesh tangent generator** and its bridge to the engine's mesh types — see
   [`NvMender2003/`](NvMender2003/README.md).

## Where it sits

First. Nothing here refers forward. It is depended on by every later chapter, most heavily
by [`xrServerEntities`](../xrServerEntities/README.md) and
[`xrGame`](../xrGame/README.md) (the broker), the level loader in
[`xrEngine`](../xrEngine/README.md) and the navigation graph in
[`xrAICore`](../xrAICore/README.md) (the level structures), and every translation unit in
the project (the platform layer).

The one thing this chapter reaches *upward* for is the global service-locator struct from
[chapter 5](../Layers/xrAPI/README.md), and only because the universal prelude
[`Common.hpp`](Common.hpp.md) names it. That is a packaging artifact, not a dependency.

## The load-bearing ideas

Name these once here so the twins can be terse.

### The broker's dispatch ladder

All five traversals ask the same questions about a value in the same order, and the **first
match wins**:

| # | The value is… | …and is treated as |
|---|---|---|
| 1 | text, a pair, a fixed-capacity vector, a vector of truth values, a queue, a stack or a priority queue | a hand-written special case |
| 2 | a **container** | walked element-wise |
| 3 | a **pointer** | followed; the pointee is walked |
| 4 | a value signing the **persistence contract** | asked to do it itself |
| 5 | anything else | a plain value: its memory image, or nothing |

The order carries the decisions. Containers before pointers, so a container of pointers is
walked rather than mistaken for one. Pointers before the persistence contract, so an object
reached through a pointer is still asked. The contract before the memory image, so a type
that knows its own encoding always wins over the byte dump.

**The container test is structural, not nominal.** A type is a container if it publishes
four member type names, and nothing registers. That means the broker works on containers
nobody anticipated — and that a type which happens to publish those names is silently
treated as one. See [`object_type_traits.h`](object_type_traits.h.md).

### Ownership is encoded in the writability of a text reference

Throughout the broker, and throughout the entity layer that uses it:

- **a read-only text reference is borrowed** — never freed, shared on clone, and cannot be
  loaded into at all;
- **a writable text buffer is owned** — freed on destroy, duplicated on clone, allocated on
  load;
- **the reference-counted shared string type is shared** — another reference is taken, never
  a copy, because the resource system compares those by identity.

Most languages do not encode this distinction in a type. A rebuild must carry it
explicitly, and every entity field that holds text has been declared with it in mind.

### There is no schema

The broker derives the byte layout of a record from its **field declaration order**. No
type tags, no field names, no lengths, no versions appear in the stream. A reader must know
the exact type it is reading, which is why the save format is version-tagged at its outer
layer and refuses a mismatch rather than guessing, and why an entity that changes its
fields changes the format. This is the single most consequential fact in the chapter.

The one shape that carries framing is the **container**: a 32-bit element count, then the
elements. That count is always 32 bits regardless of the container's own size type, and is
written even when the container is empty.

### The level data is a memory image

Level headers and navigation nodes are read by pointing a record at a mapped file region.
Little-endian, no byte swapping, unaligned fields addressed by byte offset and bit shift.
Every byte layout in [`LevelStructure.hpp`](LevelStructure.hpp.md) and
[`OGF_GContainer_Vertices.hpp`](OGF_GContainer_Vertices.hpp.md) is
[frozen](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).

### The engine's canonical path separator is the backslash

Because the game data was authored on Windows and the archives contain backslashes.
Conversion to the host's separator happens at the moment of a system call and nowhere else.
Case folding is ASCII-only and its direction is inverted between case-insensitive and
case-sensitive filesystems — see [`PlatformLinux.inl`](PlatformLinux.inl.md).

## The twins

| File | Role |
|---|---|
| [`Common.hpp`](Common.hpp.md) | The universal prelude: the four things every translation unit needs in scope |
| [`Config.hpp`](Config.hpp.md) | Build-time feature switches that change the script-visible surface |
| [`Platform.hpp`](Platform.hpp.md) | Classifies the target — OS, architecture, compiler, configuration — and selects a fill-in |
| [`Compiler.inl`](Compiler.inl.md) | What the engine asks of its compiler: inlining, alignment, visibility, the debugger trap |
| [`PlatformWindows.inl`](PlatformWindows.inl.md) | Windows fill-in: the shortest, since its conventions are the engine's own |
| [`PlatformLinux.inl`](PlatformLinux.inl.md) | Linux and Haiku fill-in: the substantive POSIX one — path conversion, case folding, complete reads, bounded strings |
| [`PlatformApple.inl`](PlatformApple.inl.md) | macOS fill-in: the Linux one, three headers apart |
| [`PlatformBSD.inl`](PlatformBSD.inl.md) | BSD fill-in: the Linux one, two headers apart |
| [`FSMacros.hpp`](FSMacros.hpp.md) | The logical root names paths are written against, and the canonical separator |
| [`Util.hpp`](Util.hpp.md) | Enumerations as bit sets; releasing a reference-counted device handle exactly once |
| [`Noncopyable.hpp`](Noncopyable.hpp.md) | The marker for types that have identity rather than value |
| [`object_broker.h`](object_broker.h.md) | One name for the whole broker family |
| [`object_interfaces.h`](object_interfaces.h.md) | The four contracts an entity signs: destroyable, serializable, replicated, scheduled |
| [`object_type_traits.h`](object_type_traits.h.md) | The compile-time questions the broker asks about a type |
| [`object_saver.h`](object_saver.h.md) | The encoder — and therefore the definition of the entity byte format |
| [`object_loader.h`](object_loader.h.md) | The decoder, plus allocation and merge policy |
| [`object_cloner.h`](object_cloner.h.md) | Deep copy: duplicate what is owned, share what is borrowed |
| [`object_destroyer.h`](object_destroyer.h.md) | Recursive teardown, honouring the same ownership rule |
| [`object_comparer.h`](object_comparer.h.md) | Structural comparison under a caller-chosen relation |
| [`LevelStructure.hpp`](LevelStructure.hpp.md) | Level chunk vocabulary and the navigation node record in seven generations |
| [`LevelGameDef.h`](LevelGameDef.h.md) | Chunk identifiers and layouts for authored spawn points, weather volumes and patrol paths |
| [`OGF_GContainer_Vertices.hpp`](OGF_GContainer_Vertices.hpp.md) | Static-geometry vertex layouts and the quantization that produces them |
| [`GUID.hpp`](GUID.hpp.md) | The 128-bit identity stamped into every output of one level build |
| [`d3d9compat.hpp`](d3d9compat.hpp.md) | The retired graphics interface's enumerations, restated portably — frozen by shipped material files |
| [`_d3d_extensions.h`](_d3d_extensions.h.md) | The engine's light and surface-material records, at their historical byte sizes |
| [`face_smoth_flags.h`](face_smoth_flags.h.md) | Per-triangle edge-hardness and winding bits, and when two faces may smooth together |
| [`NvMender2003/`](NvMender2003/README.md) | Vendored mesh tangent-basis generator and its bridge (5 files) |

## What a rebuild should do differently

- Split the four responsibilities into four modules. Nothing connects them.
- Publish the broker's dispatch ladder as an explicit, ordered specification rather than
  leaving it implicit in overload resolution.
- Unify the two disagreeing tests for "is this container ordered or a sequence" — the
  cloner and the loader use different ones.
- Give the encoder's element filter and the decoder's element filter the same semantics, or
  remove the encoder's: filtering on the encode side writes a count that no longer matches
  the elements that follow.
- Collapse the three identical POSIX fill-ins into one.
