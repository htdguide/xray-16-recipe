# src/xrGame/level_debug.cpp

> The development build's diagnostic overlay: per-object floating labels, fixed screen text, and world-space shapes, each owned by whichever system wrote it.

**Needs** — [`level_debug.h`](level_debug.h.md) · [`Level.h`](Level.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [`debug_text_tree.h`](debug_text_tree.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`level_debug.h`](level_debug.h.md)
**Tier floor** — T3: registries and a projection to screen space

## Purpose

Diagnosing AI means watching several unrelated systems reason about the same creature at
once — the memory manager, the planner, the movement manager, a script. If they all wrote
into one text buffer they would clobber each other, and none could clear its own output
without clearing everyone's. This file is the answer: three registries whose keys are
(*who is writing*, *what it is about*), so each contributor gets a private, addressable slot
it can add to and clear independently, and the overlay draws the union.

Everything here exists only in a development build.

## State

```text
RECORD InfoItem      # a label that floats over an object
  text : text ; color : int (packed RGBA) ; id : int

RECORD TextItem      # a label at fixed screen coordinates
  text : text ; x : real ; y : real ; color : int ; id : int

RECORD LevelItem     # a shape in the world
  kind      : ENUM { point, line, box }
  position1 : (real, real, real)
  position2 : (real, real, real)   # line only
  radius    : real                 # box only
  color     : int ; id : int

RECORD LevelDebug
  object_info : map<GameObject, map<text, ItemList<InfoItem>>>
                                  # outer key: the object being described
                                  # inner key: the WRITER's runtime class name
  text_info   : map<(writer, text), ItemList<TextItem>>
  level_info  : map<(writer, text), ItemList<LevelItem>>
  text_tree   : TextTree           # the structured monster diagnostic
  tree_scroll : int                # how far down the tree the overlay is scrolled
```

**Invariant** — every item list is kept **sorted by identifier** after every add and every
remove. The identifier is the contributor's own ordering key, so the overlay shows a system's
lines in the order that system means them to read, not the order they happened to be
written. That is the only reason the sort exists; nothing looks items up by identifier.

**Invariant** — the composite key orders by the *writer* alone, ignoring the class name.
Two different systems at the same address would collide — which cannot happen, because the
address is the writing object's own. The class name rides along only so the key is
self-describing in a listing.

**Invariant** — item lists are owned by the registries and freed when their entry goes. The
described objects are not owned; an object being destroyed must notify this registry, or its
map entry holds a dangling key.

**Invariant** — the sentinel identifier (the largest representable) means "unordered", and
is the default. Items sharing it keep their relative insertion order only because the sort
is stable in practice — not a guarantee a rebuild should rely on.

## `object_info` · `text` · `level_info`

**Contract** — hand a contributor its own slot, creating it on first use. Never fail, never
return nothing: the caller always gets a list it may write to. `object_info` is
two-level — an object, then a writer — because a single creature is described by many
systems and each must be able to clear only its own lines. The other two are one level,
keyed by writer directly, because their content is not attached to any object.

The key is derived from the **caller's runtime class name**, obtained by asking the language
for the dynamic type of the object making the call. That is the incidental part: what the
system needs is a stable per-contributor identity that the contributor does not have to
invent or keep unique by hand. A rebuild can supply that identity however it likes — an
explicit string, a registration token — as long as it is stable across frames and distinct
between systems.

## `draw_object_info`

**Contract** — the only pass with real work. For each described object, projects a point
above it into screen space and stacks that object's labels downward from there, one writer's
block after another. Skips anything behind the camera or outside the view. Called once per
frame from the overlay pass.

```text
FUNCTION draw_object_info()
  FOR EACH (object, writers) IN object_info
    IF the object is gone or being destroyed THEN
      free its lists, drop the entry, STOP THIS PASS   # see note

    transform = camera_view_projection COMPOSED WITH object.transform
    stack = 0
    FOR EACH (writer, items) IN writers
      p = transform applied to the writer's offset   # default: 2 metres above the origin
      SKIP IF p is behind the camera
      SKIP IF p is outside the normalized view square
      x = viewport x of p
      y = viewport y of p - stack
      draw items downward from y, one line every delta        # default: 16 pixels
      stack = how far this block descended
```

**Invariants** — the offset and the line spacing are per-writer, not global, so one system
can place its block higher than another's on the same creature. The stack accumulates *only
the previous block's height*, not the total, which means three writers on one object overlap
rather than stacking cleanly. That is a real defect in the original; a rebuild should
accumulate.

**Notes** — the dead-object sweep is folded into the draw pass, and it **stops the whole
pass** after reclaiming one entry, because the container's iteration cannot continue past a
removal. One dead object therefore costs one frame of overlay per frame until the backlog
clears. The correct shape is to collect the dead and remove them after the walk; the reason
the original gets away with it is that `on_destroy_object` normally reclaims entries first
and this path is the fallback.

## `draw_text` · `draw_level_info`

**Contract** — draw every registered fixed-position label, and every registered world shape.
Neither does any culling or ordering beyond the per-list identifier order; both just walk
their registry. A point is drawn as a small box with a five-metre vertical line above it, so
that a marker on the ground is visible from across the level; a line is drawn as given; a box
is a cube of the given half-extent. The vertical line is the reason `point` is a separate
kind from a zero-extent box.

## `draw_debug_text`

**Contract** — draws the structured monster diagnostic, which is a tree rather than a flat
list and is laid out in three fixed columns. What it shows depends on what the camera is
attached to:

- **Not a monster** — nothing, unless a debug variable asks for the actor view, in which
  case only the `ActorView` subtree is drawn in the first column.
- **A monster** — the `General`, `Brain` and `Controllers` subtrees, one per column.

The three subtree names are a contract between this overlay and the monster code that fills
the tree; they are not discoverable from here. Layout is fixed to three columns of a third
of 1024 pixels each, from a fixed origin, wrapping at 80 characters, in green with yellow
highlights — all hard-coded, all arbitrary, none of it scaled to the actual resolution. On a
wide display the columns sit in the left portion of the screen. A rebuild should derive the
column width from the viewport.

## `log_debug_info` · `debug_info_up` · `debug_info_down` · `get_text_tree`

**Contract** — dump the whole text tree to the log; scroll the overlay's view of it up or
down by one line, with up clamped at the top and down unbounded; and hand out the tree
itself so that contributors can write into it. The scroll offset is shared by all three
columns.

**Notes** — down is unbounded, so scrolling past the end shows an empty overlay with no way
to tell why except pressing up repeatedly. Clamp it in a rebuild.

## `on_destroy_object`

**Contract** — the notification an object must send before it is destroyed. Frees every
writer's list for that object and drops the entry. This is the primary reclamation path;
the sweep inside the draw pass is the backstop for an object that failed to notify.

## `CItemBase` operations

**Contract** — the shared list behind all three registries: add one item, remove every item
whose text matches, remove every item whose identifier matches, clear, iterate. Add and both
removes re-sort. Removing by text is how a contributor updates a line in place — remove the
old text, add the new — which is why text is compared and not just displayed.

**Notes** — re-sorting the whole list on every single add is the naive choice, and it is fine
here: a list is a handful of lines and the whole thing exists only in a development build.
