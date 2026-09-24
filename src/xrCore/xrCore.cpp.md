# src/xrCore/xrCore.cpp

> Process bring-up and teardown: identity (who and where we are), the build stamp, the order in which the core's subsystems come alive, and the reference count that lets several modules ask for the core independently.

**Needs** — [`xrCore.h`](xrCore.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`log.h`](log.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`FileSystem.h`](FileSystem.h.md) · [`Threading/TaskManager.hpp`](Threading/TaskManager.hpp.md) · [`Compression/rt_compressor.h`](Compression/rt_compressor.h.md) · [`Compression/compression_ppmd_stream.h`](Compression/compression_ppmd_stream.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`string_concatenations.h`](string_concatenations.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrCore.h`](xrCore.h.md)
**Tier floor** — T1: it queries the platform for the module's own location, raises the process timer resolution, and orders construction of globals that other globals' constructors already depend on.

## Purpose

Everything in the engine rests on the core, and the core has an order: the allocator and its interners, then the thread pool, then the compression tables, then the virtual filesystem. This file owns that order. It also establishes the four identities the rest of the engine reads constantly — the application's name, the directory the executable lives in, the working directory, and the user and machine names that log files and save paths are keyed by.

## State

```text
RECORD Core                        # one process-global instance
  application_name  : text (64)    # set by the caller at bring-up; names logs and window titles
  application_title : text (64)
  application_path  : text (path)  # directory the core module itself was loaded from
  working_path      : text (path)  # process working directory at bring-up
  user_name         : text (64)    # sanitized, see below
  computer_name     : text (64)    # sanitized
  params            : text         # the whole command line, owned copy
  frame             : int (32-bit) # the frame counter, advanced by the loop, read everywhere
  plugin_mode       : bool         # we are inside somebody else's process
  build_id          : int (32-bit) # days since a fixed epoch, see below

  # Module-level, not a field:
  init_counter      : int          # how many modules have asked for the core
```

**Invariants**

- **Bring-up and teardown are reference-counted.** Several modules independently ask for the core; only the first does the work and only the last undoes it. The filesystem part, however, runs on *every* request, because a later caller may be the first to want a mounted filesystem.
- **The user and machine names are sanitized before use**: backslash, forward slash, comma and period are each replaced by an underscore. They become parts of filenames (the log is named after the application and the user), and a name with a separator in it would escape its directory. This is the one piece of untrusted input the core embeds in a path.
- **The frame counter is read by every subsystem** as a cheap "has time moved" test, and is the closest thing the engine has to a global clock tick.

## `initialize`

**Contract** — Takes the application's name, the command line, whether to mount the filesystem, an optional name for the filesystem description file, and whether the process is a plugin host. Blocks until everything is up. Idempotent in the sense that repeated calls only increment the counter, except for the filesystem step.

```text
FUNCTION initialize(app_name, command_line, mount_fs, fs_description, plugin) -> void
  application_name = app_name
  print_build_info()                      # first thing in every log, see below

  IF init_counter == 0 THEN
    REQUIRE the processor supports the 4-wide float baseline   # on x86 targets
    plugin_mode = plugin
    params = owned copy of (command_line, or empty)

    raise the process timer resolution to 1 ms
    initialize the platform's component system          # the audio device needs it

    application_path = directory of the module this code lives in
    working_path     = process working directory
    user_name        = platform user name,  sanitized
    computer_name    = platform host name,  sanitized

    memory.initialize()                   # interners come alive here
    route the windowing library's log into ours
    log the command line
    detect processor features
    start the task scheduler's worker threads
    initialize the real-time compression tables
    create the virtual filesystem object
    create the path-utility object

  IF mount_fs THEN
    flags = flags_from(params)            # see below
    filesystem.initialize(flags, fs_description)
    path_utilities.initialize()
  init_counter = init_counter + 1
```

**Invariants** — The order is not negotiable in three places: the allocator's interners must exist before anything interns a name; the task scheduler must exist before anything schedules; and the filesystem object must exist before its mount is attempted. Everything else may move.

**Notes** — On the POSIX family the application path falls back, when the platform cannot report the executable's directory, to the *preference directory* of whichever of the three games the command line names — Shadow of Chernobyl, Clear Sky, or Call of Pripyat, the last being the default. That is how a packaged build with no writable install directory finds its data root.

## Filesystem mount flags

**Contract** — Four decisions are read off the command line and handed to the filesystem:

| Asked for | Effect |
|---|---|
| build mode | keep a copy of every file read, for repacking |
| editor build mode | the above, plus the editor's extra copying |
| cache files | cache file contents in memory — debug builds only |
| file-activity dump | log every file the process opens |

Additionally, on non-Windows platforms, the application root is scanned **only when the application path is not the system-wide data directory** — a packaged install reads from its fixed location and must not scan beside the executable, while a local build must.

**Notes** — The editor builds force the cache off. That is a correctness requirement, not a performance one: the editor rewrites files the engine has already read.

## `destroy`

**Contract** — Decrements the reference count; on reaching zero, tears down in reverse: the filesystem and its path utilities, the trained compression model (whose buffer is freed separately from the model itself), the task scheduler, the owned command line, the allocator's interners, and finally the platform's component system and timer resolution.

**Invariants** — The compression model's buffer is a raw block handed to the model at load time; the model does not own it and both must be released here. This is the only place that knows the pairing.

## `calculate_build_id`

**Contract** — Turns the compilation date into a single increasing integer: days elapsed since 31 January 1999.

```text
FUNCTION calculate_build_id(build_date) -> int
  parse build_date as "Mon DD YYYY"        # the compiler's own date format
  months = index of Mon in the three-letter month table
  id = (YYYY - 1999) * 365 + DD - 31
  add the day counts of every whole month before `months`
  subtract the day counts of every whole month before January
  RETURN id
```

**Notes** — The epoch is the start of the original project. The arithmetic ignores leap years entirely, and the final subtraction loop is empty because the start month is January — both are artefacts of a general formula that was only ever used with one epoch. The number is therefore **not** a day count in any exact sense; it is a monotonically increasing build stamp that appears in logs and in the multiplayer handshake, and any rebuild that changes it changes what those compare. Reproducing the formula exactly is only necessary if compatibility with the original's version reporting matters.

## `print_build_info`

**Contract** — The first lines of every log: the application name, the build configuration, the build identifier, the compilation date, then a line naming the continuous-integration system (or "Custom"), its build number and unique identifier where available, the source commit, the branch, and who built it. Assembled by repeated concatenation into one buffer and logged as a single line.

**Notes** — The commit and branch come from a generated header if one exists and are otherwise the literal text "unknown". This is the only identity the engine has for reproducing a bug report, and it belongs in a rebuild unchanged.

## Windowing-library log bridge

**Contract** — Every message the windowing library emits is reformatted into the engine's log with a one-character severity mark, the originating category, and the severity name: `%` verbose, `#` debug, `=` info, `~` warn, `!` error, `$` critical.

**Notes** — The mark characters are a convention the whole engine's log follows, and log-reading tooling keys on them. `!` in particular is what a reader scans for. The buffer this message is assembled into is sized from the *sizes of the pointers* rather than the lengths of the strings, which is a latent truncation bug in the original; a rebuild should size it from the actual lengths.
