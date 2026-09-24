# src/xrGame/ui_export_script.cpp

> Exports the main menu and its multiplayer screens to the script layer — the surface the
> entire front end is written against.

**Needs** — [`MainMenu.h`](MainMenu.h.md) · [`ui/ServerList.h`](ui/ServerList.h.md) · [`ui/UIMapList.h`](ui/UIMapList.h.md) · [`ui/UIMMShniaga.h`](ui/UIMMShniaga.h.md) · [`ui/UIMessageBoxEx.h`](ui/UIMessageBoxEx.h.md) · [`ui/UISleepStatic.h`](ui/UISleepStatic.h.md) · [`ui/UIScriptWnd.h`](ui/UIScriptWnd.h.md) · [`ScriptXMLInit.h`](ScriptXMLInit.h.md) · [`login_manager.h`](login_manager.h.md) · [`account_manager.h`](account_manager.h.md) · [`profile_store.h`](profile_store.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a binding table.

## Purpose

The game's menus are not built in this engine; they are built in script, from widgets the
engine provides. This file is the list of what the engine provides *for the menu* — as
opposed to the in-game interface, which is exported elsewhere. Every name here is addressed
by a shipped script and every one of them is frozen by the modding-surface criterion.

The interesting content is therefore not any algorithm but the **shape of the boundary**: what
the front end is allowed to do, and what it must ask the engine to do for it.

## State

Stateless.

## `CDialogHolder.script_register`

**Contract** — exports the screen stack: which dialog currently receives input, how to
replace it, how to start and stop the menu, and how to add or remove a dialog from the render
list.

**Notes** — the surface is deliberately redundant. Two names are bound to the same reader and
two more to the same writer, so that scripts written against either vocabulary keep working.
Once a name is in a shipped script it can never be withdrawn; adding an alias is the cheapest
way to rename something. A rebuild must keep every alias, including the pair where the
*top* input receiver's setter actually sets the *main* one — that asymmetry is now behaviour
the scripts depend on.

## `CUIDialogWnd.script_register`

**Contract** — exports the dialog window: show, hide, and the holder it belongs to. Plus the
message box, which is constructible from script, can be initialized from a named layout,
have its text set, and can report a host and a password back — the last two because the
multiplayer connect flow collects them through a message box rather than through a dedicated
screen.

## `CMainMenu.script_register`

**Contract** — the largest block, and four separate surfaces.

**Font alignment** — three values, exported so that script-built widgets can align text.

**Patch download progress** — whether a download is running, its status, its file name and
its fraction. Read-only: the script polls it to draw a progress bar and cannot start or stop
a download through it.

**The main menu itself** — the patch progress object, cancelling a download, validating a
product key, and reading the network build version, the stored product key, the player name
and the demo information. Plus the three account-service handles: login, account and profile
store. This is the front end's whole relationship with the online services — it asks the
engine for a handle and drives the flow from script.

**The menu carousel, the server browser and the map list** — the three complex widgets. The
carousel exposes its three pages by name and the operations to show one. The server browser
exposes refresh (full and quick), connect-to-selected, the filter set as a plain record of
flags, a sort hook and an error callback. The map list exposes loading and saving the
rotation, composing the command line that starts a server, reading the selected game mode,
starting a dedicated server, and setting the preview picture and description.

**Notes** — the division of labour is the decision worth carrying over. The engine owns
*connecting, downloading, validating and enumerating*; the script owns *everything the
player sees and every decision about flow*. The server browser does not draw a server list —
it produces one and takes a sort function back. A rebuild that moves any of these
responsibilities across the line breaks shipped menu scripts, which are a product in their
own right.

The connection-error path is exported as a **callback type**, not as a return value, because
connecting is asynchronous and the script must be told later.

## `CUISleepStatic.script_register`

**Contract** — exports the sleep-screen widget, constructible from script. No methods: the
script positions it as a window and the widget animates itself.

## Notes

The file also reaches out to touch the script XML initializer for no reason other than to
keep the linker from discarding it — the initializer is only ever reached through data, so
nothing references it. That is an artifact of separate compilation and does not survive into
a rebuild as itself; what survives is the requirement that **a class reachable only by name
from data must still be present in the program**, which every language solves its own way
and none solves by accident.
