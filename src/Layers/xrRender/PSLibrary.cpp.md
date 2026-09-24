# src/Layers/xrRender/PSLibrary.cpp

> The particle library: every effect and group definition the game ships, loaded once from a single packed file, sorted by name, and looked up by name for the rest of the process's life.

**Needs** — [`PSLibrary.h`](PSLibrary.h.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleGroup.h`](ParticleGroup.h.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`Include/xrRender/particles_systems_library_interface.hpp`](../../Include/xrRender/particles_systems_library_interface.hpp.md)
**Used by** — [`PSLibrary.h`](PSLibrary.h.md)
**Tier floor** — T1: it walks a chunked file whose nested chunks are numbered sequentially from zero, and the outer chunk identifiers separate two generations of the shipped format.

## Purpose

There is exactly one particle library in the process. It is loaded at renderer start-up from one file in the game data, it holds every effect definition and every group definition by name, and it is the only thing that creates or destroys those definitions. Every playing effect and group in the world points into it.

It exists as a separate file from the definitions themselves because it owns two things they do not: the *file*'s layout (as opposed to each record's), and the lifetime rule — a definition's compiled material is created when the library loads and destroyed when it unloads, never per instance.

## State

```text
RECORD ParticleLibrary
  effects : list<EffectDefinition>   # sorted by name, ascending, byte-wise
  groups  : list<GroupDefinition>    # sorted by name, ascending, byte-wise
```

**Invariants**

- Both lists are **sorted by name after loading and stay sorted**, because lookup is a binary search over them. A rebuild that appends at run time must re-sort or it silently fails to find the appended entry.
- The sort is a plain byte-wise comparison of the name, not a case-insensitive or locale-aware one. Names in the shipped library are already lowercase; a lookup with different case will not find its entry.
- Materials are created for every effect definition *after* the sort and *after* every definition has loaded, in one pass. Nothing between the start of loading and that pass may draw.
- Unloading destroys every effect's material first, in its own pass, and only then deletes the definitions. Group definitions own no device resources and are simply deleted.
- An effect and a group may not share a name: removal looks for an effect first and only tries groups if no effect matched.

## The file format

The library is one chunked file. Its layout is frozen:

```text
chunk 0x0001  VERSION        : int (16-bit), must equal 1
chunk 0x0003  SECOND_GEN     : container of effect definitions
chunk 0x0004  THIRD_GEN      : container of group definitions
```

Inside each container the records are **nested chunks numbered sequentially from zero** — chunk 0 is the first record, chunk 1 the second, and the walk ends at the first index that is absent. There is no count. Identifier 0x0002 is a first-generation container that the current engine does not read; it belongs to a format generation that predates the shipped games.

The names "second generation" and "third generation" are the format's own and do not mean "effects came second". They are generation markers of the *library format*: effects were re-encoded in the second revision and groups appeared in the third.

```text
FUNCTION load(path) -> bool
  IF path does not exist THEN log and RETURN false
  require chunk VERSION; IF version != 1 THEN RETURN false

  IF container SECOND_GEN present THEN
    FOR index FROM 0 WHILE nested chunk index exists
      definition = read one effect definition from it
      IF it loaded THEN append to effects ELSE stop the whole load and fail

  IF container THIRD_GEN present THEN
    FOR index FROM 0 WHILE nested chunk index exists
      definition = read one group definition from it
      IF it loaded THEN append to groups ELSE stop the whole load and fail

  sort effects by name; sort groups by name
  FOR EACH effect IN effects: effect.create_material()
  RETURN whether every record loaded
```

**Notes** — A record that fails its own version gate aborts the rest of the walk, and the library keeps whatever loaded before it. The engine continues with a partial library rather than refusing to start. That is a deliberate degradation: a modded library with one bad group still runs the game, minus that group and everything after it.

## `load_from_directory` — the loose-file form

**Contract** — an alternative to the packed library used by the authoring path: it enumerates a directory for files with the two particle extensions, reads each one in the configuration format, and names the definition by its path plus basename *without* the extension. Sorts and compiles materials exactly as the packed load does.

```text
FUNCTION load_from_directory() -> bool
  FOR EACH file MATCHING the effect and group extensions UNDER the particles root
    config = parse it as configuration text
    name   = the file's directory prefix and basename, extension dropped
    IF the extension is the effect one THEN
      definition = read effect definition from config; keep it if it loaded
    ELSE IF the extension is the group one THEN
      definition = read group definition from config; keep it if it loaded
    ELSE abort — the enumeration asked for two extensions and must see only those
  sort both lists; create every effect's material
```

**Notes** — The name is built from the *path relative to the particles root* plus the basename, so a definition in a subdirectory carries that subdirectory in its key. The packed form stores its names explicitly, so the two forms only agree if the directory layout matches the names the packed library was built with. A rebuild must not "helpfully" strip directories.

## `on_create`, `on_destroy`, `reload`

**Contract** — bring the library up from the game data, tear it down, or do both. Loading resolves the library file through the virtual filesystem under the game-data root with a fixed filename. `reload` is a console-driven convenience for modders and is the reason teardown must be complete: any definition left alive after `on_destroy` becomes a dangling target for an effect that outlived the reload.

**Invariants** — `on_destroy` releases materials before deleting definitions, in two separate passes over the effect list. One pass would work too; two is how it is written and the ordering (materials first) is the part that matters, because releasing a material after its owner is gone is the bug this shape prevents.

## `find_effect`, `find_group`

**Contract** — look up a definition by exact name. Returns nothing for an absent name and for an empty one. Does not allocate, does not load on demand — a name that is not in the library at start-up will never resolve.

```text
FUNCTION find_effect(name) -> optional<EffectDefinition>
  IF name is empty THEN RETURN none
  position = binary search of effects for name
  IF position is past the end OR the entry's name differs THEN RETURN none
  RETURN the entry
```

**Notes** — The authoring build uses a linear scan instead, because there the list is being edited and is not reliably sorted. The game build's binary search is the contract a rebuild should implement; the linear form is incidental.

## `append_effect`, `append_group`, `remove`, `rename_effect`, `rename_group`

**Contract** — the editing surface: add an empty or copied definition, remove one by name, rename one in place. Removing an effect releases its material first. These exist for the authoring tools; a shipping rebuild may omit them entirely, provided it still gets a sorted library from the loader.

**Notes** — Renaming writes the new name into the definition and does **not** re-sort, which leaves the library's binary search broken until the next load. In the authoring build lookup is linear, so nothing notices. A rebuild that keeps the editing surface and the binary search must re-sort on rename.

## The group-enumeration interface

**Contract** — the library fills a small abstract interface that lets other modules walk the group definitions and read a group's name without knowing the library's type at all. It is the seam through which the game's script layer and the sound/visual placement tooling enumerate available group names.

```text
INTERFACE ParticleGroupEnumeration
  groups_begin() -> cursor
  groups_end()   -> cursor
  groups_next(INOUT cursor)
  group_id(group) -> text
```

**Notes** — The shipped implementation returns the *last element* as the end cursor rather than one past it, and an empty library returns the same value from both ends. A caller walking `begin` to `end` with a not-equal test therefore stops one group early. A rebuild should use a half-open range and fix the off-by-one; nothing in the shipped data depends on the last group being unreachable, and the assertions inside `groups_next` are written as if the range were half-open, which is the clearest evidence that the shipped form is a defect rather than a convention.
