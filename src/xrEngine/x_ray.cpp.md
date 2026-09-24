# src/xrEngine/x_ray.cpp

> Process bring-up in order, the platform event pump that feeds the frame loop, and the shutdown that reverses it.

**Needs** — [`x_ray.h`](x_ray.h.md) · [`Engine.h`](Engine.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`device.h`](device.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`Text_Console.h`](Text_Console.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`LightAnimLibrary.h`](LightAnimLibrary.h.md) · [`AccessibilityShortcuts.hpp`](AccessibilityShortcuts.hpp.md) · [`embedded_resources_management.h`](embedded_resources_management.h.md) · [`xr_input.h`](xr_input.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`x_ray.h`](x_ray.h.md)
**Tier floor** — T1: the bring-up order is a dependency graph between device, filesystem, console and game module, and several steps run concurrently against it.

## Purpose

This is where the process becomes an engine. The constructor brings every subsystem up in
the one order that works; `Run` is the outer loop that pumps the platform's event queue and
calls the frame; the destructor tears everything down in the reverse order.

The bring-up order is the whole content of this file. Almost every line is "X must exist
before Y", and those edges are the load-bearing decisions — get one wrong and the engine
either crashes or silently loses a setting. Read the order as a specification, not as a
narration.

## State

```text
RECORD Application
  splash_window  : optional<Window>    # a borderless image shown during bring-up
  splash_thread  : Thread
  should_exit    : bool (atomic)       # tells the splash thread to stop
  game_module    : optional<GameModule>
  presence       : optional<PresenceClient>   # optional, Windows-only; see note
```

And three process-wide flags the whole engine reads, which say **which of the three games'
conventions are in force**:

```text
shadow_of_chernobyl_mode : bool
clear_sky_mode           : bool
call_of_pripyat_mode     : bool        # exactly one is set
```

Plus a deferred relaunch: an application path, an argument string and a working directory
which, if set at shutdown, are spawned as a new process after this one has released
everything. That is how the engine restarts itself into a different renderer.

## Bring-up, in order

**Contract** — constructs the application from a command line, a game module and the
available renderer modules. Blocking; ends with a window, a graphics device, a console, a
sound device and, if a game module was supplied, a persistent layer that has been told the
application started. Fatal errors abort with a message box rather than returning.

```text
FUNCTION construct(command_line, game, renderer_modules)
  name the current thread; mark the startup span for the profiler
  IF the command line asks for a dedicated server
    set the dedicated-server flag                # read by nearly everything below
  install the crash handler and the debug facilities
  initialize the windowing layer: video always, gamepads unless suppressed
  IF not a dedicated server
    disable the operating system's accessibility key shortcuts    # see note
  IF not suppressed, raise the splash image on its own thread
  turn the platform's text-input mode off        # it starts on, and we do not want it

  SPAWN  input_task:  create the input layer, capturing the pointer unless suppressed
  SPAWN  sound_task:  enumerate the audio output devices

  core.initialize(command_line, optional filesystem-description override)
        # mounts the virtual filesystem. NOTHING above this line may read a game file.

  init_settings()                                # the configuration files; see below
  IF the configuration declares a non-native input locale
    replace the detected player and computer names with neutral ones   # see note

  device.initialize_overlay_toolkit()
  device.enumerate_video_modes()
  AWAIT input_task                               # the console needs the input layer
  init_console()                                 # registers the entire command set

  engine.initialize(game, renderer_modules)      # selects and loads a renderer
  device.initialize()                            # window + graphics device
  console.on_device_initialized()

  run_user_script()                              # default bindings, then the user's settings
  initialize_presence()

  AWAIT sound_task
  sound.create()                                 # opens the device the settings named

  IF the command line carries a "start" or a "load" argument
    execute it as a console line

  SPAWN light_anim_task: load the shared light-animation bank
  device.create()                                # the swap chain and the render targets
  AWAIT light_anim_task

  IF a game module was supplied
    persistent = game.create_persistent()        # REQUIRED to succeed
    persistent.on_app_start()                    # loads the weather, scans the levels
  ELSE
    show the console                             # with no game there is nothing else
```

**Notes** — the edges that matter, and why.

**The virtual filesystem is mounted in the middle, not first.** Everything above it —
crash handling, the window library, the splash image, the thread pool — must work with no
game data at all, which is why the splash image is compiled into the executable rather than
loaded. Everything below it may read files. A rebuild that mounts earlier gains nothing and
loses the ability to report a missing installation with a message box.

**Input is created concurrently and awaited just before the console**, because the console
registers key bindings and the input layer must exist to hold them. The audio device list is
enumerated concurrently and awaited just before the sound device is opened, because a
setting names the device by string and the list is how that name resolves. Both are
latency hiding over the filesystem mount, which is the slowest step: they cost nothing and
save the two slowest device enumerations.

**The renderer is selected before the device is initialized** and the device is
*initialized* well before it is *created*: initialization brings up the window and picks a
mode, creation builds the swap chain. The gap between them is where the console's settings
file is executed, so that the resolution and renderer the player chose are in force before
anything is allocated against them.

**The user's settings file is executed after the console exists but before the device is
created**, and it is preceded by the default-bindings command. That order is the reason a
partially-written settings file still leaves a playable key map: defaults first, the user's
overrides second.

**A `start` or `load` on the command line is executed as a console line**, not handled
specially. The console is the engine's only command surface, and the launcher, the command
line and the script layer all reach the engine through it.

**The player and computer names are taken from the operating system** and replaced with
neutral ones when the configuration says the localization's codepage cannot represent them.
That check is spelled "no native input" in the data and exists for the Asian localizations,
whose fonts have no glyphs for a Latin-1 user name.

**The accessibility shortcuts are disabled** because the operating system's sticky-key and
filter-key prompts trigger on exactly the key repetition a shooter produces, and they steal
focus from a fullscreen window. They are restored at shutdown by a scoped object — which is
incidental; the decision is *disable on start, restore on exit, unconditionally*.

The presence integration (a rich-presence client for a chat service) is genuinely optional,
Windows-only, and fails silently. A rebuild may omit it entirely; nothing depends on it.

## `InitSettings` — the configuration set

**Contract** — opens four configuration files and decides which of the three games is being
run. Three of the four are fatal if missing; the engine's own is not.

```text
FUNCTION init_settings()
  settings          = open "system.ltx"     FATAL if absent
  settings_auth     = open "system.ltx" again, through an include filter   # see note
  settings_openxray = open "openxray.ltx"   not fatal
  game_settings     = open "game.ltx"       FATAL if absent

  mode = the first of:
      a "-shoc"/"-soc", "-cs", "-cop" or "-unlock_game_mode" command-line switch
      the "compatibility/game_mode" key of openxray.ltx
      call-of-pripyat
  set exactly one of the three game-mode flags from it
```

**Notes** — **the main configuration file is parsed twice**, once plainly and once through a
filter that admits only the include paths an integrity check declares un-ignorable. The
second parse is the input to the multiplayer authenticity check: a server needs a digest of
the configuration that *matters*, computed over a set that excludes the files a player is
allowed to modify. Two parses is the crude way to get it. A rebuild computing the digest
during the single parse deletes the second.

**Exactly one of three game-mode flags is set, and the whole engine branches on them.** The
three games differ in small, load-bearing ways — which loading-screen layout, which
inventory rules, which save-game fields — and rather than a data-driven compatibility
table, the engine carries a boolean per game and tests it at each divergence. The source
calls the setter trio ugly and it is. A rebuild should make it one enumerated value; a
rebuild that wants to be *good* should push each divergence into the configuration.

The fourth mode, "unlock", sets none of the three: it is the modder's mode, in which no
game-specific behaviour is assumed.

## `InitConsole`

**Contract** — creates the console — the text-only variant on a dedicated server, the
overlay variant otherwise — registers every command, and names the settings file. A `-ltx`
switch overrides the settings file name, which is how two configurations are kept side by
side.

## `execUserScript`

**Contract** — runs the default key bindings, then executes the user's settings file as a
sequence of console lines. Two calls, and the order is the whole point: defaults first so
every action is bound, overrides second.

## `destroySettings` / `destroyConsole`

**Contract** — `destroyConsole` **saves the settings file before releasing the console**, so
that anything changed during the session is written back. That single line is why quitting
is the supported way to persist settings.

## `Run` — the outer loop

**Contract** — hides the splash, calibrates the clocks, makes the window visible, then loops
until the platform reports a quit request. Each iteration drains the window events, resolves
the focus transition, runs one frame, and services the presence client. Returns zero.

```text
FUNCTION run(application)
  hide the splash image
  device.run()                          # clock calibration, then show the window

  WHILE the platform has not requested a quit      # pumping events is part of this test
    can_activate = false ; should_activate = false

    take up to 32 window events from the queue
    FOR EACH event
      IF it is a window event
        window = the window it names
        IF it means "shown, focused, restored or maximized"
          IF window is not ours -> tell the device that window activated
          ELSE                  -> can_activate = true ; should_activate = true
          CONTINUE
        IF it means "hidden, unfocused or minimized"
          IF window is not ours -> tell the device that window deactivated
          ELSE                  -> can_activate = true ; should_activate = false
          CONTINUE
      device.process_event(event)       # only events not handled above

    IF can_activate
      device.on_window_activate(our window, should_activate)

    device.process_frame()
    service the presence client

  device.shutdown()
  RETURN 0
```

**Notes** — three decisions.

**Focus transitions for the main window are collapsed to one per iteration.** Several focus
events can arrive in one batch — a minimize followed by a restore, or the duplicate pair a
mode change produces — and acting on each would run the activation sequences repeatedly, in
the worst case pausing and resuming the worker pool several times in one frame. Taking only
the *last* state seen collapses that. The source calls it a workaround for screen blinking,
which is the visible symptom; the decision is idempotence.

**Transitions for windows that are not the main one are forwarded immediately**, because
those are the debug overlay's torn-off windows and each is independent.

**At most 32 window events are taken per iteration**, and only window events — everything
else stays queued for the input layer to drain during the frame. The cap bounds the work a
storm of resize events can do before a frame is drawn; a rebuild may raise it, but must not
drain unboundedly, or a window being dragged starves the loop.

The quit test is not passive: asking whether a quit was requested is what pumps the
platform's queue in the first place. A rebuild must keep an explicit pump if its event
source has no such side effect.

## The splash image

**Contract** — a borderless, optionally always-on-top window showing an image compiled into
the executable, raised before the filesystem is mounted and hidden when the loop starts.
Its own thread blits the image once and then ticks at about 30 milliseconds, servicing only
the presence client.

**Notes** — the splash exists because bring-up takes seconds with no window, and a process
that shows nothing looks hung. Its thread does almost nothing: the image is static, and the
loop is there to keep the window responsive to the compositor and to keep the presence
client's callbacks running. A rebuild that can bring up its real window quickly should
delete this.

It is torn down at the start of `Run`, not at the end of construction, so it covers the
last of the bring-up.

## Shutdown

**Contract** — reverses construction. The order is as load-bearing as the bring-up's.

```text
FUNCTION destruct(application)
  persistent.on_app_end()                 # unload the weather, end the game type
  game.destroy_persistent(persistent)
  dump any events still queued            # a diagnostic: what never ran
  release the input layer
  release the configuration files
  release the light-animation bank
  save the settings file, then release the console
  release the video-mode list and the overlay toolkit
  release the sound device
  device.destroy()
  engine.destroy()                        # unloads the renderer module
  release the presence client

  IF a relaunch was requested
    spawn it, with its own working directory

  core.destroy()                          # unmounts the virtual filesystem
  shut the windowing layer down
  finalize the debug facilities
```

**Notes** — **the settings file is written while the console still exists and the filesystem
is still mounted**, which is why the console teardown sits in the middle rather than early.
Everything that writes must precede `core.destroy`.

**The relaunch is spawned before the filesystem is unmounted but after every device is
released.** That is the narrow window in which the successor process can take the graphics
and audio devices while this one still knows where it lives. It is how "change renderer"
works: the new value is written to the settings, a relaunch is armed, and the process
replaces itself.

Queued events are dumped rather than run: anything still deferred at shutdown is a
diagnostic, and running it would touch subsystems already torn down.
