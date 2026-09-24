# src/xrEngine/IGame_ObjectPool.cpp

> Creates a client object from its configuration section name, and warms the model and texture caches by instantiating every object the game type declares it will need.

**Needs** — [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`xr_object.h`](xr_object.h.md) · [`Render.h`](Render.h.md) · [`EngineAPI.h`](EngineAPI.h.md)
**Used by** — [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md)
**Tier floor** — T2: a class-identifier lookup, a factory call and a list; nothing here touches bytes or the device directly.

## Purpose

Two jobs that share one table. The first is the **object factory**: given the name of a
configuration section, read its declared class identifier, ask the game module's registry
for an instance of that class, and load the section into it. Every client object in the
engine is born here.

The second is the **prefetch**: before a game type starts, instantiate one of everything
the configuration says that game type will use, so that the models, skeletons and textures
those objects reference are pulled through the loaders once, at load time, rather than the
first time the player sees one. The instances themselves are never handed out — they are
held only so their resources stay referenced, and deleted when the game type ends.

The name is a leftover. An earlier design (preserved commented-out in the source) really
was a pool: `create` looked for a cached instance and removed it from the pool, and
`destroy` returned the instance to the pool instead of freeing it. That recycling was
abandoned — `create` now always constructs and `destroy` always frees — and what remains is
a factory plus a cache-warming list. A rebuild should name it for what it does.

## State

```text
RECORD ObjectPool
  prefetched : list<GameObject>   # held only to keep their resources alive

# invariant: prefetched is empty outside a running game type. Both construction and
#            destruction assert it, because a leftover entry means the previous game
#            type's resources are still pinned.
```

## `create`

**Contract** — builds one client object from a configuration section name. Reads the
section's `class` key as a class identifier, asks the game module's class registry for an
instance, records the section name on the object, then gives the object two chances to read
its own configuration: a load pass and a post-load pass. Returns the object; the caller
owns it. Fails hard if the section or its class key is missing — an unknown class is a data
error, not a runtime condition.

```text
FUNCTION create(name) -> GameObject
  class_id = config[name]."class"
  object   = class_registry.instantiate(class_id)
  object.section_name = name
  object.load(name)
  object.post_load(name)          # see note
  RETURN object
```

**Notes** — the two-phase load exists because a section's values can only be fully
interpreted once the base class has read its own: the post-load pass is where a derived
class fixes up anything that depended on what the base just read. A rebuild that loads a
section into a fully-constructed object in one pass does not need the split.

The class identifier in the data is a four-character tag, not a name — see the game
module's class registry. The indirection is the seam that lets the game module own the
concrete types while the engine owns the factory call.

## `prefetch`

**Contract** — instantiates one object for every entry of the configuration section
`prefetch_objects_<game type>`, loads each, and keeps them. Silences the renderer's
per-model load logging for the duration, because this pass loads hundreds of models and the
log is not useful. Asserts the list is empty on entry.

```text
FUNCTION prefetch(pool)
  REQUIRE pool.prefetched IS empty
  section = "prefetch_objects_" + current game type
  renderer.model_logging(off)
  FOR EACH (entry_name, _) IN config[section]
    class_id = config[entry_name]."class"
    object   = class_registry.instantiate(class_id)
    object.load(entry_name)
    append object TO pool.prefetched
  renderer.model_logging(on)
```

**Notes** — the section is named per game type (`single`, `deathmatch`, and so on) because
a multiplayer match and a single-player level need different working sets, and warming the
wrong one wastes both time and memory.

The *values* in that section are ignored — only the key names matter. The abandoned pooling
design read each value as a count and created that many copies (capped at 128, asserted);
with recycling gone there is no reason for more than one of each, but the shipped
configuration still carries the counts. A rebuilder reading those files should expect
numbers that mean nothing.

Note what this function does *not* do: it never calls `create`, so the prefetched objects
never get the post-load pass and never have their section name assigned by the pool — they
are asserted to have set it themselves during load. They are not meant to be used, only to
have pulled their resources in.

## `clear`

**Contract** — destroys every prefetched object, releasing the last reference each holds to
its models and textures. Called when a game type ends. Leaves the list empty, which is what
the next `prefetch` asserts.

## `destroy`

**Contract** — frees one object. A bare deallocation today; in the abandoned design it was
the return leg of the recycling pool. A rebuild can delete this and let the caller own the
lifetime directly.
