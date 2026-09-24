# src/xrGame/abstract_location_selector.h

> Declares the reusable "pick a good vertex to go to" component whose behaviour is written in [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md).

**Needs** — [`restricted_object.h`](restricted_object.h.md) · [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md)
**Used by** — [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`level_location_selector.h`](level_location_selector.h.md) · [`level_location_selector_inline.h`](level_location_selector_inline.h.md)
**Tier floor** — T3: a declaration; the parameterization over graph and scorer is a language convenience

## Purpose

Declares the location selector: a component that answers *where should I go* by searching
a navigation graph with a caller-supplied scorer, rather than by being told a destination.
It is written once against an unspecified graph, an unspecified vertex identity and an
unspecified scorer, because the same logic serves both the fine level graph and the coarse
game graph. In a rebuild that parameterization is one generic component, not a family of
copies.

Substance — the throttle, the "nothing changed" convention, and the search itself — is in
[`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md).

Exported units:

- `reinit` — reset to the unselected state and bind a graph.
- `set_evaluator`, `set_query_interval` — bind the scorer and the minimum time between
  searches.
- `set_dest_path`, `set_dest_vertex` — optional outputs: where to deposit the winning
  vertex and the shortest path reaching it.
- `select_location`, `actual` — run (or skip) a search; see the inline twin for the
  inverted sense of "failed".
- `get_selected_vertex_id`, `failed`, `used` — queries.
- `before_search` / `after_search` — hooks a concrete selector overrides to prepare and
  clean up around the search.

**Notes** — the file also declares an empty subclass that adds nothing. It is a naming
placeholder from an abandoned split between an abstract and a base selector; a rebuild
should drop it.
