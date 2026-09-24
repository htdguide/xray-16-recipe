# src/xrGame/ui/UIOptConCom.h

> Declares the multiplayer menu's settings: the console variables the host and join screens read
> and write, and the player name that lives outside the game's own files.

**Needs** — [`UIOptConCom.cpp`](UIOptConCom.cpp.md)
**Used by** — [`UIOptConCom.cpp`](UIOptConCom.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIOptConCom.cpp`](UIOptConCom.cpp.md). The type is a
*holder*: it owns the storage that the console variables point at, so the variables outlive any
one screen.

## Exported units

- **The multiplayer settings holder** — one instance, constructed with the menu.
- `Init` — register every multiplayer console variable against this object's storage.
- Two bit-flag vocabularies:
  - **server flags** — dedicated, publicly listed, spectators allowed;
  - **server-list filters** — hide empty, hide full, hide password-protected, hide
    unprotected, hide friendly-fire, hide listen servers.
- `ReadPlayerNameFromRegistry` / `WritePlayerNameToRegistry` — the player name's persistence.
