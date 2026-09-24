# src/Layers/xrRender/NvTriStripObjects.cpp

> The stripifier itself: build adjacency over a triangle list, grow triangle strips by competitive experiment, chop them to cache size, order them for cache reuse, and flatten the result into an index stream with the winding preserved.

**Needs** — [`NvTriStripObjects.h`](NvTriStripObjects.h.md) · [`VertexCache.h`](VertexCache.h.md)
**Used by** — [`NvTriStripObjects.h`](NvTriStripObjects.h.md)
**Tier floor** — T2: pure index arithmetic, no device contact and no on-disk format. What stops T3 is cost — three separate passes here are quadratic in triangle count and run over meshes of tens of thousands of triangles, so the working structures must be flat arrays with predictable traversal, not dictionaries of objects.

## Purpose

Given a triangle list, produce an ordering of those same triangles that minimizes how many vertices are shaded twice. The tool's model of the hardware is a *post-transform vertex cache*: a fixed-size, first-in-first-out set of recently shaded vertices, where a hit costs nothing and a miss costs a full vertex shade. The whole algorithm is a search for an ordering with many hits.

The search is greedy at three different scales, and the three scales are the file's real structure:

1. **Within a strip** — grow a strip as far as the mesh allows in both directions from a seed edge. Inside a strip, every triangle after the first reuses two vertices for free; this is the cheapest reuse available and the tool takes all of it.
2. **Across strips from one seed** — from a seed triangle, keep starting new strips adjacent to what was just built, producing a *chain* of strips covering one region. Six such chains are grown from each seed (three edges, two directions) and the best one is kept.
3. **Across the whole mesh** — after all strips exist, chop them to cache size, discard the runts, and greedily reorder the survivors so each strip follows the one it shares most vertices with.

The three scales are independent decisions and a rebuild may replace any one of them. What it may **not** change is the output contract: same triangles, same winding.

## State

```text
RECORD FaceInfo                    # one triangle, mutable through the search
  v0, v1, v2      : int            # vertex indices, in the input's winding order
  strip_id        : int            # -1 until committed to a real strip
  test_strip_id   : int            # the strip it holds in the current experiment
  experiment_id   : int            # which experiment gave it test_strip_id

RECORD EdgeInfo                    # one undirected edge, shared by up to two faces
  v0, v1          : int
  face0, face1    : optional<FaceInfo>
  next_at_v0      : optional<EdgeInfo>   # next edge incident on v0
  next_at_v1      : optional<EdgeInfo>   # next edge incident on v1
  ref_count       : int, always 2 at creation

RECORD StripInfo
  start_face      : FaceInfo       # the seed
  start_edge      : EdgeInfo       # the seed edge the strip grows along
  start_to_v1     : bool           # which way along that edge, which fixes winding
  faces           : list<FaceInfo> # in strip order
  strip_id        : int
  experiment_id   : int            # >= 0 while speculative, -1 once committed
  visited         : bool           # used only by the final ordering pass
```

**Invariants**

- The adjacency index is *per vertex*: an array indexed by vertex number, each slot holding the head of a linked list of the edges incident on that vertex. Each edge therefore appears in two lists — hence the reference count of exactly two, which is how the structure knows when both lists have released it. A rebuild with a vertex-to-edges multimap drops the count and the intrusive links together.
- That array is sized by the **index count**, not the vertex count, which is safe because in a well-formed triangle list every vertex is referenced, so the largest vertex number is below the index count. It wastes roughly two thirds of the array. A rebuild should size it by the vertex count and take the caller's word for it.
- A face is *marked* — unavailable — if it holds a real strip id, or if it holds a test strip id stamped with the experiment currently running. The two-field scheme is what lets many speculative strips be grown over the same triangles without copying the mesh: an experiment's marks are invisible to every other experiment because the experiment ids differ. A rebuild with cheap copying can use a scratch mark set per experiment instead.
- An edge with more than two incident faces keeps the first two and silently ignores the rest. The mesh is assumed manifold; a non-manifold one produces a worse ordering, never a wrong one.
- Exact duplicate triangles — same three indices in the same rotation — are dropped on the way in. This is both a correctness guard (the strip walk derails on duplicates; the code says so where it detects the symptom) and the single most expensive thing in the file, since the check is a linear scan of every face accepted so far.

## `Stripify` — the whole pipeline

**Contract** — takes the triangle list, the device cache size and the minimum strip length; returns the strips in final draw order and the leftover triangles, also in a cache-friendly order. Allocates the face and strip records and hands them to the caller; releases the adjacency structure itself. Single-threaded, and not reentrant: the strip builder keeps a scratch buffer across calls.

```text
FUNCTION stripify(indices, requested_cache_size, min_strip_length)
         -> (strips, leftover_faces)
  effective_cache <- max(1, requested_cache_size - CACHE_INEFFICIENCY)
      # the hardware cache is never fully exploitable — entries are lost to
      # the pipeline's own traffic — so the search targets a smaller cache
      # than the device reports. See the note on the margin below.

  faces, edges <- build_adjacency(indices)      # duplicate triangles dropped
  raw_strips   <- find_all_strips(faces, edges, samples = 10)
  RETURN split_and_order(raw_strips, edges, effective_cache, min_strip_length)
```

**Notes**

- `CACHE_INEFFICIENCY` is 6. Nothing in the source derives it and no comment defends it; it is a tuning constant from the library this came from. Treat it as "reserve about a quarter of a small cache", and expect to re-tune it if the target hardware's cache behaves differently. The clamp to at least one keeps a pathological setting from producing zero-length strips.
- Because the margin is subtracted here and not in the interface, the number a caller passes is the *real* cache size. A rebuild that exposes the effective size instead will silently generate longer strips than intended.

## `BuildStripifyInfo` — adjacency

**Contract** — walks the triangle list once, materializes a face record per unique triangle, and threads its three edges into the per-vertex edge lists, creating an edge the first time it is seen and attaching the second face the second time. No output beyond the two structures.

**Notes** — The three edges are handled by three copies of the same block; a rebuild writes it once. What is load-bearing is only that each of a triangle's three edges is registered, that an edge is found by either endpoint, and that the second registration of an edge attaches the second face.

## `FindAllStrips` — the experiment loop

**Contract** — covers the mesh with committed strips. Repeats until no uncommitted triangle can be reached: pick up to `samples` scattered seed triangles, grow six candidate strip *chains* from each, keep the single best chain, release the rest. Mutates the face records' strip ids. Allocates strip records; the losers are released here, the winners escape to the caller.

**Invariants** — every triangle ends up in exactly one committed strip. The loop terminates because each round commits at least one strip, and a seed is only chosen from uncommitted triangles.

```text
FUNCTION find_all_strips(faces, edges, samples)
  WHILE true
    candidates <- empty
    seen_seeds <- empty set
    FOR i IN 0 .. samples-1
      seed <- find_good_reset_point(faces, edges)
      IF seed is none
        RETURN                       # every triangle is committed
      IF seed IN seen_seeds
        CONTINUE                     # the scatter wrapped onto a seed already tried
      seen_seeds.add(seed)

      # six chains from this seed: each of the three edges, each direction.
      # Direction matters because it fixes which vertex leads, and therefore
      # both the winding of the first triangle and which way the strip grows.
      FOR EACH (edge, direction) IN the seed's three edges x two directions
        candidates.append(chain starting at (seed, edge, direction))

    FOR EACH chain IN candidates
      build the chain's first strip
      WHILE a next seed can be found adjacent to what this chain just built
        append a new strip grown from it
      # the chain's marks are stamped with its own experiment id, so the
      # chains do not see each other's claims

    best <- chain maximizing  avg_strip_size * (1 + strip_count)
    commit(best)                     # stamp real strip ids, release the losers
```

**Notes**

- The score is written as `avg_strip_size * w1 + avg_strip_size * strip_count * w2` with both weights fixed at one, which reduces to average strip length times one-plus-strip-count — and since average length times count is the triangle total, the score is **triangles covered, with a tiebreak toward longer average strips**. Coverage dominates. That is the decision: the tool prefers a chain that claims more of the mesh over one that claims fewer, prettier strips. The two weights are named constants that are never varied; a rebuild can either keep them as the tuning hook they were meant to be, or write the reduced form and be honest.
- Six chains per seed rather than three is not redundancy. The two directions along one edge start the strip with opposite leading vertices, which changes both the winding parity of the emitted stream and which neighbour is reachable first, so they genuinely diverge.
- `samples` is 10, and the candidate array is pre-sized to six per sample. Neither number is derived anywhere. Ten is a cost/quality dial: the loop's work is linear in it, and so is the chance of finding a good region.
- Growing every chain to completion before scoring is what makes this expensive, and it is deliberate — a chain that starts badly may still cover the most triangles, and a short lookahead would not see that.

## `FindGoodResetPoint` — where to start the next region

**Contract** — returns an uncommitted triangle to seed from, or nothing if there are none. Advances the scatter cursor as a side effect.

**Invariants** — the returned face is always uncommitted; the search wraps around the face array and stops where it started.

```text
FUNCTION find_good_reset_point(faces, edges)
  IF this is the first call for this mesh
    start <- first face with two or more boundary edges   # a corner of the mesh
    # Starting at a corner makes the first strips run along the surface
    # instead of spiralling out of the middle of it. A closed mesh has no
    # corner, and then this fails and the scatter below is used instead.
  IF there is no such face
    start <- floor((face_count - 1) * scatter)

  scan forward from start, wrapping, for the first uncommitted face

  scatter <- scatter + 0.1
  IF scatter > 1.0
    scatter <- 0.05
  RETURN the face found, or none
```

**Notes**

- The scatter cursor is the interesting decision. Consecutive seeds are taken from positions spread across the face array rather than from wherever the last strip ended, so the tool works on several *different* regions of the mesh before returning to any of them. The stated reason is that large open spans get stripified while they are still open; a tool that always continued from the last strip would nibble one region into a maze of short leftovers.
- The increment of 0.1 and the wrap to 0.05 rather than 0.0 are undocumented. The wrap value being nonzero means the cursor never revisits exactly the same offsets on successive sweeps, which spreads the seeds further — but nothing in the source says that was the intent.
- "Two or more boundary edges" — not one — is what the first-call search asks for. One boundary edge is merely a border triangle; two is a corner, and a corner is the only place where the direction the strip should run is unambiguous.

## `StripInfo.Build` — growing one strip

**Contract** — grows the strip from its seed edge forward until blocked, then backward until blocked, and stores the two runs joined into one. Marks every triangle it takes. Mutates the face records.

**Invariants**

- The stored order is the backward run **reversed**, then the forward run. The seed triangle sits at the join, and the whole sequence is a single walk across the mesh — which is what makes it a strip at all.
- The walk is blocked by three things, and the third is the one that matters: the edge has no other face (the mesh boundary), the next face is already marked (someone else has it), **or** the next face's three vertices all already appear in the strip. The last test forbids a strip from wrapping around and closing on itself. A wrapped strip would emit triangles that the strip topology cannot express, so it is cut instead.

```text
FUNCTION build(strip, edges)
  v0, v1 <- the seed edge's endpoints, ordered by the strip's direction flag
  v2     <- the seed triangle's remaining vertex
  # v0, v1, v2 is now the seed triangle in the winding the stream will carry

  forward  <- walk from the seed across edge (v1, v2)
  backward <- walk from the seed across edge (v1, v0)
  strip.faces <- reverse(backward) followed by forward

  WALK from a face across an edge:
    WHILE the edge has an unmarked other face that is not wholly inside the strip
      take it, mark it, and advance:
        the new leading edge is the old one's second vertex plus the taken
        face's third vertex
```

**Notes**

- "The taken face's third vertex" is found by asking which of its three vertices is neither of the last two emitted. That is the whole of the next-index rule, and it is also the duplicate-triangle detector: if no such vertex exists, the walk has been handed a triangle it already emitted, and the tool warns and gives up on that step rather than looping.
- The wrap test scans the strip so far for each candidate, making a strip's growth quadratic in its own length. Strips are short — bounded in practice by the cache chop that follows — so this is tolerable, but a rebuild should carry a vertex set alongside the face list and make the test constant-time.
- The scratch index buffer used while walking is reused between calls. That is the concrete reason the whole tool is single-threaded, and it buys nothing a per-call buffer would not.

## `SplitUpStripsAndOptimize` — chop and order

**Contract** — takes the committed strips and returns the final draw order, plus the triangles evicted from runt strips. Splits, filters and reorders; allocates fresh strip records for the pieces. The input strips' records are the caller's to release.

```text
FUNCTION split_and_order(strips, edges, cache, min_length)
  pieces <- empty
  FOR EACH strip IN strips
    cut strip into consecutive runs of at most `cache` triangles
    # A strip longer than the cache cannot keep its own head resident, so
    # the extra length buys nothing; cutting it frees the ordering pass
    # below to interleave the pieces with strips they share vertices with.
    pieces.append(each run)

  survivors, leftover_faces <- remove_small_strips(pieces, min_length, cache)

  cache_model <- empty vertex cache of size `cache`
  first <- the survivor minimizing (total neighbours / face count)
  # Start with the most isolated piece. It shares least with everything else,
  # so it will never be a good follower; spend it while the cache is cold.
  emit first, feed its vertices to cache_model

  WHILE an unemitted survivor remains
    next <- the unemitted survivor with the most cache hits per face
            against cache_model
    emit next, feed its vertices to cache_model
```

**Invariants** — the union of emitted strips and leftover faces is exactly the input triangle set. The cache model is a plain FIFO of vertex numbers: a hit is membership, an insertion evicts the oldest.

**Notes**

- The ordering loop rescans every remaining strip on every step, so it is quadratic in strip count. The source says so in a comment and calls it the reason stripification is slow. A rebuild with any priority structure keyed on shared vertices wins here immediately; the *decision* being preserved is "each strip follows the one it shares the most vertices with", not the scan.
- Hits are counted **per face**, not per strip, so a long strip is not preferred merely for being long. That normalization is what keeps the greedy choice honest when the pieces differ in size — and after the chop above they mostly do not, which makes the normalization nearly free and easy to miss.
- The isolation metric for the first piece is neighbours per face, where a neighbour is an adjacent triangle across one of the three edges. A piece deep inside the mesh scores three; a piece on the border scores less.

## `RemoveSmallStrips` — demotion to a list

**Contract** — strips shorter than the minimum are dissolved into loose triangles; the rest pass through. The loose triangles are then themselves ordered greedily against a fresh cache model, by the same hits-per-triangle rule used for strips. Returns both.

**Notes** — The leftover list is not a dumping ground: it is optimized with the same criterion, because it will be drawn as a triangle list right after the strips and its cache behaviour matters just as much. The two greedy passes are the same algorithm at different granularities, and a rebuild should write it once. Note that this pass starts with an *empty* cache model rather than inheriting the one the strips left warm — a small missed opportunity, and possibly deliberate, since the leftover list may be a separate draw call.

## `CreateStrips` — flattening, and the winding rule

**Contract** — turns the ordered strips into one flat index stream. With stitching, the strips are joined into a single continuous strip using degenerate triangles; without it, a sentinel value separates them and the separate-strip count is reported. This is where winding is preserved or lost.

**Invariants**

- A triangle strip alternates winding: triangle *n* is wound one way, *n+1* the other, and the hardware compensates. Therefore the winding a triangle is drawn with is determined by its **position in the stream**, and position parity is the only state the flattening has to get right.
- Every triangle in the output must end up with the winding it had in the input. The tool enforces this by emitting a **duplicated index** wherever the parity is wrong. A duplicated index produces a triangle with two identical vertices — zero area, drawn as nothing — and shifts everything after it by one position, flipping the parity. This is the same mechanism as stitching, used for a different purpose.

```text
FUNCTION create_strips(strips, stitch) -> (stream, separate_count)
  FOR EACH strip, in order
    first <- strip.faces[0]
    # Rotate the first triangle so the stream can continue into the second:
    IF the strip has a second face
      rotate first so the vertex NOT shared with face 1 comes first
      IF the strip has a third face
        rotate the remaining two so the vertex shared with face 2 comes last
    # After this, first's last two vertices are exactly the edge face 1
    # joins on, which is what a strip requires.

    IF this is the first strip, or we are not stitching
      IF first's winding does not match position parity
        emit first.v0 once more          # parity fix
    ELSE
      emit first.v0                       # stitch: joins to the previous strip
      IF position parity still disagrees with first's winding
        emit first.v0 again               # stitch AND parity fix
    emit first.v0, first.v1, first.v2

    FOR EACH later face IN strip
      emit the one vertex that face does not share with the previous triangle

    IF stitching
      IF this is not the last strip
        emit the last vertex once more    # stitch: the other half of the join
    ELSE
      emit the sentinel
      separate_count <- separate_count + 1

  IF stitching
    separate_count <- 1
```

**Notes**

- The first-triangle rotation is the step that is easy to get wrong and impossible to skip. The strip's own face order says which triangles follow which, but the *vertex* order inside the first triangle is still the input's, and only one of its three rotations puts the joining edge last. Getting it wrong does not produce a wrong picture — it produces a strip that silently degenerates after one triangle.
- Parity is computed from the number of indices emitted so far, which the sentinel corrupts: the sentinel occupies a position but is not a vertex. The code carries a running count of sentinels emitted and subtracts it before testing parity. A rebuild that returns a list of strips instead of a sentinel-separated stream has no such correction to make — it starts each strip's parity at zero.
- Stitching costs two degenerate triangles per join, sometimes three when the parity also needs fixing. That is the trade the stitch flag names: one draw call for the whole mesh, against a handful of wasted vertices per strip boundary. On hardware where a draw call is expensive, stitching wins; where it is cheap, it does not.
- The final update of the running "last triangle" record assigns its third vertex to itself — a no-op left in the source. The two real shifts above it are what advance the window.

## Vertex-uniqueness helpers

**Contract** — three small queries the passes above are written in terms of, each a comparison of two triangles' vertex triples: the vertex present in the second and absent from the first; the vertex present in both; and, for the walk, the vertex of a triangle that is neither of the last two indices emitted. Each returns a sentinel when there is no such vertex, which the callers read as "these two triangles are not related the way I assumed" — a duplicate or a derailed walk.

**Notes** — Written as unrolled three-way comparisons. Load-bearing only in that the first matching vertex wins, which makes the result deterministic for degenerate triangles where more than one vertex would qualify.

## Unrecovered and dead

- `CACHE_INEFFICIENCY` = 6, `samples` = 10, the scatter increment of 0.1 and its wrap to 0.05, and the two scoring weights of 1.0 are tuning constants with no derivation anywhere in the source.
- The vertex, vector and face structures declared alongside the working records carry positions and normals and are never used. They are the original library's mesh types, kept because the header was copied whole.
- The remaining-triangle counter over a strip range is declared, defined and never called.
- The direction flag chosen when starting the next strip in a chain depends on whether the candidate face shares an edge with the current strip, and carries a comment noting the condition was changed from the other endpoint. What that fixed is not recorded.
