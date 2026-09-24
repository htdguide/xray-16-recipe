# src/xrGame/ui/UIMPChangeMapAdm.cpp

> The administrator's map-change page: lists the maps the running game type allows, previews
> the highlighted one, and turns a confirmation into a remote `change level` command.

**Needs** — [`UIMPChangeMapAdm.h`](UIMPChangeMapAdm.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIGameCustom.h`](../UIGameCustom.h.md) · [`Level.h`](../Level.h.md) · [`xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIMPChangeMapAdm.h`](UIMPChangeMapAdm.h.md)
**Tier floor** — T3: list manipulation, a file-existence probe and string assembly.

## Purpose

One of the three pages of the remote-administration screen. Its whole job is to present the
*server's* map roster — not an arbitrary level list — and to let one entry of it be chosen.
The roster is a shared helper the multiplayer game layer already maintains, keyed by game
type, so this page never discovers maps itself.

It exists as a separate file because it is the only admin page with a preview asset and a
version concept; the other two are pure command panels.

## State

```text
RECORD ChangeMapPage EXTENDS Window
  map_pic     : Static     # preview image, element name "map_pic"
  map_frame   : Static     # decorative border, element name "map_frame"
  map_version : Static     # element name "map_ver_txt"
  lst         : ListBox    # element name "list"
  btn_ok      : Button     # element name "btn_ok"
```

**Invariants**

- Row *i* of the list is entry *i* of the roster for the current game type. Nothing else
  establishes the correspondence — selection is carried by row index, not by a stored key —
  so a rebuild that sorts or filters the list must carry the roster index on the row instead.
- The preview picture's sub-rectangle is authored in the layout and must survive a texture
  swap. Rebinding a texture resets it, so the page saves and restores it around every swap.

## `Init`

**Contract** — configure this window and its five children from the sections
`change_map_adm`, `…:map_frame`, `…:map_ver_txt`, `…:map_pic`, `…:list` and `…:btn_ok` of the
already-open layout document, then fill the list once. The element names are frozen by the
shipped document.

## `FillUpList`

**Contract** — clear the list and add one enabled text row per map in the roster for the
current game type, the row's text being the map's name passed through the localization
string table. Allocates; does not block.

**Notes** — the map *name* is used as a localization identifier. A map whose name has no
table entry displays its raw name, which is the intended fallback for user-installed maps.

## `OnItemSelect`

**Contract** — for the highlighted row, set the version label to the roster entry's version
in square brackets (the literal `unknown` when the roster carries no version), and bind the
preview picture to the texture named by the map. Does nothing when nothing is highlighted.

```text
FUNCTION on_item_select()
  idx <- lst.selected_index()
  IF idx IS none THEN RETURN

  entry <- roster_for(current_game_type())[idx]
  version_label.set_text("[" + (entry.version OR "unknown") + "]")

  saved_rect <- map_pic.texture_rect()        # the layout's authored sub-rectangle
  candidate  <- "intro/intro_map_pic_" + entry.name
  IF virtual_filesystem.exists(texture_root, candidate + ".dds") THEN
    map_pic.bind_texture(candidate)
  ELSE
    map_pic.bind_texture("ui/ui_noise")       # shipped placeholder for maps with no preview
  map_pic.set_texture_rect(saved_rect)        # binding reset it to the whole page
```

**Notes** — the existence probe names the file with its extension while the bind names it
without: the picture binds a *logical* texture name that the icon registry resolves, whereas
the probe asks the virtual filesystem about an actual file. A rebuild with a single naming
scheme collapses the two.

## `OnBtnOk`

**Contract** — when the highlighted row is within the roster, issue the remote-administration
console command that changes the level, naming the map and its version, and then ask the
parent dialog to close. Bounds-checks the index; a highlighted row outside the roster does
nothing.

**Notes** — the action reaches the server as *text on the console*, prefixed to mark it as a
remote-admin command rather than a local one. That indirection is the whole mechanism of this
screen: no admin page calls a game function directly, and a rebuild that replaces the console
with typed messages must still route these through whatever channel carries admin authority,
because authority is checked at the receiving end, not here.

## `SendMessage`

**Contract** — a selection change in the list refreshes the preview; a click on the confirm
button commits. Every other notification is ignored.
