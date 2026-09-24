# src/xrGame/space_restriction_manager_inline.h

> Stamps an entity's restriction border onto the level graph ahead of a path search, given the movement about to be planned.

**Needs** — [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`space_restriction.h`](space_restriction.h.md)
**Used by** — [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md)
**Tier floor** — T3: a lookup and a forward

## Purpose

One operation, separated out because it is generic over the pair of arguments describing an
intended movement — a start and a destination, given either as positions or as navigation
vertices. A rebuild should fold it into the type.

## `add_border`

**Contract** — look up the entity's restriction and, if it has one, stamp its border on the
level graph as a search barrier. A no-op for an unrestricted entity. The two movement
arguments are passed through; the shipped build ignores them.

**Notes** — this is the entry point the movement system calls immediately before running a
path search, paired with a clear immediately after. The entity's restriction must not be
changed between the two, which the edit operations assert.

The movement arguments exist for the compiled-out policy that stamps only the forbidden
volumes an entity is actually approaching; see
[`space_restriction.cpp`](space_restriction.cpp.md).

## `restrictions` (checked builds)

**Contract** — the whole map of shared restrictions, for the debug renderer that draws every
live border. Debug-only.
