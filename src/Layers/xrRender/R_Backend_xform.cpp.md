# src/Layers/xrRender/R_Backend_xform.cpp

> The transform cache: three matrices the engine sets, four products it derives, and the rule that a derived product is recomputed the moment any of its inputs moves.

**Needs** — [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_xform.h`](R_Backend_xform.h.md)
**Tier floor** — T2: it is matrix arithmetic and a cache. Nothing here is device-facing; the only reason it lives beside the backend is that it writes into the backend's named-constant cache.

## Purpose

Shipped shader sources ask for transforms by name — `m_W`, `m_V`, `m_P`, `m_WV`, `m_VP`, `m_WVP`, `m_invW` — and the seven names are a frozen part of the material contract. Three of them are *inputs*: the world matrix changes per object, the view matrix per camera, the projection per pass. The other four are products of those three. This module decides **when** each product is recomputed and **when** its value reaches the device, so that no shader ever reads a stale composite and no composite is multiplied out more often than it has to be.

The whole file is one idea with three entry points. It is a separate file because the set of derived transforms is a fixed, small, named vocabulary that the rest of the renderer should not be able to add to informally.

## State

```text
RECORD TransformCache
  # the three inputs
  world       : matrix
  view        : matrix
  projection  : matrix

  # the four derived products
  world_view          : matrix    # view applied after world
  view_projection     : matrix    # projection applied after view
  world_view_proj     : matrix    # projection after view after world
  inverse_world       : matrix    # world-to-local

  inverse_world_valid : bool      # the only lazy product

  # one binding per name: where in the current pass this transform lives,
  # or none when the current pass does not ask for it
  loc_w, loc_invw, loc_v, loc_p, loc_wv, loc_vp, loc_wvp : optional<ConstantLocation>
```

**Invariants**

- `world_view`, `view_projection` and `world_view_proj` are **eagerly** correct: after any of the three setters returns, all three products agree with the three inputs. There is no dirty flag on them, because a composite is needed by almost every pass and the multiply is cheaper than the branch plus the risk of an unflagged read path.
- `inverse_world` is the exception and is **lazy**, guarded by its own validity flag: setting the world matrix marks it stale and only inverts if some pass has actually bound `m_invW`. A general matrix inverse is an order of magnitude more expensive than the three composite multiplies put together, and the overwhelming majority of passes never ask for it.
- A binding of `none` means "the current pass does not use this transform". Writing through a `none` binding is a silent no-op, so the setters can publish everything unconditionally and let the pass take what it wants. This is the same "set everything, the pass ignores the rest" convention the whole constant system runs on.
- All seven bindings and the world-view composite are only meaningful relative to the *currently installed pass*. They must be dropped whenever the pass changes — see `unmap`.

## `set_W` — set the world transform

**Contract** — Takes the new object-to-world matrix. Recomputes world-view and world-view-projection, publishes whichever of `m_W`, `m_WV`, `m_WVP` the current pass bound, invalidates the inverse and re-derives it only if `m_invW` is bound. Allocates nothing; called once per drawn object, so it is the hottest of the three.

```text
FUNCTION set_world(m)
  world           = m
  world_view      = compose(view, world)        # rotation+translation only: the
                                                # world and view matrices are affine,
                                                # so the fourth row is known and skipped
  world_view_proj = compose(projection, world_view)   # full 4x4: projection is not affine

  publish(loc_w,   world)
  publish(loc_wv,  world_view)
  publish(loc_wvp, world_view_proj)

  inverse_world_valid = false
  IF loc_invw IS present THEN apply_inverse_world()

  notify_fixed_function_slot(world_slot, m)
```

**Notes** — The world-view product is composed with the affine-only multiply and the world-view-projection with the general one. That asymmetry is not a micro-optimisation detail to be discarded: it encodes the invariant that *object and camera transforms are rigid-plus-scale and the projection is not*. A rebuild that composes everything generally is correct but pays for a row it knows the value of; a rebuild that composes everything affinely is silently wrong at the projection.

## `set_V` — set the view transform

**Contract** — Takes the new world-to-camera matrix, recomputes *all three* composites (world-view, view-projection, world-view-projection) and publishes `m_V`, `m_VP`, `m_WV`, `m_WVP`. Note that `m_invW` is untouched: the inverse world does not depend on the camera. Called once per camera setup, a handful of times per frame.

## `set_P` — set the projection transform

**Contract** — Takes the new camera-to-clip matrix, recomputes view-projection and world-view-projection, publishes `m_P`, `m_VP`, `m_WVP`.

**Notes** — The projection is pushed to the fixed-function transform slot *unconditionally*, where world and view are pushed with the same call but the surrounding code has always treated projection as the one that must not be skipped. On every device this engine still ships against, that slot is a counter increment and nothing else — the fixed-function pipeline it addressed is gone. A rebuild drops the slot entirely and keeps only the named-constant publication; what survives is the statistic, which counts transform changes for the performance overlay.

## `apply_invw` — the lazy inverse

**Contract** — Precondition: `m_invW` is bound. Inverts the world matrix if the cached inverse is stale, marks it fresh, and publishes it. Called from `set_W` and from the moment `m_invW` is bound by a newly installed pass.

```text
FUNCTION apply_inverse_world()
  IF NOT inverse_world_valid
    inverse_world = invert(world)     # general inverse of an affine transform
    inverse_world_valid = true
  publish(loc_invw, inverse_world)
```

**Notes** — Binding `m_invW` is itself a trigger: when a pass declares it, the binder immediately calls this, so a pass that binds the inverse *after* the world matrix was last set still gets the right value. Without that, the inverse would be one object stale for the first object drawn with each such pass. This is the general shape of every constant binder in the system — binding a name both records where it lives and pushes the current value into it.

## `unmap` — forget every binding

**Contract** — Clears all seven bindings to `none`. Called whenever the installed pass changes or the whole command list is invalidated. Does not touch the matrices themselves: the *values* remain correct across a pass change, only the places to put them are gone.

**Notes** — This is the single most dangerous thing in the file to get wrong. A location is an offset into a specific pass's constant layout; carrying one across a pass change writes a matrix over whatever that offset means in the new pass. The command list's invalidation path calls this for exactly that reason, and the comment in the source is explicit that it exists because the device's constant buffers are themselves unmapped at that moment.

## Construction

**Contract** — All seven matrices start as identity, all bindings as `none`, and the inverse is marked *valid* — which is consistent, since the inverse of the identity is the identity.
