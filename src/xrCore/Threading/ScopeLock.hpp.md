# src/xrCore/Threading/ScopeLock.hpp

> Declares the guard that holds a mutex for the length of a block.

**Needs** — [`ScopeLock.cpp`](ScopeLock.cpp.md) · [`Lock.hpp`](Lock.hpp.md)
**Used by** — [`StackTrace.cpp`](../Debug/StackTrace.cpp.md) · [`Notifier.h`](../Events/Notifier.h.md) · [`FTimer.h`](../FTimer.h.md) · [`ScopeLock.cpp`](ScopeLock.cpp.md) · [`xrDebug.cpp`](../xrDebug.cpp.md) · [`GameSpy_BrowsersWrapper.cpp`](../../xrGameSpy/GameSpy_BrowsersWrapper.cpp.md)
**Tier floor** — T2: scope-bound acquisition and release.

## Purpose

Declares the surface implemented in [`ScopeLock.cpp`](ScopeLock.cpp.md).

## Exported units

- **`ScopeLock`** — constructed around a mutex, which it enters; leaves it when the enclosing scope ends. Not copyable.
