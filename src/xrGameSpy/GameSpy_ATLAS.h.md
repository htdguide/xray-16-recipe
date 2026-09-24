# src/xrGameSpy/GameSpy_ATLAS.h

> Declares the statistics and persistent-storage channel: authorisation, session and
> intent, the report builder, and submission.

**Needs** — [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`login_manager.cpp`](../xrGame/login_manager.cpp.md) · [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md) · [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the surface implemented in [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md), where
the credential model, the report document's shape and what is actually recorded are
written out.

- construction / destruction · `Think` — channel lifetime and one poll per frame.
- `WSLoginProfile` — authorise a logged-in profile for statistics; yields the certificate
  and private-data pair.
- `CreateSession` · `SetReportIntention` · `GetConnectionId` — open a reporting session,
  declare before a match that this participant will report, read the session's id.
- `CreateReport` · `ReportBeginGlobalData` · `ReportBeginPlayerData` ·
  `ReportBeginNewPlayer` · `ReportSetPlayerData` · `ReportAddIntValue` ·
  `ReportAddStringValue` · `ReportEnd` — the builder, in the order it must be called.
- `SubmitReport` — submit under a credential.
- `TryToTranslate` — three overloads, one localization key per failure category.

**Notes**

The builder is a free-function sequence with a report handle threaded through it, wrapped
here as methods on an object that does not own the handle. A rebuild should make the
report a value with a typed builder, which removes the ordering hazard the implementation
twin describes.
