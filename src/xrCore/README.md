# src/xrCore — the filesystem, the formats, the strings, the memory, the threads

Chapter 6 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order). Around 190 source
files across a root directory and thirteen subdirectories. After the renderer interfaces this
is the most load-bearing chapter in the recipe, and unlike them it is load-bearing in a way
that is *checkable*: most of what is decided here is a byte layout that already exists on a
retail disc.

## What this module is responsible for

Nine things, which is too many for one module and is a fact about the original rather than a
recommendation. They are listed here in the order a rebuild should tackle them, which is not
the order they appear in the directory.

1. **The chunked binary container** — the recursive `(identifier, size, payload)` framing that
   every non-text engine file is built out of: levels, models, animation banks, spawn files,
   save games. [`FS.cpp`](FS.cpp.md).
2. **The virtual filesystem** — one flat namespace assembled from a configuration file, a tree
   of loose directories and a chain of archives, with logical roots (`$game_data$`, `$level$`
   and about twenty more) standing in for physical paths.
   [`LocatorAPI.cpp`](LocatorAPI.cpp.md).
3. **Four generations of archive format**, because four generations of the game shipped, plus
   the two codecs and the obfuscator they need. [`LocatorAPI.cpp`](LocatorAPI.cpp.md),
   [`LzHuf.cpp`](LzHuf.cpp.md), [`Compression/`](Compression/README.md),
   [`Crypto/`](Crypto/README.md).
4. **The `ltx` configuration format** — the INI-like text with section inheritance and include
   directives that carries every tunable number in the game. [`xr_ini.cpp`](xr_ini.cpp.md).
5. **Interned strings and interned blobs** — the mechanism by which every name in the engine
   is a pointer into one global table, so that comparing two names is comparing two pointers.
   [`xrstring.cpp`](xrstring.cpp.md), [`xrsharedmem.cpp`](xrsharedmem.cpp.md).
6. **The allocator** and everything routed through it. [`xrMemory.cpp`](xrMemory.cpp.md),
   [`Memory/`](Memory/README.md).
7. **Logging, assertion and the crash handler** — the failure path, including the stack walker
   and the crash-report snapshot. [`log.cpp`](log.cpp.md), [`xrDebug.cpp`](xrDebug.cpp.md),
   [`Debug/`](Debug/README.md).
8. **Threading**: the primitives and the work-stealing scheduler every parallel loop in the
   engine runs on. [`Threading/`](Threading/README.md).
9. **The skeletal animation data model** — both the authored form and the quantized runtime
   form, and the conversion between them. [`Animation/`](Animation/README.md).

Plus, scattered through the root directory, the small vocabulary everything else is written
in: the class identifier that a spawn record names an entity kind by, the random generator the
simulation's determinism rests on, the timers, the network packet's byte layout, the
separated-list parser every configuration tuple goes through, and the declarations of the
math types.

What it is *not* responsible for: it draws nothing, simulates nothing, and knows about no
entity. It is also not where the math lives — see below.

## Where it sits

It rests on chapters 1 through 5: the platform and compiler names, the container aliases, the
four-wide float layer, the renderer and editor interfaces it hands buffers to, and the global
environment struct.

There are two structural oddities worth knowing before opening a twin.

**The math types are declared here and implemented in chapter 3.** Every vector, matrix,
quaternion, box, plane and sphere type in the engine is *declared* in this directory's root,
with the short operations inline, while the long bodies live in
[`src/utils/xrMiscMath`](../utils/xrMiscMath/README.md). That is a build-order inversion — a
lower chapter holding a higher one's declarations — and it exists because every file in the
engine includes these headers and pulling chapter 3 in with them would be circular. A rebuild
should put the types and their operations together in one module below this one.

**Chapter 2 reaches forward into this one.** The container aliases in
[`src/xrCommon`](../xrCommon/README.md) need the allocator, the interned-string type and the
numeric constants, all of which are here. The source's own layering wanted those three in
chapter 2, and a rebuild should move them.

Everything from chapter 7 onward consumes this module, usually heavily. Nothing here refers
forward except through those two oddities.

## The load-bearing ideas

The twins are terse because these are stated once, here.

**Everything is a chunk.** `(identifier, size, payload)`, little-endian, no alignment padding,
nested arbitrarily. The high bit of the identifier is a compression mark, not part of the
identity. Find-by-identifier returns the **first** match in file order, and several shipped
files depend on that. This framing is the single most reused decision in the whole engine:
levels, models, animation banks, spawn files, save games and the archives themselves are all
chunk trees. Learn it from [`FS.cpp`](FS.cpp.md) before anything else.

**A file is read by pointing at it, not by copying it.** The readers hand out a pointer *into*
a memory-mapped region and callers place structures over it in place. That is the reason the
whole chapter is T1 and it is why reader lifetime is a correctness concern rather than a
tidiness one: the mapping must outlive every reader over it. The sliding-window reader
([`stream_reader.cpp`](stream_reader.cpp.md)) is the same idea for files too large to map
whole.

**One namespace, assembled from a text file.** Logical roots map to physical paths with flags
for recursion and writability; archives and loose directories merge into one flat registry of
entries. A lookup is a hash and a comparison, not a directory walk.

**Case is folded, and the fold is ASCII-only, everywhere.** The shipped data references files
and sections with inconsistent capitalization, so every lookup folds — but over exactly the
twenty-six unaccented Latin letters. A locale-aware fold changes which files a level matches
and is forbidden ([§4](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)).

**Names are pointers.** Every configuration key, section name, object name, bone name and
texture path is interned: stored once in a global checksum-bucketed table and referred to by a
reference-counted handle. Two names are equal when their pointers are equal, and a name hashes
in one field read. This is why the engine can afford to key everything on text. The same idea
applied to byte arrays is the blob interner, which is how a hundred instances of one creature
share one skinning table.

**The archive formats are frozen and their differences are in the header, not the entries.**
Four generations; the entry layout is identical in all four, and what varies is whether there
is a header chunk, what the entry point is, and whether the directory is obfuscated. The
obfuscation has **no field saying which of two keys was used** — trial and rollback is the only
way to tell. [`LocatorAPI.cpp`](LocatorAPI.cpp.md) has the layouts.

**The configuration parser must accept everything the shipped files contain**, including
duplicated keys, multiple inheritance on a section header, include directives, and values with
embedded commas that are parsed as tuples. It is frozen on the read side and free on the write
side. [`xr_ini.cpp`](xr_ini.cpp.md) and [`xr_trims.cpp`](xr_trims.cpp.md) together.

**An entity kind is an eight-character name packed big-endian into a 64-bit integer.** That
packing is the key in every spawn record and every factory table.
[`clsid.cpp`](clsid.cpp.md) — and note that big-endian here is an exception to the
little-endian rule that governs everything else in the chapter.

**Little-endian, unaligned, no byte-swapping anywhere.** On-disk structures are memory images.
A big-endian port must add swapping at every load site, and there is no existing hook for it.

**There is no allocation failure path.** The allocator either succeeds or terminates the
process, and nothing in the engine checks a result. A rebuild may disagree, but must then add
checks at several thousand call sites.

**The crash report is the product.** There is no test suite
([§6](../../SYSTEM-REQUIREMENTS.md#6-conformance)), so what the log says when the engine dies
is the primary evidence any bug carries. That is why the failure path
([`xrDebug.cpp`](xrDebug.cpp.md), [`Debug/`](Debug/README.md)) gets more care than its size
suggests it should.

## The subdirectories

| Directory | What it owns |
|---|---|
| [`Animation/`](Animation/README.md) | The skeleton and motion data models, authored and runtime, and the quantizer between them. |
| [`Compression/`](Compression/README.md) | Four compressors and the decision about which is used where. |
| [`Containers/`](Containers/README.md) | Two collections the standard ones could not be: a sorted-array map and an arena-backed tree. |
| [`Crypto/`](Crypto/README.md) | The archive obfuscator, and the configuration-dump signature chain. |
| [`Debug/`](Debug/README.md) | The stack walker, and the device error-code decoder. |
| [`Events/`](Events/README.md) | The core-level event bus, with safe self-unsubscription. |
| [`Math/`](Math/README.md) | Two specialized random generators — not the general one. |
| [`Media/`](Media/README.md) | The one in-memory raster image and its two file forms. |
| [`Memory/`](Memory/README.md) | The container allocator adapter, and aligned allocation. |
| [`PostProcess/`](PostProcess/README.md) | The authored screen effect: eleven tracks and the rules for combining two. |
| [`Text/`](Text/README.md) | Decoding the shipped single-byte text, and the line-breaker's character classes. |
| [`Threading/`](Threading/README.md) | The work-stealing scheduler and the primitives under it. |
| [`XML/`](XML/README.md) | The second configuration format: trees and localized text. |

## The twins — the root directory

### The chunked container and the virtual filesystem

| File | Role |
|---|---|
| [`FS.cpp`](FS.cpp.md) | **The chunk format**: framing, nesting, back-patched sizes, the four reader flavours, compression marks. The first thing to read in this chapter. |
| [`FS.h`](FS.h.md) | Declares the reader and writer surface. |
| [`FS_internal.h`](FS_internal.h.md) | The concrete reader and writer flavours, visible only inside the module. |
| [`FS_impl.h`](FS_impl.h.md) | The chunk-lookup strategy, isolated so it can be swapped and measured. |
| [`LocatorAPI.cpp`](LocatorAPI.cpp.md) | **The virtual filesystem and the four archive generations**. The longest page in the chapter. |
| [`LocatorAPI.h`](LocatorAPI.h.md) | Declares the filesystem's surface and the on-disk records behind it. |
| [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md) | The mounting vocabulary: a logical root, a listing entry, the wildcard matcher. |
| [`LocatorAPI_defs.cpp`](LocatorAPI_defs.cpp.md) | Resolving a logical root plus a relative name into one physical path; wildcard matching. |
| [`LocatorAPI_auth.cpp`](LocatorAPI_auth.cpp.md) | Folding the mounted filesystem and loaded configuration into one number two machines can compare. |
| [`stream_reader.cpp`](stream_reader.cpp.md) | **The sliding memory-mapped window**: reading arbitrarily large archive regions without mapping the whole archive. |
| [`stream_reader.h`](stream_reader.h.md) · [`stream_reader_inline.h`](stream_reader_inline.h.md) | Its declaration, and the trivial half — position arithmetic, remap, self-release. |
| [`file_stream_reader.cpp`](file_stream_reader.cpp.md) · [`file_stream_reader.h`](file_stream_reader.h.md) | The variant that owns the file and its mapping rather than borrowing a region. |
| [`FileSystem.cpp`](FileSystem.cpp.md) · [`FileSystem.h`](FileSystem.h.md) · [`FileSystem_borland.cpp`](FileSystem_borland.cpp.md) | Path-string surgery, the tools' file-chooser, and the backup-by-renaming convention. |
| [`ChooseTypes.H`](ChooseTypes.H.md) | The editor's asset-chooser vocabulary: what kinds of thing can be picked, and the events a chooser raises. |
| [`FileCRC32.cpp`](FileCRC32.cpp.md) · [`FileCRC32.h`](FileCRC32.h.md) | Checksumming a text file together with everything it includes, for a shader cache key. |
| [`crc32.cpp`](crc32.cpp.md) | **The standard CRC-32**, plus the separator-insensitive path variant — the hash under the interner, the archive checksums and path lookup. |

### The codecs the formats need

| File | Role |
|---|---|
| [`LzHuf.cpp`](LzHuf.cpp.md) | **The engine's own codec**: LZSS over a 4096-byte ring with an adaptive Huffman symbol coder. Frozen; archive directories and legacy chunk bodies use it. |
| [`lzhuf.h`](lzhuf.h.md) | Declares it. |

### Configuration and text

| File | Role |
|---|---|
| [`xr_ini.cpp`](xr_ini.cpp.md) | **The `ltx` parser and writer**: sections, inheritance, includes, tuples. Frozen on read. |
| [`xr_ini.h`](xr_ini.h.md) | Declares it, plus the typed-read helpers every caller uses. |
| [`xr_trims.cpp`](xr_trims.cpp.md) · [`xr_trims.h`](xr_trims.h.md) | **The separated-list vocabulary**: how a configuration value holding several items is split, counted, indexed and trimmed. Every tuple in the game data passes through here. |
| [`xr_token.cpp`](xr_token.cpp.md) · [`xr_token.h`](xr_token.h.md) | Name/number tables both ways — the mechanism behind every enumerated setting spelled as a word. |
| [`xrstring.cpp`](xrstring.cpp.md) | **The string interner**: one global checksum-bucketed reference-counted table; every name in the engine is a pointer into it. |
| [`xrstring.h`](xrstring.h.md) | Declares the interned record, the handle, and the ASCII string helpers. |
| [`xrsharedmem.cpp`](xrsharedmem.cpp.md) · [`xrsharedmem.h`](xrsharedmem.h.md) | **The blob interner**: the same idea for byte arrays — skinning tables, vertex arrays, animation keys. |
| [`string_concatenations.h`](string_concatenations.h.md) | **Joining without allocating**: bounded concatenation into a caller buffer, and the stack-allocating form the hot path-building code uses. |
| [`string_concatenations_inline.h`](string_concatenations_inline.h.md) · [`string_concatenations.cpp`](string_concatenations.cpp.md) | The measuring half, and the failure side — the oversized-join report and the stack-exhaustion probe. |
| [`_std_extensions.h`](_std_extensions.h.md) | The replacement standard library for scalars and raw text: float validity, branch-free min/max, bounded string operations, the compile-time name hash. |
| [`_std_extensions.cpp`](_std_extensions.cpp.md) | The local-time stamp used in file names. |

### Memory, lifetime and sharing

| File | Role |
|---|---|
| [`xrMemory.cpp`](xrMemory.cpp.md) | **The allocator seam**: one routing point, three interchangeable fillings, and the process-memory queries the out-of-memory path reports. |
| [`xrMemory.h`](xrMemory.h.md) | Declares it, and the object-lifetime helpers used instead of the language's own. |
| [`xrPool.h`](xrPool.h.md) | A free-list allocator for one object type; the freed object's own storage holds the list link. |
| [`xr_resource.h`](xr_resource.h.md) | **The reference-counted resource base**: what it means for a texture, shader, mesh or sound to be shared and destroyed by the last user. |
| [`intrusive_ptr.h`](intrusive_ptr.h.md) | The second sharing scheme — for objects carrying their own count that are not loadable assets. |
| [`xr_shared.h`](xr_shared.h.md) · [`xr_shared.cpp`](xr_shared.cpp.md) | A keyed container of shared values: create-or-find by name, released by the last holder. |
| [`buffer_vector.h`](buffer_vector.h.md) · [`buffer_vector_inline.h`](buffer_vector_inline.h.md) | **A dynamic array over memory somebody else owns** — how per-frame and per-query collections live on the stack. |
| [`FixedVector.h`](FixedVector.h.md) | A sequence with compile-time capacity and no heap, for the short lists the frame loop produces and drops. |

### Failure, logging and process

| File | Role |
|---|---|
| [`xrDebug.cpp`](xrDebug.cpp.md) | **The failure path**: assertion reporting, the fatal dialog and its three outcomes, signal and unhandled-exception installation, stack capture, the crash snapshot. |
| [`xrDebug.h`](xrDebug.h.md) | Declares it, and the two interfaces the failure path needs another module to supply. |
| [`xrDebug_macros.h`](xrDebug_macros.h.md) | **The assertion vocabulary**: which checks survive into a shipping build, which vanish, what each carries into the report. |
| [`log.cpp`](log.cpp.md) | **The log**: an in-memory line buffer that exists before any file does, a file writer attached once the filesystem is up, and a mirror callback for the console. |
| [`log.h`](log.h.md) | Declares it. |
| [`dump_string.cpp`](dump_string.cpp.md) · [`dump_string.h`](dump_string.h.md) | Human-readable renderings of the math types; development builds only. |
| [`xrCore.cpp`](xrCore.cpp.md) | **Process bring-up and teardown**: identity, the build stamp, the order the subsystems come alive in, and the reference count that lets several modules ask for the core. |
| [`xrCore.h`](xrCore.h.md) | The umbrella header: the process-wide core object plus everything a consumer is expected to have. |
| [`ModuleLookup.cpp`](ModuleLookup.cpp.md) · [`ModuleLookup.hpp`](ModuleLookup.hpp.md) | Loading a named shared module, resolving symbols, unloading — three platform answers made one. |
| [`os_clipboard.cpp`](os_clipboard.cpp.md) · [`os_clipboard.h`](os_clipboard.h.md) | Clipboard read, write and append, with the codepage translation the engine's single-byte text needs. |
| [`stdafx.h`](stdafx.h.md) · [`stdafx.cpp`](stdafx.cpp.md) | The precompiled-header root. Incidental. |
| [`resource.h`](resource.h.md) | Numeric identifiers for one legacy dialog's controls. Incidental. |

### Time, identity and the wire

| File | Role |
|---|---|
| [`FTimer.h`](FTimer.h.md) | **The engine's clocks**: elapsed time that can be paused, scaled, or accumulated into a per-frame statistic. |
| [`FTimer.cpp`](FTimer.cpp.md) | The global pause registry, and the bodies the header leaves out. |
| [`clsid.cpp`](clsid.cpp.md) | **Class identifiers**: eight characters packed big-endian into 64 bits — the key every spawn record names its entity kind with. |
| [`clsid.h`](clsid.h.md) | Declares the type and the compile-time packing. |
| [`client_id.h`](client_id.h.md) | A connected client's identity, in a type of its own so it cannot be confused with an entity or a frame number. |
| [`NET_utils.cpp`](NET_utils.cpp.md) | **The packet's byte layout**: a fixed-capacity buffer with two cursors, and the quantizations that turn angles, directions and bounded floats into one or two bytes. |
| [`net_utils.h`](net_utils.h.md) | Declares it, and the optional mirror that dumps every field as text for debugging. |
| [`xr_shortcut.h`](xr_shortcut.h.md) | A key binding: scancode plus modifier bits in 16 bits, so a binding table is an array of integers. |
| [`_random.h`](_random.h.md) | **The simulation's random generator**: a 32-bit linear congruential sequence with fixed constants, and the one global instance. Its determinism is a conformance criterion. |
| [`fastdelegate.h`](fastdelegate.h.md) | The engine's callback currency: a callable reference that is comparable and orderable, so it can be stored in a set and removed again. |
| [`cdecl_cast.hpp`](cdecl_cast.hpp.md) | Turning a capture-free closure into a plain function pointer for foreign code. |

### The math types (declared here, bodies in chapter 3)

| File | Role |
|---|---|
| [`_vector3d.h`](_vector3d.h.md) | **The 3-vector**: three floats, no padding, and the operation vocabulary the whole engine speaks. |
| [`_vector3d_ext.h`](_vector3d_ext.h.md) | Value-returning vector arithmetic, for code where readability beats the no-temporaries rule. |
| [`_vector2.h`](_vector2.h.md) | Screen positions, texture coordinates, and the horizontal plane the AI navigates in. |
| [`_vector4.h`](_vector4.h.md) | Shader constants, planes, and anything that must reach the device as a four-float register. |
| [`_matrix.h`](_matrix.h.md) | **The 4×4 transform**: sixteen reals in a fixed order, and the conventions every transform obeys. |
| [`_matrix33.h`](_matrix33.h.md) | The rotation-and-inertia workhorse, with a Jacobi eigen-decomposition attached. |
| [`_quaternion.h`](_quaternion.h.md) | **The quaternion**, and the spherical interpolation every blended animation runs through. |
| [`_fbox.h`](_fbox.h.md) · [`_fbox2.h`](_fbox2.h.md) | The axis-aligned box in 3D and 2D — the vocabulary every spatial structure is built from. |
| [`_obb.h`](_obb.h.md) | The oriented box: the tight bound where an axis-aligned one would be mostly empty. |
| [`_sphere.h`](_sphere.h.md) | Centre and radius, with the cheap first tests visibility, audio and collision all use. |
| [`_sphere.cpp`](_sphere.cpp.md) | **The smallest enclosing sphere** of a point set — what turns a mesh's vertices into its bounding sphere. |
| [`_plane.h`](_plane.h.md) · [`_plane2.h`](_plane2.h.md) | The plane and its 2D twin: the classify/project/intersect vocabulary frustums and portal clipping are written in. |
| [`_cylinder.h`](_cylinder.h.md) · [`_cylinder.cpp`](_cylinder.cpp.md) | The finite capped cylinder, and the ray test with its two degenerate orientations. |
| [`_rect.h`](_rect.h.md) | The rectangle, real and integer — the screen-space region the UI and render targets are written against. |
| [`_color.h`](_color.h.md) | **Colour in two forms** — four floats for arithmetic, one packed word for the device — and the exact conversion. |
| [`_compressed_normal.h`](_compressed_normal.h.md) · [`_compressed_normal.cpp`](_compressed_normal.cpp.md) | **Sixteen-bit unit vectors**: the encoding shipped vertex data and network packets use for normals. |
| [`_bitwise.h`](_bitwise.h.md) | Integer-level operations on floats, power-of-two arithmetic, population counts, and two polynomial approximations. |
| [`_flags.h`](_flags.h.md) | **The bit field**, in four widths — and several of those words are serialized, so the bit positions are frozen. |
| [`_math.cpp`](_math.cpp.md) | **Numeric bring-up**: detect the vector instruction sets, seed the global generator, build the normal table, and set every thread's floating-point mode. |
| [`_math.h`](_math.h.md) | Declares the capability record, the monotonic clock, and the two bring-up entry points. |
| [`math_constants.h`](math_constants.h.md) | The three comparison epsilons and the circle constants, single-precision. |
| [`vector.h`](vector.h.md) · [`_stl_extensions.h`](_stl_extensions.h.md) | Aggregation headers — the whole math vocabulary, and every container alias, in one include each. |
| [`xr_types.h`](xr_types.h.md) | **The scalar vocabulary**: fixed-width integer names, pointer-to-text names, numeric limits, and the fixed-size character buffer names every stack string is declared with. |

### Model file framing

| File | Role |
|---|---|
| [`FMesh.hpp`](FMesh.hpp.md) | **The visual-model container format**: its chunk numbering, mesh kinds, and vertex-format identifiers. |
| [`FMesh.cpp`](FMesh.cpp.md) | Serializing the model file's authoring-provenance record. |

## What could not be recovered

Collected from the twins, for the root README's honesty section:

- The archive obfuscator's two key sets are dates written as digits; why *those* dates, and why
  two different iteration counts, is not discoverable.
- Nothing in an archive says which obfuscation key it used. Trial-and-rollback is the only
  method, and it may be the original's method too or may be a later reconstruction.
- The statistical compressor's fitted tables — eight binary-escape seeds, sixteen exponential
  escape values, two inheritance frequency formulas with their thresholds, a glue budget and a
  compaction threshold — have no derivation anywhere in the source and must be copied.
- That compressor's trained-model *writer* survives only as a commented-out block, and the flag
  that would preserve a trained model across streams is commented out too, leaving the intended
  lifetime ambiguous.
- Its progress hook is called every 256 escapes and bound to an empty function; nothing says
  what it was meant to report.
- The sorted-array map's lazy-sort hook is empty and its positional-hint insert has an
  unsatisfiable condition. Both are vestiges; neither behaviour exists.
- One character in the line-breaker's forbidden-start set — the ideographic digit one — is not
  explained anywhere, and the most plausible reason (it is used as a full-width dash in some
  shipped Chinese text) is unconfirmed.
- The archive entry point's alias form is parsed but not resolved: only the alias's length is
  used, to strip it. That looks like a defect and is preserved because the shipped archives use
  it in a form where the two happen to agree.
