# src/xrGameSpy/GameSpy_Full.h

> Declares the client-side facade over all five online services.

**Needs** — [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`account_manager_console.cpp`](../xrGame/account_manager_console.cpp.md) · [`login_manager.cpp`](../xrGame/login_manager.cpp.md) · [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md), and names
the five services a consumer can reach through it.

- construction / destruction — brings the whole online layer up and down.
- `GetGameSpyAvailable` · `GetGameSpyHTTP` · `GetGameSpyBrowser` · `GetGameSpyGP` ·
  `GetGameSpyATLAS` — hand out the owned services. Each is a plain accessor.
- `Update` — one poll per frame; yields the status the menu renders.
- `CoreThink` — advance the shared service core for a bounded time, used by screens that
  wait on a long operation outside the frame loop.

**Notes** — every consumer of this module reaches its service through one of the five
accessors, which makes this type the chapter's single injection point. A rebuild that
wants a null online layer replaces *this one object* and touches nothing else.
