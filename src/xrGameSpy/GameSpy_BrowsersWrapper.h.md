# src/xrGameSpy/GameSpy_BrowsersWrapper.h

> Declares the multi-master-list aggregate and the state accumulator it reduces list
> states with.

**Needs** — [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md) · [`GameSpy_Browser.h`](GameSpy_Browser.h.md)
**Used by** — [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) · [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the surface implemented in
[`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md). Two types:

- `CGSUpdateStatusAccumulator` — collects a sequence of list states and reduces it either
  optimistically (the best anyone reported) or pessimistically (the worst), plus the
  predicate that decides which states count as working.
- `CGameSpy_BrowsersWrapper` — several master lists presented as one: subscribe and
  unsubscribe to change notifications, refresh everything, per-server detail fetch,
  indexed access, per-field reads, and one poll per frame.

**Notes**

The declaration deliberately mirrors the single-list type's method-for-method, so the two
are interchangeable at a call site. That was worth doing when there was one list and
became the way to add two more without touching the UI; a rebuild should make the
single-list case just an aggregate of one and delete the duplication.
