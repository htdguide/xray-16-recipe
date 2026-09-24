# src/xrGame/game_cl_mp_messages_menu.h

> A fragment of a class body, textually pasted into the multiplayer client's declaration: the quick-speech menu's members and methods.

**Needs** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md)
**Used by** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md)
**Tier floor** — T3: a declaration fragment

## Purpose

This is not a header in the ordinary sense. It has no include guard and it does not open a
scope: it is a run of member declarations, complete with access specifiers, pasted textually
into the middle of the multiplayer client class in
[`game_cl_mp.h`](game_cl_mp.h.md). Nothing else includes it, and it is not independently
compilable.

The reason is organisational — the quick-speech feature was kept in its own pair of files —
and it is **purely incidental**. A rebuild declares these members with the rest of the class,
or better, makes the speech menu a component the class holds rather than a set of members it
absorbs.

What it declares, implemented in
[`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md):

- the list of loaded speech menus;
- `AddMessageMenu` / `LoadMessagesMenu` — build one menu from a configuration section, and
  build the whole set;
- `DestroyMessagesMenus` / `HideMessageMenus` — release the sounds, and close any open menu;
- `OnMessageSelected` — the local player picked a phrase; send it;
- `OnSpeechMessage` — somebody said something; show it and play it.
