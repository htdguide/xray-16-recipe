# src/xrGame/space_restriction_abstract_inline.h

> The lazy border build and the accessible-neighbour cache, defined inline because the neighbour walk is a template over the restriction type.

**Needs** — [`space_restriction_abstract.h`](space_restriction_abstract.h.md)
**Used by** — [`space_restriction_abstract.h`](space_restriction_abstract.h.md)
**Tier floor** — T2: neighbour iteration on the navigation mesh

## Purpose

Holds the bodies declared in
[`space_restriction_abstract.h`](space_restriction_abstract.h.md), where their contracts
and algorithms are written. The separation exists because these are templates over the
restriction type — a C++ compilation constraint with no counterpart in a rebuild, which
would use one interface and one definition.

The genericity is not incidental, though, and is worth recording: the same border and
neighbour logic is applied to a *bridge*, to a *composition* and to a *whole in/out pair*,
each of which answers `inside` and `border` differently. A rebuild needs one interface with
those two operations and can then share this code exactly as the original does.

## State

`Stateless.` — see [`space_restriction_abstract.h`](space_restriction_abstract.h.md).

## Contents

Construction (both flags cleared), `border`, `initialized`,
`accessible_neighbour_border`, and the two private steps `accessible_neighbours` and
`prepare_accessible_neighbour_border` that it is built from. All are contracted in the
declaration's twin.
