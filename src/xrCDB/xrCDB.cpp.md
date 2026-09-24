# src/xrCDB/xrCDB.cpp

> Owns a level's collision geometry, builds the immutable box tree over it, and
> reads and writes that tree to disk so a level need not pay for the build twice.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it owns two large flat arrays that are handed out as raw spans to
traversals in other files, and the serialized tree is a memory image whose node records
must land at fixed byte offsets.

## Purpose

This file is the *model*: the pair of arrays that is the collision world, plus the tree
index over them, plus that tree's whole lifecycle — build, save, load, measure, release.
The three query files are deliberately separate because each is a different traversal over
the same node array, and none of them needs anything from this file except two pointers
and the root node.

The tree build itself is the part the original delegates to a vendored library. It is
described here in full, because a rebuilder should write it: it is one recursive partition
and one flattening pass.

## State

The model owns four things, and the invariants between them are what make a query safe.

```text
RECORD Model
  verts       : list<vec3>          # positions, in level space
  tris        : list<Triangle>      # index into verts; the soup
  tree        : optional<FlatNode>  # the index; see "the tree build"
  status      : enum{ init, building, ready }
  source_crc  : int (32-bit)        # checksum of the file the geometry came from

# Invariants
#   every tris[i].v[k] is a valid index into verts
#   tree is present and complete  <=>  status == ready
#   a query may read verts, tris and tree only while status == ready
#   once status becomes ready, none of the four is ever written again
#   count(tris) >= 2 and count(verts) >= 4  -- a tree of one triangle has no interior node
```

```text
RECORD Triangle                     # exactly 16 bytes; frozen, see the level format
  v                : list<int (32-bit)>[3]   # indices into verts
  material         : int (14-bit)            # index into the material library
  suppress_shadows : bool (1 bit)            # cached from the material
  suppress_wm      : bool (1 bit)            # cached from the material: takes no decals
  sector           : int (16-bit)            # visibility sector this surface belongs to
# The last four fields share one 32-bit word and are also addressable as that whole word,
# which is how a query copies the payload out in one move without unpacking it.
```

The payload's bit widths are a data-format decision, not a packing convenience: they are
read from and written to level files and to the build cache, so a rebuild must reproduce
them exactly. 14 bits of material caps a level at 16 384 distinct static materials, which
is two orders of magnitude more than any shipped level uses.

## `Model.build`

**Contract** — takes ownership of a copy of a triangle soup and produces the tree over it.
Takes an optional *fix-up step* invoked after the arrays are copied but before the tree is
built, which is how the game layer rewrites material ids from the level's numbering into
the running material library's numbering (see
[chapter 8](../xrMaterialSystem/README.md)) — the tree must be built over already-fixed
triangles because the fix-up may not change geometry and the build must see final data.
Blocks until the tree is ready. Allocates two arrays sized to the input plus the node
array. Must be called with the model in the `init` state.

**Invariants** — on return, `status == ready` and the tree is complete.

```text
FUNCTION build(V, T, fixup) -> ()
  REQUIRE status == init
  REQUIRE count(V) >= 4 AND count(T) >= 2

  verts <- copy of V                # copy, not borrow: the caller's buffer is a mapped
  tris  <- copy of T                # region of the level file and is released after load
  IF fixup IS NOT none: fixup(verts, tris)

  status <- building
  tree   <- build_tree(verts, tris)   # see below
  status <- ready
```

**Notes** — the original can run the build on a worker thread when a command-line switch
asks for it, and then immediately spins waiting for it to finish, so the two paths are
observably identical. The worker path exists so that a *query issued during the build*
blocks on the build lock instead of reading a half-built tree; that guard is the
`await_ready` contract below. In a rebuild, either build synchronously and delete the guard,
or build asynchronously for real and make load order wait on it — the half-measure in the
original buys nothing.

The build takes seconds for a real level, which is why this path is the fallback and the
cached paths below are the normal ones.

## the tree build

This is the delegated part, stated so it can be written.

**What is built.** A *complete* binary tree over the triangle array: every leaf holds
exactly one triangle, so a soup of `N` triangles produces `N` leaves and `N-1` interior
nodes. Each interior node carries the axis-aligned box that encloses every triangle beneath
it. Leaf boxes are never stored — a leaf is only a triangle index — because at a leaf the
traversal goes straight to the exact triangle test and a box test there would be wasted
work. This is the *no-leaf* shape, and it is the reason the stored node count is `N-1`
rather than `2N-1`.

**Phase 1 — partition.** Work over an index array holding `0..N-1`, which is permuted in
place; a node is a contiguous range of that array. Recursively:

```text
FUNCTION subdivide(range) -> ()
  box(range) <- the axis-aligned box enclosing all three vertices of every triangle in range
  IF count(range) == 1: RETURN                  # leaf criterion: one triangle, full stop

  # choose the axis: greatest spread of triangle centroids
  FOR EACH axis IN {x, y, z}
    mean[axis]     <- average over range of centroid(tri)[axis]
    variance[axis] <- average over range of (centroid(tri)[axis] - mean[axis])^2
  axis <- the axis with the largest variance

  # choose the plane: the mean of the *vertices* along that axis, not the box centre
  split <- (sum over range of (v0[axis] + v1[axis] + v2[axis])) / (3 * count(range))

  # partition: everything strictly above the plane goes first
  stable-free partition of range so that centroid(tri)[axis] > split comes first
  k <- number of triangles above the plane

  IF k == 0 OR k == count(range):               # degenerate: every centroid on one side
    k <- count(range) / 2                       # forced halving -- see note

  subdivide(range[0 .. k))
  subdivide(range[k .. end))
```

`centroid(tri)[axis]` is the mean of the triangle's three vertex coordinates on that axis.

**Why these three choices.** Picking the axis by *centroid variance* rather than by longest
box extent costs two passes over the node's triangles and produces a better partition on
the geometry that actually occurs in a level: long thin buildings and terrain strips have a
longest box axis along which the triangles are not actually spread. Splitting at the
*vertex mean* rather than the box centre biases the plane towards where the geometry
actually is, which matters because a level's triangle density is wildly uneven — one node
may enclose a whole empty courtyard and a single dense doorway. Both choices are cheap
approximations of a surface-area cost model; neither computes one.

**The forced halving is what "complete" costs.** When every centroid falls on one side of
the plane — coplanar triangles, a stack of identical quads, a degenerate face — the split
is refused and the range is cut exactly in half instead, by index. The resulting two boxes
overlap heavily and queries into that region get slower, but the tree stays complete and
therefore every leaf still holds exactly one triangle, which is what lets the flattened
form below drop leaf boxes entirely and lets every traversal assume a strict binary shape
with no leaf-size loop. The alternative — stop subdividing and keep a multi-triangle leaf —
is the right choice for a general-purpose library and the wrong one here.

**The build-time / quality trade-off.** The partition is `O(N log N)` on well-behaved
geometry and degrades toward `O(N²)` on pathological input, with a small constant: three
passes per node (box, variance, partition) and no sorting anywhere. There is no
surface-area heuristic, no binning, no spatial splitting of a triangle across two nodes.
A rebuilder tuning this has four dials, in increasing cost:
*pick the longest box axis* (one pass, noticeably worse trees);
*pick by centroid variance* (what this does);
*try all three axes and keep the most balanced* (three partitions per node, better balance,
not better queries — balance is not the objective);
*binned surface-area heuristic* (several times the build time, materially faster queries).
Given that the built tree is cached and shipped, spending more at build time is the
obviously correct direction for a rebuild; the original's choice is a 2001 choice.

**Phase 2 — flatten.** The pointer tree is then copied into one contiguous array of `N-1`
records, depth-first, and thrown away. Each record is one box and two child slots:

```text
RECORD FlatNode
  center  : vec3        # the node's box, stored as centre and half-extent rather than
  extents : vec3        # min/max, because every traversal wants it that way
  pos     : slot        # the "above the plane" child
  neg     : slot        # the "below the plane" child

# A slot is one machine word that is either a reference to another FlatNode, or a
# triangle index. The two are distinguished by the low bit: a leaf slot is
# (triangle_index shifted left 1) with the low bit set; a child slot is a reference
# with the low bit clear. This is why the array is walked, never indexed by rank.
```

The root is record 0; children are assigned successive indices as the recursion descends,
so a node's two subtrees occupy one contiguous span each and a descent walks forward
through memory. That layout, plus dropping leaf boxes, is the whole point of flattening:
the node array for a large level is a few megabytes and a traversal touches it with good
locality.

The low-bit tagging is a space decision that survives into the file format (see
*serialization* below), so a rebuild that stores the tag in a separate field must still
reproduce the tagged form on disk or version the format.

## `Model.load_geom`

**Contract** — copies in a triangle soup *without* building anything, leaving `status` as
it was. Used only on the path where a prebuilt tree is about to be read from the level
file: the geometry and the tree arrive separately and the geometry must be in place first.

## `Model.serialize`

**Contract** — writes the model to a file so the next run can skip the build. Returns
whether the file could be opened. Takes an optional callback that writes caller-specific
validity data into the file after the header — the game layer writes the checksum of the
material library there, so that editing materials invalidates the cache.

**Invariants** — everything written is checksummed, and the checksums are what make the
cache safe to trust.

```text
FUNCTION serialize(path, extra_writer) -> bool
  file <- open for writing at path   ; IF failed: RETURN false

  write source_crc                    # checksum of the level file the geometry came from
  IF extra_writer IS NOT none: extra_writer(file)

  model_crc <- checksum over, in this order:
                 count(verts), count(tris), the verts bytes, the tris bytes
  write model_crc
  write count(verts), count(tris)
  write verts bytes
  write tris bytes

  write tree                          # see "tree serialization"
  RETURN true
```

Two independent checksums, because they answer two different questions. `source_crc`
answers *did the level's collision file change under us* — if it did, the cached tree
describes different geometry and must be discarded. `model_crc` answers *is this cache file
itself intact* — a truncated or corrupted cache must be rejected rather than walked, since
a bad node array is a wild pointer chase inside every query.

## `Model.deserialize`

**Contract** — reads a model written by `serialize`. Returns false — without disturbing the
model — for any of: file missing, source checksum mismatch, caller's extra check failing,
declared sizes larger than the remaining file, model checksum mismatch, tree read failing.
Every one of those means *build from scratch instead*, so none of them is an error the
caller reports; they are the normal outcome after a game update.

```text
FUNCTION deserialize(path, skip_checksum, extra_check) -> bool
  file <- open at path                       ; IF failed: RETURN false
  IF read int != source_crc: RETURN false
  IF extra_check IS NOT none AND NOT extra_check(file): RETURN false

  stored_crc <- read int
  mark <- current position
  nv <- read int ; nt <- read int
  span <- size of (nv, nt, nv vectors, nt triangles)
  IF span > bytes remaining: RETURN false                 # truncated
  IF NOT skip_checksum AND checksum(from mark, span) != stored_crc: RETURN false

  release verts, tris, tree
  verts <- read nv vectors ; tris <- read nt triangles
  IF NOT read_tree(file): RETURN false
  status <- ready
  RETURN true
```

**Notes** — the "skip the checksum" switch exists for developers who are editing geometry
and regenerating caches by hand and do not want to pay a checksum over a few hundred
megabytes on every start. It is a debugging affordance, not a mode, and a rebuild may drop
it. The size check before the checksum is not optional: the checksum reads the declared
span, so a corrupt length field would read past the buffer.

## tree serialization

The flattened node array goes to disk almost as it stands, which is the reason the flat
form exists at all.

```text
FUNCTION write_tree(file)
  write flag: leaves-are-inline (always true here)
  write flag: boxes-are-quantized (always false here)
  write node_count
  copy the node array
  in the copy, rewrite every non-leaf slot from a reference into a byte offset
    from the start of the array         # references are position-dependent; offsets are not
  write checksum over the copy
  write the copy

FUNCTION read_tree(file) -> bool
  IF read flag leaves-are-inline is false: FAIL WITH unsupported shape
  IF read flag boxes-are-quantized is true: FAIL WITH unsupported shape
  node_count <- read int
  IF node_count * size(FlatNode) > bytes remaining: RETURN false
  IF checksum over the node bytes != read checksum: RETURN false
  read the node array
  rewrite every non-leaf slot from an offset back into a reference
  RETURN true
```

The two shape flags are the vendored library's — it can also emit trees with leaf nodes,
and trees whose boxes are quantized to 16-bit fixed point. The engine uses neither and the
reader rejects both rather than implementing them. A rebuild should keep the two flags in
the format (so an existing cache file still parses) and may keep rejecting the other three
combinations.

The relocation pass is the whole subtlety: a node's child slot is a reference in memory and
must be a position-independent offset on disk, and the low-bit leaf tag must be preserved
across both directions, so only slots whose tag says "child" are adjusted. Because the file
is *the array plus a relocation rule*, the record size is part of the format — a rebuild
that widens a slot or reorders the box fields has changed the cache format and must bump
the version the level file carries.

## `Model.await_ready`

**Contract** — returns once the tree is safe to read. Returns immediately in the normal
case; when a build is in flight it blocks on the build lock and logs a warning, because a
query racing a build means the level's load order is wrong. Every query calls it first.

**Notes** — this is the only synchronization in the entire query path, and once the model
is ready it is a single read of a status word that never changes again. That is the
concurrency contract of the whole module: *the tree is immutable after build, so readers
need nothing*. All mutable per-query state — the accumulated results, the shrinking ray
range — lives in the caller's own collider object, one per thread (see
[`xrXRC.h`](xrXRC.h.md)). Nothing is shared, so nothing is locked.

## `Model.memory`

**Contract** — reports the model's total footprint: nodes plus vertices plus triangles.
Reports zero and complains if asked while a build is in flight. Diagnostics only.

## `Collider.add_result` · `Collider.clear` · `Collider.release`

**Contract** — the result buffer's lifecycle. A query clears the buffer, the traversal
appends to it, and the caller reads it before issuing the next query. The buffer keeps its
capacity across queries on purpose: a thread issues thousands of queries per frame and the
steady state is zero allocations. Releasing is only for shutdown.

**Notes** — there is no result *budget* enforced here. The bound on a result set comes from
the query's option bits (see [`xrCDB_ray.cpp`](xrCDB_ray.cpp.md)): first-only stops at one,
nearest-only keeps exactly one and shrinks the search range as it goes, and an unbounded
query really is unbounded — a box query over a large volume can return tens of thousands of
triangles. Callers that cannot afford that bound the query volume instead.
