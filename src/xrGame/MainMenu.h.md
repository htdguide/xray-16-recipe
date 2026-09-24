# src/xrGame/MainMenu.h

> Declares the main menu overlay implemented in [`MainMenu.cpp`](MainMenu.cpp.md), and the small record the patch downloader reports progress through.

**Needs** — [`MainMenu.cpp`](MainMenu.cpp.md) · [`UIDialogHolder.h`](UIDialogHolder.h.md) · [`DemoInfo.h`](DemoInfo.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrEngine/IInputReceiver.h`](../xrEngine/IInputReceiver.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md) · [`Level_start.cpp`](Level_start.cpp.md) · [`MainMenu.cpp`](MainMenu.cpp.md) · [`account_manager.cpp`](account_manager.cpp.md) · [`account_manager_console.cpp`](account_manager_console.cpp.md) · [`autosave_manager.cpp`](autosave_manager.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`player_account.cpp`](player_account.cpp.md) · [`profile_store.cpp`](profile_store.cpp.md) · _and 4 more_
**Tier floor** — T3: a declaration, plus one small progress record

## Purpose

Declares `CMainMenu`. The declaration's own content is the *set of roles* the menu plays at
once, which is worth naming because it is what a rebuild must provide: it is the persistent
layer's main-menu interface, an input receiver, a render-list participant, a dialogue holder,
a window-callback target, and a listener for both device reset and user-interface reset. Each
role is a different subsystem's way of reaching it, and the menu's job is to reconcile them.

Substance is in [`MainMenu.cpp`](MainMenu.cpp.md).

Exported units:

- `CMainMenu` — the overlay.
- `Patch_Dawnload_Progress` — a small record the patch downloader publishes into: in-progress
  flag, fraction, status text and file name, with read-only accessors exposed to the script
  layer. Spelled as in the original.
- `EErrorDlg` — the error kinds. Its enumerator values index the message-box template table in
  the implementation, so the order is load-bearing.
- `Activate` / `IsActive` / `CanSkipSceneRendering` / `IgnorePause` — the suspension contract
  with the device.
- `OnFrame` / `OnRender` / `OnRenderPPUI_query` / `OnRenderPPUI_main` / `OnRenderPPUI_PP` —
  the per-frame and drawing entry points.
- The `IR_*` family — input, all inert while the menu is inactive.
- `Screenshot` / `RegisterPPDraw` / `UnregisterPPDraw`.
- `SetErrorDialog` / `GetErrorDialogType` / `CheckForErrorDlg` / `OnSessionTerminate` /
  `OnLoadError` / `OnPatchCheck` / `Show_CTMS_Dialog` / `Hide_CTMS_Dialog` — the deferred
  message-box machinery.
- `Show_DownloadMPMap` / `OnDownloadMPMap` / `OnDownloadMPMap_CopyURL` — offering a missing
  multiplayer map's download address, opened in the platform browser or copied to the
  clipboard.
- `SetNeedVidRestart` / `OnDeviceReset` / `OnUIReset` — the two deferred restarts.
- `IsCDKeyIsValid` / `ValidateCDKey` / `GetPlayerName` / `GetCDKeyFromRegistry` / `GetGSVer` /
  `GetAccountMngr` / `GetLoginMngr` / `GetProfileStore` / `GetGS` / `GetPatchProgress` /
  `CancelDownload` — the matchmaking and account surface, all behind the dead seam.
- `GetDemoInfo` — the demo file summary reader.
- `SwitchToMultiplayerMenu`, `DestroyInternal`, `ReloadUI`, `FillDebugTree`.
- `MainMenu()` — the free function returning the single instance, which lives on the
  persistent game layer rather than here.

The class also registers itself with the script binding layer, exporting the dialogue holder,
dialogue window and window types.
