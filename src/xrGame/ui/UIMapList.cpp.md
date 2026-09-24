# src/xrGame/ui/UIMapList.cpp

> The map-rotation editor: two lists and four buttons that compose an ordered rotation, and
> the place where every choice on the host screen is finally turned into one command string.

**Needs** — [`UIMapList.h`](UIMapList.h.md) · [`UIMapInfo.h`](UIMapInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UICDkey.h`](UICDkey.h.md) · [`UIGameCustom.h`](../UIGameCustom.h.md) · [`game_base.h`](../game_base.h.md) · [`xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`xrUICore/ComboBox/UIComboBox.h`](../../xrUICore/ComboBox/UIComboBox.h.md) · [`xrUICore/SpinBox/UISpinText.h`](../../xrUICore/SpinBox/UISpinText.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`UIMapList.h`](UIMapList.h.md)
**Tier floor** — T2: it writes a file and arranges a process launch; otherwise list editing.

## Purpose

The host screen's centre. Everything else on that screen — the game-mode picker, the weather
picker, the preview picture, the description panel — is *injected into* this editor rather than
owned by it, because the editor is the only thing that knows which map is current and it is
the thing that finally serialises the whole screen into a command.

That injection is the file's shape: five setters, each taking a widget this window will read
but never free. A rebuild that instead gives the editor a reference to the host screen collapses
all five.

## State

```text
RECORD MapRotationEditor EXTENDS Window
  available   : ListBox      # element "list_1": every map legal for the current game mode
  rotation    : ListBox      # element "list_2": the chosen sequence, in order
  frames, headers            # decoration only
  btn_left, btn_right        # remove from / add to the rotation
  btn_up, btn_down           # reorder within the rotation

  # injected, not owned:
  weather_selector : optional<ComboBox>
  mode_selector    : optional<Window>    # a drop-down OR a stepper; see below
  map_pic          : optional<Static>
  map_info         : optional<MapInfoPanel>
  srv_params       : text                # the rest of the host screen's settings, pre-joined

  command : text                         # scratch: the assembled command line
```

**Invariants**

- **A row's payload is its index into the roster for the current game mode**, stored on the
  row; the row's *text* is the map's localized name. Both lists carry it, and a row copied
  from the available list to the rotation copies the payload with it. The roster is re-read
  through that index every time, so the editor never holds a map record.
- The rotation may contain the same map more than once. Nothing deduplicates, and that is
  deliberate: a rotation is a sequence.
- Changing the game mode invalidates every index, so the available list is rebuilt and the
  rotation is **re-resolved by name** — see below.
- The first row of the rotation is the map the session starts on. Everything after it is only
  meaningful to a server that reads the rotation file.

## `GetCurGameType` — reading the mode picker

**Contract** — answer which of the four multiplayer game modes the injected mode picker
currently shows. The picker may be either a drop-down or a text stepper, and the two are
distinguished at the point of use.

**Notes** — the mode is recovered by **comparing display text**, not by reading a value: the
drop-down's current text is compared against the four modes' localized names, and the
stepper's against their untranslated token names. This is fragile in exactly the way it looks
— two modes with the same localized string would be indistinguishable, and an unrecognised
string is fatal — and it exists because the two picker widgets have no common typed value. A
rebuild gives the picker a token value and deletes this function's body.

## `UpdateMapList` — a mode change

**Contract** — rebuild the available list from the roster for the new mode, each row tagged
with its roster index, then rebuild the rotation from its own *row texts*: for each text,
find a row with that text in the new available list and, if there is one, re-add it with that
row's new index. A rotation entry whose map does not exist in the new mode silently vanishes.

```text
FUNCTION on_mode_change(mode)
  available.clear()
  FOR i, entry IN roster_for(mode)
    row <- available.add_text_row(localize(entry.map_name));  row.payload <- i

  IF rotation IS empty THEN RETURN
  kept <- [rotation.text_of(i) FOR i IN 0..rotation.size-1]   # snapshot before clearing
  rotation.clear()
  FOR EACH name IN kept
    src <- the row of available whose text equals name
    IF src EXISTS THEN rotation.add_text_row(name).payload <- src.payload
```

**Notes** — the re-resolution is by *localized display text*, which is the same weakness as the
mode read-out and for the same reason: the row's payload is only valid within one roster, so
text is the only thing that survives the switch. A rebuild that stores the map's own name on
the row resolves against the roster directly and keeps working in any language.

## The four buttons, and the double click

**Contract** — the right button copies the highlighted available row into the rotation, payload
and all; the left button removes the highlighted rotation row; up and down move the highlighted
rotation row one place. A double click on either list does what the button pointing away from
it does, so double-clicking an available map adds it and double-clicking a rotation entry
removes it. A single click on an available row refreshes the preview and description.

## `OnListItemClicked` — the preview

**Contract** — for the clicked available row, bind the preview picture to that map's preview
texture, or to the shipped placeholder when the file is absent, preserving the picture's
authored sub-rectangle across the swap; and rebuild the description panel for that map's name
and version. Identical in mechanism to the admin screen's map preview.

## `GetCommandLine`

**Contract** — assemble the single command that starts a server and joins it, from the first
row of the rotation, the current game mode, the injected server parameters, the map's version
and the selected weather's start time; then append a local client connection carrying a player
name. Returns nothing when the rotation is empty. The name is the caller's if given, otherwise
the stored profile name, otherwise the operating-system user name, otherwise the machine name.

```text
FUNCTION command_line(player_name) -> optional<text>
  first <- rotation.row(0);  IF none THEN RETURN none
  entry <- roster_for(current_mode())[first.payload]

  server <- "start server(" + entry.map_name + "/" + short_name_of(current_mode())
                            + srv_params + "/ver=" + entry.map_ver
  IF a weather is selected THEN server <- server + "/estime=" + that weather's start time
  server <- server + ")"

  name <- player_name IF non-empty
          ELSE stored_profile_name() IF non-empty
          ELSE os_user_name() IF non-empty
          ELSE machine_name()
  RETURN server + " client(localhost/name=" + name + ")"
```

**Notes** — the whole host screen ends here as a **string**. Map, mode, every tuned setting,
version and weather are joined into one line that the console parses. That is the frozen
interface between the menu and the game layer, and it is also why `srv_params` arrives
pre-joined from elsewhere: nobody owns the whole grammar. A rebuild may pass a record instead,
but must keep the command form too, because a player can type it.

The weather contributes a *start time*, not a weather name: choosing weather means choosing the
hour the day cycle starts at, exactly as on the admin screen.

## `LoadMapList` and `SaveMapList`

**Contract** — the load fills the injected weather drop-down from the shared weather table and
selects the first entry; it refuses to run with no drop-down injected. The save writes the
rotation to a fixed file under the writable data root, one `add map` command per entry, naming
each map and its version — and **deletes the file when the rotation has one entry or none**,
since a single map is not a rotation.

**Notes** — the file is a list of console commands, not a data format. It is read back by a
dedicated server at startup by executing it. That is the same "everything is a command"
decision as the command line above, and it means the rotation file is trivially hand-editable,
which is how servers are actually configured.

## `StartDedicatedServer`

**Contract** — record the path to this program, the working directory, and a parameter string
combining the headless flags with the assembled command line, then quit. The recorded values
are what the process launches *after* it has exited.

**Notes** — the server is started by relaunching the same executable with different flags,
after this one is gone, because the two cannot share the data root's write locks. The
indirection through "launch this on exit" belongs to the engine layer; this file only fills it
in. The flags disable sound and request the dedicated path; the trailing bare dash separates
the flags from the command that follows.

## `InitFromXml`

**Contract** — apply the given subtree to this window and its ten children by fixed relative
element names: two headers, two frames, two lists, four buttons. The subtree's path is a
parameter, so several host screens can embed the editor at different places in their document.
