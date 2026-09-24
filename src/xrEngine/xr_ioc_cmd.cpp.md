# src/xrEngine/xr_ioc_cmd.cpp

> The engine's own command set: the actions that start, quit and reconfigure the session, and the one function that registers every engine-level name.

**Needs** — [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`device.h`](device.h.md) · [`Engine.h`](Engine.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`Render.h`](Render.h.md) · [`xr_input.h`](xr_input.h.md) · [`editor_base.h`](editor_base.h.md) · [`defines.h`](defines.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md)
**Tier floor** — T2: parsing, dispatch and file writing.

## Purpose

Two things. The concrete commands that are *actions on the engine* rather than bound
variables — start a session, disconnect, quit, save the configuration, reset the device —
and `CCC_Register`, the single function that installs every engine-level name into the
console.

Registration being one function rather than a scatter of static constructors is the
decision worth keeping: the command set becomes enumerable, its order is fixed, and the
build configuration's effect on it (which names exist only in a debug build) is readable in
one place.

## `CCC_Register` — the frozen name set

**Contract** — called once from the console's initialization. Registers, in order: the
general actions, the render-state flags, the video and window-mode variables, the sound
variables, the mouse and gamepad tuning, the camera inertia, the renderer and audio-device
selectors, and a handful of network and debug integers. Several groups exist only in
non-shipping builds.

**Invariants** — every name here is **frozen**. The shipped `user.ltx`, the game's own
configuration and every published troubleshooting instruction address them.

The set, grouped by what it configures. Bounds are part of the contract; a value outside
them is rejected, not clamped.

```text
ACTIONS      help  quit  start  disconnect  cfg_save  cfg_load  hide  rs_editor
             vid_restart  snd_restart

FRAME        rs_fps_limit           int [30, 501]      # 501 means uncapped; see device.cpp
             rs_fps_limit_in_menu   int [30, 501]
             rs_always_active       flag               # keep full rate without focus
             rs_v_sync              flag

DISPLAY      vid_monitor            token, built at run time from the monitor list
             vid_mode               token, "WxH (RHz)", per monitor, built at run time
             vid_window_mode        token: windowed / windowed borderless /
                                           fullscreen / fullscreen borderless
             rs_fullscreen          flag   } compatibility shims; see note
             rs_refresh_60hz        flag   }
             renderer               token, built from the renderer modules that loaded

COLOUR       rs_c_gamma, rs_c_brightness, rs_c_contrast   real [0.5, 1.5]
                                        # setting any one re-applies all three to the device

OVERLAYS     rs_stats  rs_fps  rs_fps_graph  rs_cam_pos       flags
             rs_vis_distance        real [0.4, 1.5]
             r__supersample         int [1, 4]
             r__wallmarks_on_skeleton int [0, 1]
             texture_lod            int [0, 4]
             disable_lens_flare     int [0, 1]

SOUND        snd_device             token, built from the enumerated output devices
             snd_volume_eff, snd_volume_music   real [0, 1]
             snd_acceleration, snd_efx, snd_use_float32   flags
             snd_targets            int [4, 256]     # simultaneous sources
             snd_cache_size         int [4, 64]      # megabytes

INPUT        mouse_invert           flag
             mouse_sens             real [0.001, 0.6], default 0.12
             input_exclusive_mode   on/off, never saved
             gamepad_invert_x / _y                    flags
             gamepad_stick_sens_x / _y                real [0.00001, 2]
             gamepad_stick_inner_deadzone / _outer_   real [0, 1]
             gamepad_sensor_sens                      real [0.01, 3]
             gamepad_sensor_deadzone                  real [0.001, 1]
             gamepad_sensors_enable                   flag
             gamepad_cursor_autohide_time             real [0.5, 3]

CAMERA       cam_inert  cam_slide_inert                real [0, 1]

NETWORK      net_dedicated_sleep              int [0, 64]
             sv_console_update_rate           int [1, 100]
             sv_dedicated_server_update_rate  int [1, 1000]
             net_dbg_dump_export_obj / _import_obj   int [0, 1]

DEBUG ONLY   mt_particles  mt_sound  mt_physics  mt_network   # threading toggles
             rs_detail  rs_render_statics  rs_render_dynamics
             rs_render_particles  rs_wireframe  rs_clear_bb  rs_occ_draw
             e_list  e_signal  dbg_str_check  dbg_str_dump  dump_open_files
             vid_bpp  snd_stats*  error_line_count  debug_destroy  debug_show_red_text
```

**Notes** — three groups deserve explanation.

**The render-state flags are shipped-build-gated in two tiers.** The five that disable whole
classes of geometry (`rs_detail`, `rs_render_statics`, `rs_wireframe` and their siblings)
exist in a development build but not a gold one, because they are cheating aids in
multiplayer. The threading and event-inspection commands exist only in a debug build. A
rebuild must keep the first distinction; the second is a convenience.

**One value is read from configuration rather than bound to a name**: the sound occlusion
scale, read once here from the sound section and clamped to [0.1, 0.5]. It is the only
setting in this function that a player cannot change, and there is no recoverable reason
for the asymmetry.

**The gamepad angular-deadzone variable is registered nowhere** — the line is present and
commented out — while the value it would bind still exists. A rebuild should either expose
it or delete it.

## `CCC_Start`

**Contract** — the command that begins a session. Parses up to three parenthesized groups
out of its argument — `server(...)`, `client(...)`, `demo(...)` — and defers a start event
carrying the first two, or a demo-playback event carrying the third. Disconnects from any
current session first. Refuses when neither a client nor a demo is named.

```text
FUNCTION execute(arguments)
  server = the text inside "server(...)",  lowercased
  client = the text inside "client(...)",  lowercased EXCEPT the player name
  demo   = the text inside "demo(...)"

  IF client is empty AND server names a single-player session
    client = "localhost"                    # single player is a client to its own server

  IF client is empty AND demo is empty
    log "cannot start without a client" AND RETURN

  IF a level exists
    defer a disconnect
  IF demo is non-empty
    defer a demo-playback event carrying the demo name
  ELSE
    defer a start event carrying the server and client options
```

**Notes** — **single player is a networked session against a local server.** The engine has
exactly one session path; a single-player game is a server and a client in one process
talking over a loopback address. That is the single most consequential structural decision
in the engine, and this defaulted `localhost` is where it becomes visible.

**The player's name is protected from the lowercasing.** The whole option string is folded
to lower case so that level names, game types and flags match case-insensitively — but the
value of `name=` inside the client options is restored from a copy taken before the fold.
Player names are displayed and compared verbatim, and a game whose players were all
lowercase would be wrong. The restore is done by character index over the pre-fold copy,
bounded by the `name=` key and the next separator; it is fiddly and a rebuild should parse
the options into fields and fold each field according to its own rule instead.

Both actions are **deferred**, and the disconnect is deferred *before* the start, so the
order they run in is the order they were queued. Neither could run inline: this command
executes from inside the console's frame handler, and both destroy the level.

## `CCC_SaveCFG`

**Contract** — writes every registered command's current value to a settings file. The
argument names the file; with no argument, the console's configured one. A relative name is
resolved against the writable application-data root, an absolute one is taken as is; the
extension is forced. On a platform with a read-only attribute, clears it first and refuses
if that fails.

```text
FUNCTION execute(arguments)
  path = arguments, or the console's configured file
  IF path is not absolute
    resolve it against the writable application-data root
  force the extension to the configuration extension
  IF the file exists AND its read-only attribute cannot be cleared
    log a failure AND RETURN
  open for writing
  FOR EACH command IN the console's registry, in name order
    command.save(writer)          # writes "<name> <value>", or nothing for an action
  close AND log success
```

**Notes** — iterating the registry in name order is what makes the written file stable
across runs and therefore diffable. Each command decides its own representation, which is
how `bind` writes a hundred lines from one registration and how an action writes none.

## `CCC_LoadCFG`

**Contract** — reads a file line by line and executes each line as a console command, if a
predicate admits it. Searches three places in order: the writable application-data root, the
installation root, then the name as given. Logs and gives up if none has it. The predicate
is the extension point that `LoadCFG_custom` uses to restrict a file to one command.

**Notes** — the three-root search is what lets one name mean "the user's copy if they have
one, otherwise the shipped one". It is the same precedence the virtual filesystem applies
elsewhere, restated here because this command resolves its own path.

Lines are executed **without recording**, so loading a settings file does not fill the
console's history with a hundred lines.

## `CCC_VidMode` and the compatibility shims

**Contract** — `vid_mode` parses `WxH (RHz)` into the pending display mode. Two or three
numbers are accepted; a refresh rate is optional. Its token list is the mode list of the
*currently selected monitor*, rebuilt whenever the monitor changes. Setting it does not
apply anything — `vid_restart` does.

Two older names are kept working as *derived* flags over the newer variables:

```text
rs_fullscreen  on  -> window mode := fullscreen
               off -> window mode := windowed borderless      # NOT plain windowed
rs_refresh_60hz on -> refresh rate := 60
               off -> refresh rate := 0                        # 0 = let the device choose
```

Each shim's flag is also kept in sync when the *newer* variable is set, so reading either
name gives a truthful answer whichever one was written.

**Notes** — these exist because shipped and third-party configuration files set the old
names. Mapping "not fullscreen" onto *borderless* windowed rather than plain windowed is a
judgement: it is the behaviour a modern player expects from a file that predates the
distinction. A refresh rate of zero meaning "the device picks" is the same convention the
display seam uses.

## `CCC_renderer`

**Contract** — selects the renderer by name from the list of renderer modules that
successfully loaded. Honours one rule: if the renderer was named on the command line, every
later attempt to change it is refused for the rest of the process, including the one the
user's settings file makes. Refuses to save a value that was forced that way.

**Notes** — the engine cannot switch renderers at run time, so this command sets a value
that takes effect on the next launch. The command-line lock exists because the settings
file is executed *after* the command line is parsed, and without the lock a `-renderer`
switch would be silently overwritten by the file it was meant to override.

The refusal is permanent for the process, not scoped to startup, and that is deliberate: a
renderer chosen on the command line is usually chosen because the saved one crashes.

## `CCC_soundDevice`

**Contract** — selects the audio output device by name. Its token list is the enumerated
device list, fetched on every use because enumeration finishes asynchronously during
bring-up. Every operation — execute, status, save — re-fetches and does nothing if the list
is still empty, so a settings file executed before enumeration completes leaves the value
alone rather than clearing it.

## `CCC_Quit`

**Contract** — hides the console and defers a disconnect followed by a quit. Two events, in
that order, so a multiplayer session leaves its server cleanly before the process ends.

## `CCC_Help`

**Contract** — prints every registered name with its current value and its accepted syntax,
then the console's own key map. This is the discoverability mechanism: there is no other
documentation of the command set, and the printed table is generated from the registry, so
it cannot go stale.

## `CCC_VID_Reset`

**Contract** — rebuilds the graphics device against the pending display mode, but only if a
device exists. This is the apply step for every display variable.

## `CCC_ExclusiveMode`

**Contract** — puts the input layer into or out of exclusive mode. Accepts on/off, true/false
and 1/0 — the only command that accepts all three spellings. **Never saved**, because
exclusive input is a troubleshooting setting and persisting a bad value can leave a machine
with no usable input.

## `CCC_Editor`

**Contract** — opens the debug overlay in its fully-visible state. The console's route into
the editor; see [`editor_base.cpp`](editor_base.cpp.md).

## `CCC_SND_Restart`, `CCC_Gamma`, `CCC_Disconnect`, `CCC_DumpOpenFiles`, `CCC_E_Dump`, `CCC_E_Signal`, `CCC_DbgStrCheck`, `CCC_DbgStrDump`

**Contract** — one line each. Restart the sound device. Re-apply all three colour values to
the device after any one changes. Defer a disconnect. List the files the virtual filesystem
currently holds open, at a verbosity the argument selects. Dump the event registry. Signal
one event by name with one text parameter. Verify and dump the interned-string table.
