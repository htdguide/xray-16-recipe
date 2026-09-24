# src/utils/xrCompress/main.cpp

> The packer's command line — it chooses between packing and differencing, turns flags into a job description, and mounts exactly one folder as the whole filesystem.

**Needs** — [`xrCompress.h`](xrCompress.h.md) · [`StdAfx.h`](StdAfx.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/xrDebug.h`](../../xrCore/xrDebug.h.md) · [`xrCompressDifference.cpp`](xrCompressDifference.cpp.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: argument handling and a filesystem mount. Nothing here touches a byte layout.

## Purpose

The composition root of the packer, and the answer to a question the packer object
deliberately does not ask: *what is being packed, and how hard*. It does three things that
matter and one that is a historical wart.

The thing that matters most is the **filesystem mount**. The engine mounts a chain of
archives and directories described by a configuration file; the packer mounts exactly one
physical folder as the entire namespace, with no archives, no roots file and no recursion
into parents. That is what makes the archive's entry names come out relative to the folder
the operator named, which is the whole contract between the packer and the mount path.

The wart is that options are recovered by **substring search over the raw command line**
rather than by parsing arguments in order. A rebuild should parse properly; the
observable consequence of not doing so is that an option's value must be separated from
its name by exactly one space, and that a path containing the text of another option's
name is misread. Preserve the option *names* — build scripts use them — not the scanning.

## State

```text
RECORD PackerJob                  # what the command line resolves to
  mode          : ENUM { pack, difference }
  source_folder : text            # the one folder mounted as the whole namespace
  store_only    : bool
  fast          : bool
  patch_archive : bool            # emit the patch container variant
  output_name   : optional<text>  # explicit archive name, extension included
  volume_limit  : optional<int>   # megabytes on the command line, bytes internally
  job_file      : optional<text>  # a folder list with recursion flags
  header_file   : optional<text>  # contents copied into the mount-configuration chunk
```

**Invariants**

- Exactly one of `job_file` and `header_file` is used. When a job file is given the header
  file is never consulted; when it is not, a header file is **required** and its absence
  aborts the run rather than producing a headerless archive. A rebuild that makes the
  header optional produces archives the patch path will not mount.
- `volume_limit` is stated in megabytes by the operator and held in bytes everywhere else.
  The conversion is the only place the unit changes, and getting it wrong produces archives
  a thousand times too small or silently over the format's limit.
- `source_folder` is lowercased and given a trailing separator before the mount. Entry
  names in the archive inherit that normalization, and the mount path compares names
  case-sensitively, so skipping it produces an archive whose files cannot be found.

## `main`

**Contract** — reads the process command line, initializes the crash handler, the core
layer and the log, then dispatches to either the differencing pass or a pack run. Returns a
distinguished non-zero status when the operator asked for help or supplied no folder, and
zero otherwise — including when a pack aborted for a missing header, which is a defect
worth fixing in a rebuild. Blocks for the duration of the run. Writes to the output folder
and to the log.

```text
FUNCTION main(arguments) -> int
  raw <- the whole command line as one string     # options are found by substring
  install_crash_handler(raw)
  initialize_core(app_name = "xrCompress", log_to_file = false)

  IF raw CONTAINS "-diff"
    RETURN process_difference()                   # xrCompressDifference

  IF arguments.count < 2
    print_usage()                                 # also prints the format's volume ceiling
    RETURN 3

  folder <- lowercase(arguments[1] + path_separator)
  # Mount this one folder as the entire namespace: no archives, no parent roots.
  filesystem.initialize(target_folder_only, folder)
  filesystem.add_root("$working_folder$", "", recurse = false)

  packer <- new Packer
  packer.store_only    <- raw CONTAINS "-store"
  packer.fast          <- raw CONTAINS "-fast"
  packer.patch_archive <- raw CONTAINS "-xdb"
  packer.target_name   <- arguments[1]

  IF raw CONTAINS "-filename"  THEN packer.output_name <- token_after(raw, "-filename")
  IF raw CONTAINS "-max_size"  THEN packer.volume_limit <- megabytes(number_after(raw, "-max_size"))

  IF raw CONTAINS "-ltx"
    job <- read_configuration(token_after(raw, "-ltx"))
    packer.run_from_job(job)                      # ProcessLTX
  ELSE
    header <- token_after(raw, "-header")
    IF header IS none THEN log("cannot process -header, aborting"); RETURN 0
    packer.header_file <- header
    packer.run_over_target_folder()               # ProcessTargetFolder

  shut_down_core()
  RETURN 0
```

**Notes**

- The usage text is part of the contract, because build scripts and modding tooling were
  written against these exact option names. The job-file shape it documents — a section
  whose keys are paths and whose values are a recursion flag — is the same shape
  [`xrCompress.cpp`](xrCompress.cpp.md) consumes.
- `-store` and `-fast` are not mutually exclusive on the command line and `-store` wins,
  because the packer checks it first. That is a consequence of the flag order, not a
  decision; a rebuild should make it explicit.
- The differencing mode shares nothing with the packer but the executable. It is in the
  same binary because the same person needed both when building a patch, not because the
  two have anything to do with each other. A rebuild may ship two programs.
