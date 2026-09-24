# src/xrGameSpy/GameSpy_GP.h

> Declares the account session and its seven operations.

**Needs** — [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`account_manager.cpp`](../xrGame/account_manager.cpp.md) · [`console_commands_mp.cpp`](../xrGame/console_commands_mp.cpp.md) · [`login_manager.cpp`](../xrGame/login_manager.cpp.md) · [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md) · [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) · [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the surface implemented in [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md), where the
account model and each request's shape are written out.

- `Init` · `ShutDown` · construction / destruction — session lifetime.
- `Think` — one poll per frame; completes pending operations.
- `Connect` · `Disconnect` — log in with `(email, nick, password)`; end the session.
- `NewUser` — create an account and its first profile.
- `GetUserNicks` — which profiles an `(email, password)` owns.
- `ProfileSearch` — does a profile exist for this nick, unique nick or email.
- `SuggestUNicks` — free alternatives to a taken unique nick.
- `SetUniqueNick` — claim a unique nick for the logged-in profile.
- `DeleteProfile` — delete the logged-in profile.
- `GetLoginTicket` — the bearer token for this session, fetched after login.
- `TryToTranslate` — a failure code as a localization key.

**Notes**

Every operation takes a completion callback and an opaque caller value, and returns a
submission result rather than a result — the shape a rebuild must preserve even if it
replaces callbacks with something better. The declaration also fixes two bounds a rebuild
inherits from the request shape rather than from any decision here: the login ticket's
fixed length, and the unique nick's maximum length.
