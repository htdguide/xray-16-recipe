# src/editors/xrWeatherEditor/resource.h

> Names one embedded image: the eyedropper cursor the three-dimensional view shows when a colour can be sampled.

**Needs** — _(none)_
**Used by** — [`window_view.cpp`](window_view.cpp.md)
**Tier floor** — T4: a generated table of numbers naming embedded resources.

## Purpose

A generated index of the application's embedded resources. It holds exactly one real
entry — the identifier of the colour-picker cursor loaded by
[`window_view.cpp`](window_view.cpp.md) — and a block of bookkeeping the resource editor
uses to allocate the next identifier in each class.

## State

```text
CONSTANT picker_cursor = 105     # the eyedropper cursor image
```

**Notes** — Embedding a cursor as a numbered resource is an artifact of the platform's
executable format, and the *number* is not load-bearing — nothing outside this file and
its one consumer knows it. What survives a rebuild is that **the application ships one
custom cursor and it means "click to sample a colour"**; a rebuild loads it by name from
its own asset system.

The bookkeeping block is pure tooling state and carries no information about the program.
