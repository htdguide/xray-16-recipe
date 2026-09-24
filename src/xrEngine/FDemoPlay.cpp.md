# src/xrEngine/FDemoPlay.cpp

> Drives the camera along a recorded path and measures the frame rate while it does — the engine's benchmark.

**Needs** — [`FDemoPlay.h`](FDemoPlay.h.md) · [`Effector.h`](Effector.h.md) · [`CameraManager.h`](CameraManager.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`Render.h`](Render.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md) · [`device.h`](device.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: spline evaluation and timing. The raw path file is read as a memory image, which is the only T1 touch.

## Purpose

A demo is a camera path recorded through the level by [`FDemoRecord.cpp`](FDemoRecord.cpp.md), and playing one back serves two purposes that share all their machinery: showing the level (the attract-mode loop in the main menu) and measuring performance (the benchmark that ships with the game and whose numbers appear in reviews).

It is a camera effector — the highest-priority one — rather than a camera, because it must override whatever camera the game has active without the game knowing.

## State

```text
RECORD DemoPlayback  (extends CamEffector, identity = demo)
  motion          : optional<Motion>     # path as an authored animation curve, if one exists
  motion_params   : optional<MotionCursor>
  keyframes       : list<matrix4>        # path as recorded camera transforms, otherwise
  frame_count     : int
  elapsed         : real (seconds)
  frame_period    : real (seconds per keyframe)
  cycles_left     : int

  measuring       : bool
  frame_timer     : timer                # restarted each frame
  total_timer     : timer
  start_frame     : int
  frame_times     : list<real>           # one entry per rendered frame
```

Two path representations, and which one is used depends on what is on disk. An authored animation curve is preferred; a raw keyframe file is the fallback. They are not interchangeable — a curve carries its own interpolation and its own loop point, while a keyframe list is interpolated here.

## `open`

**Contract** — Given a path name, a per-keyframe period and a cycle count, loads the path and arms playback. Suppresses the first-person weapon, and in benchmark mode the whole heads-up display, so measured frames contain only the scene. Warms the device with fifty frames before measuring begins. Removes itself from the camera stack if neither representation can be loaded, or if the keyframe file's length is not a whole number of transforms.

```text
FUNCTION open(name, frame_period, cycles)
  console.run("hud_weapon 0")
  IF benchmark_mode THEN console.run("hud_draw 0")
  cycles_left = cycles OR 1                       # zero means once, not forever

  try the name with an animation extension, in the level's directory then the shared one
  IF found THEN load as an animation curve and start it
  ELSE
    IF the raw file does not exist THEN remove self ; RETURN
    IF its byte length is not a multiple of one transform THEN remove self ; RETURN
    read the whole file as an array of transforms
  device.precache(50 frames, no user input)
```

**Notes** — The keyframe file is a bare array of 4×4 transforms with no header, which is why its length modulo the transform size is the only validation available. It is written by the recorder in this repository and read by nothing else, so the format is frozen only against itself.

**Notes** — Fifty warm-up frames before measurement, versus twenty after a device reset. The larger number is because a benchmark must not charge the first measured frame for streaming in textures and compiling shaders, and fifty frames is roughly a second of settling.

## `apply`

**Contract** — Called once per frame as a camera effector. Skips entirely while the device is still precaching. Records the previous frame's duration, advances the path, and writes the camera basis into the frame's camera description. Reports expiry only when the cycle count runs out on a keyframe path; an animation-curve path runs until its lifetime expires.

```text
FUNCTION apply(cam_info) -> bool
  IF device is still precaching THEN RETURN alive     # do not measure warm-up frames
  IF not measuring THEN start_measuring()
  frame_times.append(frame_timer.elapsed) ; frame_timer.restart()

  IF path is an animation curve THEN
    evaluate the curve at the cursor into (position, euler angles)
    advance the cursor by the frame delta
    IF the curve wrapped THEN stop_measuring() ; start_measuring()
    derive forward and up from the euler angles
  ELSE
    elapsed = elapsed + frame_delta
    position_in_path = elapsed / frame_period
    frame_index = floor(position_in_path) ; t = fractional part
    IF frame_index >= frame_count THEN
      cycles_left = cycles_left - 1
      IF cycles_left == 0 THEN RETURN expired
      elapsed = 0
    take four consecutive keyframes, wrapping at the end
    interpolate each of the transform's four rows with a Catmull-Rom spline at t
    write the result as the view matrix, invert it, and read the basis out of the inverse
  RETURN alive
```

**Notes** — Interpolation is a Catmull-Rom spline through four consecutive keyframes, applied *row-wise to the raw transform*. That is not a correct interpolation of a rigid transform — the rotation rows are splined as free vectors and the result is not orthonormal — but it is smooth, and the camera manager re-orthonormalizes the basis afterwards, so the visible artifact is only a slight shear in the intermediate matrix. A rebuild should spline the position and slerp the rotation; the resulting path will differ slightly from the original's.

**Notes** — Catmull-Rom is the choice because it passes *through* every keyframe rather than near it, which matters: the recorder captures keyframes where the author stopped and looked, and a B-spline would cut those corners.

**Notes** — The result is written into the device's view matrix and then *read back out* by inverting it, rather than being computed as a basis directly. That is a leftover from when demos drove the device directly, and it costs an inversion per frame. The decision that survives is only "the recorded transform is a view matrix, so its inverse is the camera's basis".

**Notes** — Restarting the measurement when an animation-curve path wraps means each loop is measured and reported separately. The keyframe path deliberately does *not* do this — the code to do so is present and commented out — so a multi-cycle keyframe run produces one aggregate number.

## `start_measuring`

**Contract** — Records the current frame number, starts both timers, and clears the sample table with room reserved for about a thousand frames. Sleeps one millisecond first, to land the first measured frame on a fresh scheduler quantum rather than on the tail of the caller's.

## `stop_measuring`

**Contract** — Computes and reports four frame-rate figures, and in benchmark mode writes them plus every per-frame time to a result file and quits the process.

```text
FUNCTION stop_measuring()
  average = frames_rendered / total_elapsed_seconds

  # A sliding window smooths out single-frame spikes, so that "minimum frame
  # rate" means a sustained dip rather than one long frame.
  window = max(16, max(average, 10) / 2)          # about half a second of frames
  IF sample_count > window * 4 THEN
    FOR EACH window position, starting at sample 2
      rate = window / (sum of that window's frame times)
      track min, max, and the running mean of these windowed rates
  ELSE
    # too few samples to window: use raw per-frame rates
    FOR EACH sample after the first
      rate = 1 / sample ; track min, max, mean
  report average, min, max, mean
```

**Notes** — Four numbers, and they answer different questions. *Average* is total frames over total time and is what a marketing figure quotes. *Minimum* and *maximum* are over the smoothed window, so they describe sustained behaviour. The *mean of windowed rates* is deliberately not the average: it weights each window equally rather than each second, so a long stall drags it down harder. Reporting all four is the honest thing to do and the original does it.

**Notes** — The window is half the average frame rate, floored at sixteen frames — that is, about half a second of play. Below four windows' worth of samples the smoothing is skipped, because a window comparable to the whole run measures nothing. The first one or two samples are dropped in both branches: they include the transition into measurement.

**Notes** — The benchmark result file is named from a command-line-supplied benchmark name, or `benchmark` by default, and records the renderer generation alongside the numbers because a frame rate without a renderer generation is meaningless. Every per-frame rate is written as a separate key with a zero-padded index, so the file can be plotted. Writing it is the last thing the process does — it quits immediately afterwards.
