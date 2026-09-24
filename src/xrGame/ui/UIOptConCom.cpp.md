# src/xrGame/ui/UIOptConCom.cpp

> Registers the multiplayer menu's console variables, and makes the player name a special case
> that persists outside the game's own configuration.

**Needs** — [`UIOptConCom.h`](UIOptConCom.h.md) · [`UICDkey.h`](UICDkey.h.md) · [`gametype_chooser.h`](../../xrServerEntities/gametype_chooser.h.md) · [`RegistryFuncs.h`](../RegistryFuncs.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [`xrEngine/xr_ioc_cmd.h`](../../xrEngine/xr_ioc_cmd.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`UIOptConCom.h`](UIOptConCom.h.md)
**Tier floor** — T3.

## Purpose

Every widget on the multiplayer menu is a *settings control* bound to a console variable by
name. This file is where those variables come into existence. It is therefore the authoritative
list of what the multiplayer menu can express, and — because shipped configuration files and
user settings address variables by name — that list is frozen.

## The game-mode token table

**Contract** — the four multiplayer modes, each pairing a localization identifier with the mode's
own numeric identity: deathmatch, team deathmatch, artefact hunt, capture the artefact. This is
the table the mode pickers on the host screen and in the admin menu both read, and the one
[`UIMapList`](UIMapList.cpp.md) compares display text against.

## `Init`

**Contract** — read the player name from the platform's user store, then register the variables:

```text
  mm_net_player_name               text, capped at the account-name length, saved externally
  mm_net_srv_maxplayers            int   2..32, default 32
  mm_net_srv_gamemode              token from the game-mode table, default deathmatch
  mm_mm_net_srv_dedicated          flag in the server mask
  mm_net_con_publicserver          flag in the server mask
  mm_net_con_spectator_on          flag in the server mask
  mm_net_con_spectator             int   1..32, default 20
  mm_net_srv_reinforcement_type    text, default "reinforcement"
  mm_net_weather_rateofchange      real  0..100, default 1
  mm_net_srv_name                  text, default "Stalker"
  mm_net_filter_{empty,full,pass,wo_pass,wo_ff,listen}   flags in the filter mask, all set
```

**Notes**

- **The two masks are the interesting part.** Several named variables write different bits of
  one integer, so a screen can present six independent check boxes that the code reads as one
  value. The server-list filter mask starts with *every* bit set, meaning "show everything",
  and the server mask starts empty.
- The dedicated-server variable's name carries a duplicated prefix. It is wrong and it is
  frozen: shipped configuration writes it that way, so a rebuild that corrects the spelling
  stops reading the player's saved setting.
- The spectator count's default of twenty against a maximum of thirty-two, and the player cap's
  default of thirty-two, are the shipped values with no discoverable derivation.

## The player-name variable

**Contract** — a text variable that differs from the others in three ways: it truncates its
value to the account service's maximum nickname length, it refuses an empty argument, it writes
itself back to the platform's user store on every change, and **it does not save itself to the
game's configuration file**.

**Notes** — the name lives outside the game's own settings because it was the account identity
for a matchmaking service, and had to survive a reinstall and be shared between installs. The
service is gone; the storage decision remains, and a rebuild is free to move the name into the
ordinary settings file — at the cost of not reading what an existing installation saved.

The length cap is the *service's* nickname limit, not a display limit. With the service dead it
constrains nothing but is preserved because names already stored obey it.
