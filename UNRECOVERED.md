# What could not be recovered

A recipe that only described the design would be a worse artifact than one that also says
where the design and the code disagree, and where the source simply does not answer.

This file collects, chapter by chapter, what the writers of this recipe could **not**
recover from the source: constants with no derivation, fields written and never read,
abandoned designs, decisions whose intent is nowhere stated — and defects, where the
shipped behaviour is demonstrably not what the code appears to intend.

Three conventions, used throughout:

- **"Could not recover"** means the fact is stated and the *reason* is not. The value is in
  knowing you are free to choose, rather than reverse-engineering a constant somebody
  picked by feel in 2007.
- **"Bugs found while recipifying"** are recorded in the twin where they live, marked as
  defects rather than transcribed as contracts. A rebuilder implements the intent — or
  reproduces the bug deliberately for compatibility — instead of inheriting it blindly.
- **Dead behaviour** is stated as dead. Several complete features in this engine are
  registered and never selected, commented out of their own build, or superseded by test
  scaffolding left in the shipped product. Reimplementing them as though they ran is wasted
  work.

Each chapter's `README.md` carries the items that matter most for reading that chapter;
this is the complete list. Entries are phrased so they can be read without the source
open.

---

## src/Layers/xrAPI
- Renderer fallback order is effectively arbitrary: the mode map is keyed by raw string pointers, so it iterates by address. Whether a particular backend was meant to be preferred is not discoverable.
- The environment header forward-declares four types no slot references (material library, a concrete render class, a token type, the draw-utility pairing). Presumably removed slots; nothing confirms it.
- Why the script engine's destruction was wrapped in a catch-all: a comment says the original did so, without saying what it caught.
- A dedicated server sets a renderer mode name that no module registers, so it is always overridden by the fallback. Vestigial, or compatibility with an old settings file — not recoverable.
- The renderer's five environment slots are never cleared: both backends implement the clear correctly and nothing calls it. The invariant survives only because nothing reads the renderer after the device dies.

## src/xrCommon
- A fixed-size array type is padded by one dead 32-bit field to preserve the size of an older count-carrying type it replaced. Nothing uses it, so the record whose stride it was protecting is gone or was never named.
- The yaw/pitch/roll composition order of the orientation record is stated nowhere; it is inferred from matrix conversions in a later chapter.
- Whether the engine-wide allocation routing is intentionally triple-redundant (container aliases + replaced global construction operators + raw-byte helpers) or is layered history. The contract is recoverable; the intent is not.
- The fast path of angle normalization admits exactly one full turn while the unconditional version maps it to zero. Looks like an oversight; no caller or comment settles it, so it is recorded as contract.

## src/utils/xrMiscMath
- The quaternion-extraction tolerance 0.1 has no derivation; it is a generous floor, not an epsilon.
- The factor of sixteen in the gimbal-lock threshold (16 x float-epsilon) is undiscoverable; only its order of magnitude is defensible.
- The exact-normalize guard 1.192092896e-05 is recognizably 100x single-precision epsilon, written as a single-precision literal then used as a double. Whether the x100 or the narrow literal was intended is unknowable.
- Straight up (0,1,0) as the zero-vector fallback direction: nothing derives it.
- Why exactly five vector functions are out of line while the rest of the same header is inline. They are neither longer nor heavier.
- Whether the x86-only flush-to-zero / denormals-are-zero mode is known to the maintainers: no comment, no guard, no compensating code, and the instruction does not exist on ARM or PowerPC. Same source, different rounding by target.
- Whether the duplication between the combined and single-value Euler extractors is an inlining decision. Nothing says so.

### Bugs found while recipifying (recorded as contract, flagged in the twins)
- Transform composition applies its SECOND argument first; argument order is reversed from the underlying product, nothing asserts it, and a rebuild that reads it the other way is silently and consistently wrong.
- The quaternion extraction's diagonal ranking compares the third diagonal against the first in both branches, never the second. The fixed fallback chain is what makes it harmless.

## src/xrMaterialSystem
- The ~50-entry acoustics table has no derivation and no consumer: absorption, scattering and transmission are loaded but nothing reads them back. Its chunk id is a project addition; no shipped library carries it.
- The multiplayer shoot factor is loaded and never read anywhere in the repository; the penetration path that would use it does not appear.
- Four authored flags (breakable, skidmark, shootable, transparent) and the sound-occlusion factor are loaded and never read.
- Sound lists are capped two entries higher than particle and decal lists. No comment explains the difference.
- Pair inheritance fields are read and never consulted; the authoring tool resolves inheritance before writing, but that tool is not in this repository.
- The write side of the frozen material format is declared and implemented nowhere.
- Dead bit and chunk positions are burned without recorded reason, and why the four derived flags sit at the top of the word rather than continuing the sequence is undiscoverable.
- Two flags are documented as derived from numeric fields but nothing recomputes them at load: a rebuild that writes libraries must honour an invariant nobody enforces.

### Bugs found while recipifying
- The library checksum hashes from the current read position for the whole file length, over-reading the buffer by ten bytes. Stable per file, so nothing fails; the recipe tells the rebuilder to hash the whole file.
- The "no material" sentinel is declared 32-bit but initialized from a 16-bit all-ones value. Comparisons against 16-bit indices work; the type is inconsistent with the name.

## src/xrCDB
- The per-face 32-bit flag word on the packed collector is written and read only by the level compiler, which is not in this repository. Its bit meanings are unrecoverable here.
- The welding grid's 24x16x24 resolution and the minimum octree node size are tuning values with no derivation anywhere in the source.
- The index update entry point takes a node count that nothing reads: the remnant of an incremental maintenance pass that no longer exists.
- Two complete alternative ray-query strategies sit unreferenced. Whether they were abandoned for correctness or for cost is not recorded.

### Worth fixing rather than reproducing
- Neither the static tree's nor the octree's ray descent orders children by ray entry distance, which is the obvious available win for nearest-only queries.
- The kind mask is read as AND in the ray query but OR in the box and frustum queries, undocumented, and callers of both depend on it.

## src/Common
- The gap between the level format version the engine reads and the one the shipped tools emit: four intervening versions exist and nothing says what they changed.
- Why a 64-bit PowerPC target takes the signed pointer-sized type for message parameters where every other 64-bit target takes the unsigned one.
- The face smoothing edge bits are inverted (set means hard). Almost certainly a vestige of a smoothing-group mask; nothing confirms it, and the encoding is frozen by authored data either way.
- The instanced-model vertex writes four 16-bit values and only two are ever read. The other two are produced by the level compiler and consumed by nothing; the bytes must still be written for the stride.
- The mesh mender's degeneracy threshold in the texture-gradient solve is absolute rather than scale-relative, with no derivation; likewise its near-parallel repair and length cutoffs.
- The default light range is the square root of the largest representable float. The headroom reason is inferable from squared-distance comparison but not stated.
- Nothing in the tree shows what generates a level build identity; only comparison and persistence are here.
- The level chunk enumeration skips one value. Whatever that chunk was, it is gone and unreferenced.

### Bugs found while recipifying
- The serialization encoder's element filter writes a count taken before filtering, producing a stream the decoder reads past the end of.
- The cloner and the loader use different tests for "ordered container" versus "sequence".
- In the vendored mesh mender, a triangle that can smooth with both neighbours is recorded in two groups but names one, making the result mildly order-dependent.

## src/xrParticles
- The fractal-sum octave lacunarity (2.059) and the normal sampler's fitted mean (0.7975) have no discoverable derivation.
- The noise table shuffle walks backwards in steps of two and starts one past the filled range, reading a still-zero slot. It produces a valid but lopsided permutation; whether this was intended is unrecoverable, and it IS the field, so it must be copied literally.
- Noise initialization reseeds the platform integer generator as a side effect - the same generator the normal sampler draws its sign bit from. Whether the coupling was noticed is unrecoverable.
- The orbit actions use a square root where every sibling action multiplies. Almost certainly a typo, frozen because authored magnitudes were tuned against it.
- The velocity-matching action does not match velocities: it nudges one particle toward the other's velocity and the other away from its own. The name records intent, the arithmetic records behaviour.
- The avoid action's rectangle path computes its third and fourth candidate edges identically, so one edge is never chosen as an escape direction.
- The look-ahead parameter has three inconsistent readings across the domain kinds that support it (time, distance, speed x time).
- The action writer does not round-trip with the reader and has no caller: the authoring tool is not in this repository, so the format is documented only by its reader.

### Bugs found while recipifying
- Unsigned loop-bound underflow in the follow action on an empty pool.
- The source action fills its second vertex from an unassigned value when vertex-B tracking is off.

## src/xrScriptEngine
- The interpreter-patch dependency opened into the VM before any game script runs is an unchecked-out submodule. What it changes about the language is unknown, and it is potentially part of what "runs unmodified" means.
- The export sort iterates an unordered map, so two independent nodes' relative order varies between runs. Whether anything ever depended on an order is undeterminable.
- The per-frame collector step is guarded to developer builds; read as an oversight, not recoverable.
- The coroutine registry reference is never released in the shipping configuration, behind a switch citing unspecified binding-layer defects.
- The debugger's run-to-cursor compares the file for inequality and its enabling message is a no-op; unclear whether it ever worked. The global-variable pane iterates and sends nothing.
- Round numbers with no derivation: three discarded random draws, a 1 MB source buffer, a 2048-byte message buffer, a 256-frame shadow stack, a 128-entry report limit.

## src/xrSound
- Why the reverb blend uses the raw frame delta as its interpolation factor, which makes it frame-rate dependent. No comment, no tuning constant.
- Why the AI announcement range is scaled by instance volume but not by occlusion, when the occluded value sits two lines away. Deliberate "AI hears through walls" or an oversight - unknowable.
- The occlusion scale and cull volume magnitudes are plausible but unjustified anywhere; both are console variables, so tuned by ear.
- The sidecar's level-version field is declared and never read.
- Per-emitter reverb is computed every frame and never applied: the shipped backend's reverb is listener-global. Vestigial or preparatory, not determinable.
- Whether the "generic hardware to generic software" device substitution is still needed. The source's own comment asserts the problem persisted for twenty-two years with no measurement behind either claim.

### Bugs found while recipifying
- The source cache's double-checked insert overwrites on a race, leaking the loser's description.
- The music streamer's slot-reuse scan can never find a free slot, because deletion never clears the entry. Dead code; noted so nobody preserves it.

## src/xrNetServer
- Two magic constants read as dates, almost certainly birthdays. No structure.
- The 36-byte compression threshold has no derivation anywhere.
- The 250-port search window is inherited from the dead matchmaking service's scan range.
- The 256-sample convergence and 512-sample ring for clock sync are stated, never justified.
- Why the 8-bit quantizer divides by a value fractionally above 255 while the 16-bit one divides by an exact 65535. The effect is recoverable, the intent is not.
- The server's pending-send limit is one higher than the client's. The direction is explicable, the magnitudes are not.

### Bugs found while recipifying
- The compressed path cannot work: both compress and decompress checksum the body plus five bytes past it, and those trailing bytes differ between sender and receiver. A genuinely compressed envelope fails verification, which is fatal rather than a drop. It never fires because compression defaults to off.
- The message-name table has drifted out of step with the message enumeration, and names are matched by position, so every name past roughly index 26 is wrong.
- The roster lookup returns a reference after releasing its lock. Survivable only because all call sites are on the simulation thread.
- Address equality is asymmetric: a zero fourth octet on the right-hand side acts as a class-C wildcard, so it is not an equivalence relation.
- The subnet filter's masked-comparator binary search breaks silently on overlapping blocks, nothing validates against them, and prefix length is unbounded.
- The null server's governor reads two never-assigned locals; the depth brake was removed along with the vendor call.
- The null client's classifier lost an early return, so an unrecognized system packet falls through into the engine-message path.
- Declared and never defined: two update-rate accessors and a client-abort callback. A packet timestamp is never written or read. A stream tag is accepted but never produced.

### CORRECTION NEEDED IN SYSTEM-REQUIREMENTS
The build lists the NULL networking filling as the default on every platform; the real client and server are commented out. Multiplayer is opted into by editing the build description. Seam: Networking transport and conformance items 13-14 must say so.

## src/xrAICore
- Patrol paths: the 15 cm lift applied to a waypoint before mesh matching has no derivation. A name collision between two waypoint-kind enumerations in the script namespace means which registration wins must be checked against a running original. An unused named constant holds one path's name.
- Planner components: the visited-state budget is 8000 against a pool of 8192, and a source comment flags that the solver assembly is constructed for 16384 while its manager and allocator are sized for 8192.
- The level-mesh heuristic weight of two and the bucketed queue's 0-2000 range are reproducible and clearly deliberate but undocumented.
- The straight-line path request's range field carries three meanings at once, with no statement that this was a choice.

### Bugs found while recipifying
- The world-state weight accessor is declared with the wrong parameter and compiles only because it is never instantiated.
- The property accessor returns the NEXT property when the requested one is absent, and is reachable from script.
- The patrol-path registry keeps two keys to one owned path and coexists with a free-everything teardown.

## src/xrUICore
- The 250 ms double-click window is the conventional desktop value, not derived from game data.
- The hover-hint dwell interval and the spin box's two repeat delays are tuning with no recoverable derivation.
- One scrollbar visibility predicate carries a source comment reading literally "no comment".

## CORRECTION to an earlier chapter brief
The AI searches are budgeted but explicitly NOT resumable: exceeding any of the three budgets reports failure and leaves no frontier behind, so a caller spreading work across frames re-issues the whole search.

## src/xrCore (math and geometry primitives)
- The quaternion fast arccosine is a fitted minimax odd polynomial with no recorded fitting criterion or error bound. Documented as "refit for your precision", not "copy".
- None of the five quaternion tolerance constants is derived from anything stated; two of the five are declared and never used.
- The cylinder's axis-parallel epsilon is far tighter than every other geometry tolerance in the chapter. Why it must be tight is arguable; the specific value is unexplained.
- The eigen-decomposition's three-sweep thresholding comes from the standard published form, not from anything in this engine.
- One matrix builder performs a handedness flip named after a collision library that no longer appears in the tree, and has zero callers, so the frame it converts to is unrecoverable.
- Two box "pick" routines never write the coordinates of axes the origin is already inside on, so the caller's output point must be pre-initialized. Whether callers rely on that or are latently buggy was not determinable.
- The negated Euler-extraction variants exist because two parts of the engine disagreed on an angle's sense; which caller wanted which is not recoverable.

### Dead code with no recoverable intent
- A 2D box's corner accessors are wrong as written (four corners, one extent never read) and have no callers.
- Three rectangle predicates reference non-existent members and compile only because they are never instantiated.
- Five 3x3-matrix operations, including the eigen-decomposition, have zero callers.
- One plane-segment intersection's guard rejects exactly the crossing case; its handful of callers appear to want a different question than the name asks.

## src/xrUICore (controls)
- The spin box's auto-repeat exponent, delay decrement and accumulator growth are tuned feel constants with no derivation.
- A scroll box's drag margin is annotated in the source with a smiley and nothing else; the page and wheel multiplier is likewise unexplained.
- The integer spin box refuses an overshooting step while the real-valued one clamps. Whether the divergence is intentional is not recoverable.
- One scroll bar's strict "cursor inside the thumb" drag test replaced a commented-out generous-margin test, with no recorded reason.

### Bugs found while recipifying
- The spin box's first auto-repeat burst is a no-op because the accumulator starts at zero, and its enable path sets the enabled and disabled text colours the wrong way round (masked by the per-frame update).
- The horizontal frame line applies its aspect correction to the second cap's length only, observable as a wider right cap on wide displays.
- The animated static divides the frame index by the row count to get the row, where it needs the column count. Harmless only because every shipped sheet is square. Its per-frame duration is module-wide, shared by all instances, rather than a field.
- The properties box sizes its inner list to the whole box, discarding the frame inset applied at construction; its width cushion is unexplained; and hiding it unlocks the focus system without checking that it owns the lock.
- The tab control declares four colours but only ever applies the idle pair, and only at insertion. Removing by index reorders the strip by swap-and-pop where removing by identifier preserves order, and removing the active tab leaves the active identifier dangling with nothing to repair it.
- A frame window reuses one vertex-count accumulator across three contributions, so only the last-computed branch survives if an earlier one did not run. Masked because shipped panels always have spare space in both axes.
- Both scroll bars keep their auto-repeat timestamp in a module-wide variable rather than per instance.
- The track bar's "set default value" treats the default as the midpoint of the range, which is not a stored default and will not match settings whose shipped default sits elsewhere.

## src/xrEngine
- The maximum load-stage counts (18 with the alife simulation, 14 without) are hand-counted; nothing verifies that callers advance the stage that many times, and the absolute numbers have no derivation.
- Prefetch sections still carry per-entry counts from an abandoned pooling design; nothing reads them, and the old cap of 128 has no recoverable reason.
- The sound occlusion scale is the only setting read from configuration and clamped rather than bound to a console name. No recoverable reason for the asymmetry.
- One gamepad deadzone setting exists as storage with its registration line commented out. Intent unknown.
- Read-only mode in the line editor is enforced by binding fewer keys, but the text-input path has no read-only case, so a read-only control still accepts typed characters. Cannot tell whether that is a bug or relied upon.
- The console's word terminators include the underscore, so an underscore-separated command name is several words to word motion. Plausibly unintended; no evidence either way.
- The multi-window driver allow-list is empirical: it names five drivers and excludes one platform from hover reporting. The underlying capability has no query, so the list is observation, not derivation.

### Bugs found while recipifying
- The binding context check reports a conflict when contexts DIFFER, and also when both are "none" - the opposite of what the matching rule makes possible. Reads as an inverted condition.
- One key name ships to script with a stray closing parenthesis in it, and several gamepad names carry an inherited d-pad prefix for buttons that are not on the d-pad. Both are now frozen typos.

## src/Layers/xrRender (material templates, debug primitives, interface fillings)
- Magic constants with no derivation: the shadow-pass alpha reference (which differs from the authored one); the alpha threshold that detours a model from the deferred path to the forward one; the cloud dome's scale and its two wind headings; the sphere-wedge cap angle.
- The unit sphere's 92-vertex, 180-triangle table does not decompose into a clean subdivision rule. It is authored data and must be copied, not regenerated. The same holds for the sphere wedge.
- One rain-blender element names a program that is absent from the multisample template, and nothing explains what it is for. Probably a leftover diagnostic layer.
- A light-occlusion element's source comment names a graphics card from 2004 and is stale; its actual role (writing a per-light stencil marker bit) was reconstructed from the stencil masks in the reset element and is inferred, not stated.
- The multisample mask blender's albedo path uses the non-multisample copy program. Deliberate (it reads a resolved surface) or a copy-paste slip; documented as deliberate with the reasoning shown.
- Who writes the sun-mask texture is outside this module; it is referenced by fixed name from every sun and rain pass.
- That the parenthesised tessellation-feature suffix on a shader name serves as the compiled-blob cache key is inferred from its shape, not documented.

### Bugs found while recipifying
- The debug renderer paints a whole flush with the FIRST vertex's colour; an overflowing shape is dropped rather than appended after the flush; and the frame-loop-registered variant clears its lists on every append, so only the last shape survives.
- One post-process blender header closes its namespace inside a conditional branch, leaving it open on the oldest deferred generation.

## src/xrPhysics
- The camera-collision near-plane box is stretched in depth and height and shifted back and down by half of each, with no derivation. Authored by eye. The anti-character cylinder's two offsets are likewise arbitrary.
- Activation-shape iteration counts, the resolve depth, and the contact effector's drag multiplier are all bare magic numbers.
- The dynamic-box limiter's divisor and its fixed phase-one iteration bound have no source justification.
- The actor's free-fly upward force limit is an absolute force, so it silently depends on the player's mass. The source never acknowledges the coupling.
- A commented-out alternative activation path (a full island-merging contact handler with separate static and dynamic solver parameters) is dead but substantial; whether it was abandoned for correctness or performance is not recoverable.
- The collision validator's group-restore entry point is an empty function with live call sites: the other half of a removed mechanism.
- The actor character's speed-goal field is declared and never used, and one capture state is never entered.
- The physics commander mixes locked and unlocked call sites with no discoverable intent. The file itself carries a comment saying crashes here appeared after memory pools were removed and were never root-caused.
- Two collider translation units are near-empty, and one preserves a filename misspelling.

### Bugs found while recipifying
- Capture's angular-motor perpendicular axis is chosen by a branch tree testing only the positive side of each direction component, so it is not symmetric under negating the direction.

## src/Layers/xrRender (skinning)
- Why the one-influence vertex format spends 32 bits on a bone reference that is always read back as 16 bits with a zero upper half.
- Why the two-influence blend interpolates while the three- and four-influence blends accumulate weighted terms. The results agree; the rounding differs.
- Why only the four-influence routine is threaded in the portable path when all four are equally parallel, and why the 4-wide path threads none.
- Whether bone picking's change from nearest-hit to first-hit was deliberate. The nearest-hit implementation survives commented out with no note.
- Whether the 4-wide skinning path was ever measured against the portable one on a modern compiler.

### Cross-chapter hazard
- The 4-wide skinning path reaches bone matrices by computed byte offset, hard-coding both the bone-instance size and the render-transform offset. Those hold only in the 32-bit build (the callback pointers widen at 64 bits), and it reads the RENDER transform, not the local one. Both numbers derive from a record in chapter 6 that nothing in chapter 18 states; the twin documents the derivation so a rebuilder does not inherit the constants blind.

## src/Layers/xrRender (texture descriptions, wallmarks, blender templates)
- The texture-usage entry that names both diffuse and bump combines the two flags by ORDINAL, so it sets only the bump flag: in the running engine it is identical to naming bump alone. Intent and shipped behaviour cannot be separated.
- One screen-space template writes three disagreeing mode counts (the constructor writes one version and count, the loader forces another). No explanation; treated as presentation-only.
- The model template's high-quality fixed-function element draws the light projector alone and never samples the base texture. The two-stage version that samples both is present and disabled, with a commit link but no reason.
- One alpha-tested template's fallback shape has its alpha-test line commented out, so cut-out foliage draws opaque there. Defect or deliberate is unknowable.
- One multisampled bloom template is byte-for-byte equivalent to its non-multisampled twin in every generation. Unfinished distinction or a needless one.
- The deferred alpha-tested template defaults its alpha reference to 200 while the forward filling of the SAME class tag defaults to 32. Never reconciled; both must be kept.
- The grayscale luminance constant's bias is derivable from the fixed-function signed-bias convention, but the exact value has no recorded derivation.
- One lightmap-sampling variant omits the clamped addressing the canonical lightmap stage uses. Possibly an oversight; harmless because lightmap charts carry a border.
- One reserved element index in the particle template is empty in every generation; nothing requests it and nothing says what it was for.

### Bugs found while recipifying
- The deferred alpha-tested loader gates on version EXACTLY one rather than at least one, so a later version would silently lose its parameters.
- The wallmark engine removes at most one expired static decal per material per frame, because it breaks out of the loop it is mutating. It changes the visible decal population, so the twin documents the behaviour rather than correcting it in prose.

## src/xrPhysics (shells, hits, triangle collider, cylinder)
- The explosion impulse is divided by the fourth root of the element count. No derivation exists; it is a presentation curve, not a conservation law.
- A cross-product separating axis must beat the incumbent by five percent to displace it. Clearly hysteresis against normal flicker, but the value is undocumented.
- The contact-budget slack is the flag word minus ten, where at most three contacts can be emitted per call. The extra seven is unexplained.
- The box path of the triangle-list collider pads its half-extent where the sphere and cylinder paths do not. Looks like residue of a specific resting-contact bug.
- A probe-hit depth multiplier is commented out in all three collision cases. The intent (treat a probe hit as an emergency push-back) is inferable; the reason for disabling it is not.
- The circle-intersection routine takes the square root of a negative discriminant and proceeds rather than failing, with the original calling this "somewhat strange". Deliberate, but no justification is recorded.
- The cylinder-versus-cylinder face-to-face path is marked temporary in the original and is still there; the proper manifold it wants was never written.
- The per-triangle and per-batch collider callbacks are declared, settable and gettable, and read by nothing. Dead interface inherited with the code.
- One debug-tracking flag exists in both draw masks with different meanings, next to a comment warning not to confuse them. Which is authoritative is not recoverable.
- The spawn-to-shell application carries a standing todo saying the condition treats "has fixed bones" as "is immobile", which the author knew was too coarse. Unsettled, not decided.
- The two fixed-bone paths (text list versus bone identifiers) differ in strictness, asserting in one and silently skipping in the other. The asymmetry is not justified anywhere.

### Bugs found while recipifying
- The guard before biasing explosion scatter toward the blast direction tests the blast position's distance from the WORLD ORIGIN, which is almost certainly not what was intended. Frozen into shipped behaviour.
- The cylinder's length field is named and commented as being along the z axis, but every routine uses local y. The name is simply wrong.

## src/xrCore (compression, containers, crypto, debug)
- The statistical compressor is full of fitted constants with no derivation: eight binary-escape seeds, sixteen exponential-escape values, two inheritance frequency formulas with their thresholds, the glue budget and the compaction threshold.
- The trained-model writer exists only as a commented-out block, so the format is documented from the reader alone. The flag that would preserve a trained model across streams is also commented out, making the second branch of model startup unreachable and the intended lifetime ambiguous.
- The coder's progress hook fires on a fixed escape count and is bound to an empty function; nothing indicates what it was meant to report.
- The archive obfuscator's key constants are four dates written as decimal digits. Why those dates, and why the two iteration counts differ, is undiscoverable.
- The sorted-array map has two dead features: its lazy-sort hook is empty, and its positional-hint insert has a condition that cannot be satisfied. Its reverse-end accessor is declared with the wrong return type.
- The arena map's unit-size arithmetic describes a 32-bit record layout; on 64-bit targets the packed records are larger, so the constant and the structures it serves can disagree. Whether this is live or masked by the size classes is not determinable.

### Security-relevant observations (recorded, not exploited)
- The hash wrapper FAILS OPEN (an all-zero digest, which compares equal to itself) while the signature verifier FAILS CLOSED. The combination is safe by accident; no comment says which was intended.
- The signature loses leading zero bytes by round-tripping through a big number to hex text. Both sides lose them identically so it works, but nothing shows this was intended.

## src/xrPhysics (character controller, joints, breaking)
- The fracture torque reconciliation factor (ten million) is a pure unit reconciliation between authored torque values and the tensor product's magnitude. Frozen with the game data; no derivation exists.
- The suspension spring and damping constants and the walk acceleration force are bare tuning constants.
- The impulse time constant, the lose-control distance, the climb distance and the whole character-offset set are tuning with no reasoning in the source.
- A slow running average of the character's vertical velocity is written every step and read nowhere.
- Two joint-limit setters are empty bodies with live callers, and the global-time setter has an inverted sign plus a guard its own next line overwrites. Only the resulting behaviour is recoverable.
- The default third joint axis is built as a cross product of the first axis with itself (zero), and one joint kind falls through into another, giving it five axes instead of three. Both are invisible because everything is overwritten.
- Four fields record the breaking blow and are then never read: the code that would apply it to the new fragment is present but commented out.
- Assorted dead paths named but not explained, including a static-root bone callback, a recursive callback setter, a missing non-matching-root case, a second angle accessor, and an empty re-enable body.

### Bugs found while recipifying
- The joint-destruction check compares all four reaction readings against the FORCE threshold; the squared break torque is stored and never read. The shipped data is balanced against this.
- The fracture update adds the first side's gravity to both sides, and attributes a first-side impact's force to the first side but its torque to the second. Both are load-bearing for shipped break thresholds.

## src/Layers/xrRenderDX11 and xrRenderPC_R4
- Fluid: the dynamic-obstacle velocity scale is physical in its reciprocal-timestep part and hand-tuned in the rest, with three superseded multipliers preserved in comments. The march oversampling factor multiplies the cube diagonal by an undocumented constant. Emitter jitter applies to velocity but never density, is flagged in-source as a hack, and is not scaled by grid size. The density oscillation's phase offset is unobservable, and saturation also lowers mean density, possibly unintended.
- Fluid: one buffer is bound at the top of the draw only to establish a constant-buffer layout; why every pass shares one layout is documented nowhere. One effect has a pass slot but no compiled pass.
- The exact compiler diagnostic that triggers the backwards-compatibility retry is matched by error-code string, and the source itself asks whether there is a better way.
- The level-of-detail curve is a fifth power scaled by eight with a ceiling of seven. The shape's consequence is derivable; the exponent and the eight-versus-seven mismatch are not.
- One renderer rank is commented out because vanilla shaders fail to compile with static sun; whether any shipped data set works with it is unrecoverable.
- Nothing outside one file adds a shader option, so whether the option list was meant as a mod-facing surface is undeterminable.
- The shadow cascade box's near face spans only the far thirty percent of the cascade's depth range, underived and not tied to any configured split.
- The depth-bias sign convention differs across three paths, one marked a work-around for inverse culling in the far region.
- The doubling of ambient and environment colour in the combine pass is reproducible but underived; levels come out half-lit without it.
- Two occlusion quality settings both march the same number of directions, so one quality step changes nothing.
- Cloud-shadow wind speed is read and discarded; only direction is used, with a hardcoded drift.
- One accumulation target is named for luminance but measures nothing.

### Bugs found while recipifying
- The downsampled-occlusion branch runs the depth downsample and then never calls occlusion; the call is struck out.
- The shadow cascade box submits sixteen triangles when only twelve are initialized.
- Fluid boundary lines are submitted as a triangle list with a primitive count computed for triples while the vertices describe two-vertex segments, and their cell coordinates are all zero. Either dead or rasterizing garbage.

## src/xrGame/ik, CdkeyDecode, gamespy
- The key checksum's rolling multiplier is unexplained vendor trivia. Odd and 32-bit is all the arithmetic needs; the specific value must be copied verbatim.
- The hinge-angle tie-break takes the second trigonometric solution when both are positive, with a bare question-mark comment in the source. Unreachable for leg goals, so no behaviour pins it down.
- The joint-limit epsilons and the union tolerance are settled-on values. Only their ordering (tight, then epsilon, then loose) is derivable.
- The pretty-printer puts the SHORT group first rather than last. Plausibly right-alignment on a printed box, but nothing says so, and the game never calls it.
- The base-32 short-group rule silently drops leftover bits with no pad character, so a truncated key and a short key are indistinguishable. Whether that was intended or simply never hit is not recoverable.
- Two matchmaking callbacks are empty and the server self-authentication callback is commented out: the integration was never finished, so what it would have done is not recoverable from this repository.

### Bugs found while recipifying
- The interval-union routine has unbounded loops that cannot terminate. Reachable only with joint limits enabled, which shipping never does.
- One key-verification entry point omits the decoder's length guard and reads its checksum from a nonsensical offset on a short key.

## src/xrServerEntities
- The anomalous-zone record at one exact version writes one extra 32-bit word. The source annotates it with an expletive and nothing else. No reason discoverable; must be reproduced.
- The trader mixin below one version reads a 32-bit word, asserts it is zero and discards it. No record of what it ever meant.
- One spawn-message option bit is unused, with no trace of what occupied it.
- The rank and reputation deltas are both ten, on scales running to hundreds. Pure tuning, no derivation.
- The maximum conversion-chain length for the fast cross-cast is hard-clamped to one in every build, while the search machinery is written for arbitrary lengths. Compile time is the plausible motive; nothing states it.
- The creature network update writes four full-precision angles with the quantized calls sitting commented out beside each. No reason for the reversion recorded.
- The 8-bit signed property creator is commented out in the script property export while every other width ships. No explanation.
- Smart-cover loophole animation names are synthesized by a string convention that replaced a proper derivation from the transition table, which is present but commented out. Why the switch happened is not recorded.
- Rank is exported twice under one script name on the human record (combat rank from one level, social rank from the identity mixin). Which one wins depends on the binding layer's overload resolution; it could not be determined statically, and shipped scripts read it.

### Bugs found while recipifying
- The profile loader's class list reads element zero n times instead of the i-th.
- The float property handle is registered under the 32-bit-unsigned handle's script name.
- The visual-filling routine returns instead of continuing, hiding every figure after a non-enterable loophole.

### SECURITY (network-facing, multiplayer only)
- The actor's corpse update reads an attacker-controlled bone count from a network message into a fixed 1024-byte stack buffer with no bound check. Reachable only when multiplayer is built in (the default build ships the null transport). Recorded in the twin; not exercised.

## src/utils and src/xr_3da
- The baked-light record's energy and emitter-triangle fields are meaningful only inside a radiosity solve whose compiler is not in this repository. Nothing reads them; their units and conventions are unrecoverable.
- The profile award date's epoch and encoding: fetched, cached and printed as a bare integer end to end. Nothing interprets it.
- The record store's shared secret and the service account's password are present verbatim, but why those bytes, and whether the buffer's zero tail was significant to the service, is not discoverable.
- The verifier's one-megabyte decompression ceiling is a judgement about honest dump size with no stated basis.
- Tie-breaking between two equal-length prefix rules in the balance tool is resolved by whatever order the sort left, which is not stable. The shipped job description contains no such pair, so the intended rule is unknowable.
- Whether six profile-server defects are bugs or compensating behaviour is genuinely ambiguous: an inverted name-column test and a wrong name space cancel each other out.

### Bugs found while recipifying
- The archive-difference tool's help text advertises one spelling of the checksum flag and the parser matches another, so the advertised option does nothing.
- The same tool parses and stores a file-age flag that nothing ever reads. The check appears to have been removed and the option left behind.
- The profile server uses a processor-time clock where wall time is meant, mis-scales a sleep, runs tasks under the queue lock, and hangs on an all-rejected batch. Its cache sweep is near-inert.

## src/xrGame (relations, rats, script binding)
- Sympathy scales every faction-level goodwill award and has no stated range. Nothing says whether it is bounded by one, so a single character could in principle move a faction more than the configured award.
- Two configuration keys are misspelled in the shipped data ("neutal" for "neutral") and read under the misspelling. Frozen: a rebuild must reproduce the typo.
- Rat state asymmetries with no discoverable reason: which states push versus change to the no-way state; a goal timer renewed by exactly ten seconds each update; one retreat state registered but unreachable from any shipped transition; a dead pursuit branch behind a guard that already popped.
- The nested-planner action wrapper omits the weight floor its plain-operator sibling enforces. The admissibility argument applies equally, so it reads as an omission, but nothing says so.
- A script action-condition flag has a bit and a name but is exported nowhere, so no shipped script can set it.
- One relation action reserves a bit, is raised by no call site, and has no case in the handler.
- The anti-cheat screenshot relay's four random padding values are documented in-source as length-matching another message, but nothing enforces the coupling; if that message's layout changed, the disguise breaks silently.

### Bugs found while recipifying
- The refreshable obstacle query compares the clock against the small radius plus the interval instead of against the last refresh time. Narrow queries for the first second of the process, wide ones forever after; the recorded timestamp is never read.
- Fight bookkeeping captures the defender's opinion of the attacker at the start of a fight correctly, and the kill handler's use of it is commented out. Shipped behaviour judges a kill by the relation at death, so provoking a neutral and killing them scores as killing an enemy. Two more fields are written and never read.
- Clearing a character's relations leaves every other character's opinion of it keyed by the recycled 16-bit identifier. Latent; the shipped games never reach the wrap point.
- A failing script binder is detached silently, so a broken binder presents as an object that quietly stops behaving.

## src/xrGame (script action channels, game-object facade)
- The particle action channel creates an effect and the destructor's deletion is commented out. Ownership is genuinely undetermined: either self-removing effects clean themselves and others are adopted by the playing entity, or the delete was removed to stop a crash and non-self-removing effects leak.
- The fixed particle slot identifier in the facade is an arbitrary large number with no discoverable meaning beyond being outside the engine's own range. It silently limits an object to one bone-attached effect.
- The loophole direction-distance default and the smart-cover dwell-time defaults have no discoverable justification.
- A one-entry constant table duplicates a message value, is unrelated to class identifiers despite its name, and has no reader in the source.
- One facade method exists and unconditionally aborts: a placeholder never implemented. Whether it was meant to exist is unrecoverable.
- Several action-channel constructors get their goal kind purely from statement order. Invisible and fragile; the twins say to set it explicitly.

### Bugs found while recipifying
- The movement selection setter logs a wrong-type error and then dereferences anyway.
- One dialog accessor discards its result.
- The additional-walk-weight accessor reads the carry-weight field.
- The animation adder reports a smart-cover conflict without returning.
- The trader sound-action constructor clears the completion flag AFTER the setter may have set it, so a missing sound plays forever instead of being skipped. (Elsewhere in the same batch, a missing sound file correctly marks the channel completed so a typo cannot hang the action queue.)

## src/xrGame (sight manager, smart covers)
- Sight timing constants with no derivation: the lean-out glance dwell and amplitude, the aimed-fire commit distance, the settle interval, the re-latch movement threshold, the glance head speed. None are in configuration, so a rebuild cannot retune them without a code change, and nothing explains the values.
- Every loophole and action probe is lifted two metres before the level-graph containment query. The problem is recoverable (the authored eye point can sit below or inside geometry and the graph query searches downward); the specific height is not.
- The "less cover look" sample counts, and a minimum-gain threshold that was multiplied by zero rather than deleted. The intended minimum is unrecoverable.
- The "last hit was long ago" window driving blind fire and suppression, and the firing-spell limit with its fraction-of-a-magazine waiver. All three shape combat visibly; none is configurable or explained.
- The convention that the third animation marker is the smart-cover event (the first two are footsteps) is a frozen contract with the shipped motion data, with no in-source statement of why three.
- A loophole-preference score starts high, only decreases, has every caller pass zero as the decrement, and is read by nothing: built and never wired.
- A three-times-repeated todo marks an intended loophole-selection behaviour (move to a loophole that covers the target instead of staring at the arc edge). Whether it was abandoned deliberately is not recoverable.
- The weapon animation-slot variants are compiled out of shipping builds; whether the shipped motion data still contains them is not determinable from source.
- The danger arc fields are guarded by a comment asking for a version check that was never written. Covers authored before the danger arc existed get a forward default and a zero-width arc.

### Bugs found while recipifying
- The smart-cover containment test returns true on the inner side of ANY ONE of the six face planes instead of all six, and its sphere arm tests an untransformed centre while the box arm transforms.
- Loophole evaluation selects the winner by raw angle but writes out the arc-normalized value, so the comparison and the reported score are different orderings. Fixing it changes which loophole creatures pick in shipped covers.

## src/xrGame (space restrictions, stalker animation)
- The "die in anomaly" switch defaults to off, so shipped creatures do not perceive anomalies at all. Whether that preserves the original or is a disabled modification is undeterminable.
- One look-back delay constant is zero and exists solely so it could be non-zero. No discoverable intended value.
- The debug animation-stats record stores a visual name that nothing reads or prints, so the report merges models it was probably meant to separate.
- A live-composition counter is incremented and decremented but never read in the shipped path: a leak tripwire with no reader.
- The commented-out restriction-accessibility branch in the gather-items action is live in the smart-terrain operator, so the omission looks deliberate, but the reason is not recoverable.
- The composition-level correctness check treats a flood that hits the 65535 cap as correct, which is exactly what a leaking border looks like.
- Preparation carries an authored sphere's radius through unscaled while rebuilding boxes from transformed corners. Correct only because no shipped restrictor is scaled; the assumption is undocumented.

### Bugs found while recipifying
- The leg-animation assignment retries the selection on an invalid result, discards the retry and returns the invalid value anyway. Its only effect is running the selection's side effects twice.
- The restriction intersection test uses a sorted-set intersection assuming identifier order, while borders are sorted by packed horizontal position. Latent only because its caller is compiled out; a blocker if that path is restored.

### CORRECTION to GLOSSARY.md (applied)
Restrictors do NOT intersect. Within a list the volumes UNION; the two lists then SUBTRACT: effective space = (union of permitted) minus (union of forbidden). Naming two permitted regions widens the space. A restriction naming an unspawned restrictor stays inert rather than forbidding everything.

## src/xrGame (planner evaluators, search, trade, steering)
- Bare constants with no derivation: the grenade self-blast radius, the "player in my way" width, the wounded-enemy reach; the group combat assessment's cache window and radius (written longhand at all three call sites, so it is unclear whether they were meant to be one tunable); the ambush cover search's two widening radii; the enemy-memory horizon in the ambush hold; the step manager's audibility cut and sound-emitter lift; trade's distance band, facing window, condition-curve floor and exponent; the trajectory chord and landing tolerances.
- The low-cover evaluator always returns false: the real computation is present but disabled, with the author's note that several other conditions were still needed. A whole sub-planner is registered against that property and is therefore unreachable in the shipped game.
- The four rat-flocking headers are dead: nothing includes them, none of the three implementations has a body anywhere in the tree, and they collide in namespace with a live, unrelated type. Recorded as abandoned design, not carried forward.
- One test translation unit is not compiled into anything and contains lines its author annotated as deliberately failing to build.
- Trade dead state: a last-trade timestamp is stamped and never read (a removed cooldown); an artefact-task flag is set and cleared but never read; one accessor exists only to name a disabled alternative in which a trader's goods came from a separate store.
- The trade parameter constructor crosses its hostile and friendly factors into a pair named the other way round. No price changes, because the interpolation is written to work whichever way the pair is ordered and clamps to the pair's own range - very likely why it is written that way, but that is inference.
- Ground contact gates the footstep sound but not the dust particle, with no recoverable reason.
- One detector class is registered under a script name that differs from its internal class name. A rebuild must carry the registered names, not the internal ones.

### Bugs found while recipifying
- The steering distance clamp takes the minimum where the stated intent requires the maximum, so the force grows without bound as distance approaches zero. The file's own local "max" helper is itself written as a minimum, which is the likely origin. Fixing it changes the feel of every behaviour with a non-zero inverse term.
- The trajectory segmenter is passed the start point where a velocity is expected, so segment lengths are chosen against a different arc than the one tested. A shadowed axis variable also leaves the box orientation built from an unwritten axis for any trajectory with a horizontal component.

## src/xrGame/ai (bloodsucker, boar, burer, cat, chimera)
- The animation hit-time fallback is a magic constant with no derivable basis.
- The bloodsucker feed gate draws a value that can be negative, which makes the gate a no-op. Why a negative hit requirement is admissible is not stated anywhere.
- The chimera reads and stores a forced-attack distance from its section; nothing ever reads it back.
- The chimera's minimum-run-distance calculation names its terms for the half scan angle but feeds them the whole angle. Intent unrecoverable; shipped behaviour is the whole angle.
- The burer's shield measures the raise clip's length at entry and never uses it. The commented-out condition shows the shield was meant to come up only after that clip played.
- One burer attack hard-codes its damage and impulse, the only burer attack whose numbers are not authored. It reads as placeholder, and the ability is registered but never activated - its one call site is commented out.
- The cat's retired jump-turn omits an angular-speed factor the boar's has. Tuning difference or an unported fix, not recoverable.
- Dead subtrees recorded as such: the chimera hunting tree cannot compile (one state header is a verbatim copy of its sibling and defines the same method twice); the chimera threaten tree is registered nowhere; one burer melee state is in the substate table but never selected.
- Declared-and-never-touched fields across three creatures: four shield-state fields on the burer's anti-aim state (an untrimmed copy of the shield state's declaration), three prediction fields on the chimera attack state, a morale threshold nothing consults, and the cat's jump-turn timestamp with its companion constant.

### Bugs found while recipifying
- The chimera's jump-target scan increments its index twice per iteration, so only half the scan directions are tried.
- The chimera's move-target scan loops over the move points but tests success against the pounce points, so a fully failed sweep still reports success.
- A non-fire-wound hit landing on a raised burer shield falls through both branches and is silently absorbed.

## src/editors/xrWeatherEditor
- The nudge steps (an absolute step for an unconstrained real, a four-hundredth of the range for a limited one, a fixed step for a vector component) are self-consistent and the reasoning they imply is recorded, but no comment, commit or data file justifies either number.
- Which weather properties are registered as typeable and which are locked to a chooser is decided in engine-side registration code outside this directory. The rule is recoverable; the table is not from here.
- Four presentation flags are plumbed through every one of roughly thirty-five overloads and discarded by all of them, and the value handle the interface promises is never produced. Unfinished work or an interface that outlived its implementation could not be determined: no caller passes anything but defaults.
- One container marker interface is empty. What it marks is inferable from its two implementors but never stated, and the concrete recovery in the converters goes through it by cast, so its emptiness is load-bearing in a way nothing explains.
- A compiled-out thumbnail file dialog has no record of why it was abandoned.
- Six adapters fall back to entry zero and none defines what happens if the list is empty. Whether an empty list is impossible by construction or merely never happened could not be established.

### Design facts worth keeping (written into the twins)
- A "named set" stores the authored number while a "label list" stores a POSITION, so reordering such a list silently repoints every document written against it.
- Reading is clamped as well as writing, so a document holding an out-of-range value displays in range and is quietly corrected by the first edit.

### Bugs found while recipifying
- The colour picker is seeded by truncating rather than clamping, so an overbright channel opened and confirmed untouched comes back as something unrelated.
- One vector property leaks its native trampoline: the container is released, the trampoline is not. Written up as an ownership-ordering requirement rather than silently repaired.
- One converter formats text it then declares it cannot convert back. Deliberate (it suppresses text editing of the parent row) but inconsistent.

## src/xrGame/ai/monsters/states
- Almost every constant in this directory is compiled in, identical for every creature, and derived nowhere: the hunger timer and eating cap, sniff and rest pauses, the flee and cover bands, the threat display, the sweep steps and probe distance, the flee-from-sound distance, the panic exit, the squad pauses, the idle period and its wrap, the loiter ring and follow slack, the stalk band, the corpse impulse multiplier.
- Only seven configuration keys reach this whole directory (corpse distance, three eating parameters, three sound delays). The one real piece of authoring leverage is an aggressive flag on a HOME REGION rather than on the creature, which switches gait, acceleration profile and voice together.
- One cover-path parameter set and one "near cover" band repeat verbatim across many unrelated leaves: one tuned band reused, with no record of what it was tuned against.
- Whether the eating cap equalling the hunger timer was intended. The coincidence means a creature finishing a full sitting is immediately ready to be hungry again.
- Why the danger home-point state has no retry loop against drawing the creature's own cell, when the otherwise-parallel combat version retries five times twice.
- Whether the "fun" rest state was ever selected in an earlier revision, or was written and never wired.

### Dead behaviour
- The "fun" rest state is fully implemented (it shoves corpses with physics impulses), registered in the solitary peacetime composite, and selected in NO file in the repository. Its gating field is zeroed on entry and never read.
- The sleep state is registered by the solitary peacetime cascade and never named by it; only the pack variant selects it.
- Six more declared-and-unused fields or constants, each flagged in place, including one whose only use is commented out.

### Bugs found while recipifying
- The distress-response state fills the wrong parameter record: its look-around leaf is the facing variant, but the composite writes only the plain record, so the facing point is never set and the creature turns toward whatever that leaf last held.
- The minimum hide time is declared in seconds and added to a millisecond timestamp, so the guard has never held back a frame and the leaf exits on distance alone.
- The eating state claims the memory component's CURRENT corpse rather than the snapshot it is acting on.

## src/xrGame/ui
- The death marker's rectangle in a shared multiplayer icon sheet is four literals with no registration and no comment.
- The quantisation of all four weapon statistics is consistent with a fixed-width bar track, but nothing states it.
- The sleep dial's pixels-per-hour against its strip width leaves several pixels unaddressed. Unexplained.
- The talk camera's turn factor, threshold and cap, and the head-height stand-in - the latter marked twice in-source as something that should track the head bone.
- Hover dwell delays differ between a task-list row and a task panel with no stated reason.

### Design facts worth keeping (written into the twins)
- Six task notifications are the ONLY way the task list and task panels reach the map; the talk window splits the same way, notification one way and method call the other.
- Two inversions a rebuild will get backwards: the task row's view check box is checked when the spot is HIDDEN, and the voting menu's subject is gated by the NEXT mask bit.
- The artefact-parameter normalisations are mirror images: restoration rates divide by the player's own rate only in the oldest game's data, immunities divide by the player's max protection only in the newer games'.
- One drag-drop initialiser is where every inventory and trade grid rule enters from data, and its custom-placement flag defaults ON while every other flag defaults off.

### Dead or emptied files, all still in the build
- The trade window is emptied (the trade screen is now script over this directory's pieces) while another header still forward-declares the removed type.
- A text-vote screen is wholly commented out; a wheel menu loads a layout, returns failure and is never constructed; a text banner is commented out of the build description and includes a header path that no longer resolves.

### Bugs found while recipifying
- The server-info logo path decodes a JPEG only to validate it, then writes the compressed payload out under a texture extension. Marked with an in-source to-do.
- The skin selector labels a zero shortcut for the tenth skin but only matches the digits one to nine.
- One stats icon copies a material handle into both team slots, so teardown releases it twice.

## src/editors/xrWeatherEngine
- Four sun sub-records (flares, flare, blend, gradient) are complete, compiled and unreachable: nothing constructs them, the sun holds no reference to any, and each declares a save that is never defined. Abandoned work or work never wired is not discoverable. The twins record the shipping defaults, since those defaults are the only specification of those fields anywhere in the repository.
- The level manager's save is declared and never defined anywhere; its only call site is commented out with a note that saving happens when the editor exits. The inferred mechanism (two write-on-close configurations held open since load) is not stated.
- The shader-library chunk that holds material-pass names is an unexplained constant: a format fact of a file this module only reads.
- The small-sky-texture suffix is a derived companion name the engine relies on. Nothing says what makes it small or where it is resolved.
- The units of the sun blend's three times are stated nowhere; the twin describes them by role.
- The sun altitude and longitude map to the direction's heading and pitch in the reverse of the intuitive order. Load-bearing, with no comment explaining the choice; it may simply be how the first cycle file was authored.

### Bugs found while recipifying
- The thunderbolt altitude is a two-component value of which only the first is read, and the whole environment file is rewritten on save, so the second component is silently lost.
- Two identifier caches never clear their change flags, unlike every sibling cache.

## src/editors/xrSdkControls and the editor module boundary
- Why the text conversion buffer is sized at twice the length plus one: sufficient, but nothing says which encoding it is sizing for.
- The colour drag factor and the per-component step are bare feel constants with no justification anywhere.
- The numeric spinner's signed accumulation counter runs against a threshold of one, so the sign can never matter. Looks like hysteresis that did not survive a tuning pass.
- The colour picker's alpha-row shift is one pixel larger than the row pitch. Off-by-one or deliberate; unrecoverable.
- Adding a node group sets the new node's kind to single-item. Named as a bug, but it cannot be ruled out that something depends on folders being selectable.
- The path walker creates its first segment even in non-creating mode, and the node getter ignores missing middle segments. No rationale discoverable; both look like accidents.
- The tree view's root is fetched by reflection in the wrong visibility and is always null, so the node getter walks from nothing. Whether any live call path reaches it could not be determined.
- One sub-node selection routine is dead: missing feature or leftover, the source does not say.
- A property container's category list is never constructed or populated but is cleared on reset: dead state from an earlier version of the category rule, which was replaced by reading categories off the ordered descriptions.
- One collection add returns a value two off from the insertion position, and one copy routine indexes source and target identically. Both unexercised.
- The display-name buffer has no truncation signal and no statement of why its size was chosen.

### Two complete features that are inert in the shipped editor
- The tree filter panel's text box is never bound to its complete, correct filtering algorithm.
- The colour picker is a full alpha-aware live picker that is never opened, because the double-click body that would open it is compiled out. The likely reason is structural: a modal dialog starves the engine's idle pump, so any picker freezes the rendered view. That one consequence of inverted control explains several otherwise-arbitrary decisions in the chapter.

### Bugs found while recipifying
- The real-number converter prints locale-independent and parses locale-dependent, so on a comma-decimal machine the editor refuses its own output. The most consequential defect in the directory; whether it was ever noticed cannot be told.

## src/xrGame/ui (map, buy screen, overlay, PDA)
- The shelf-accelerator draw remaps one particular accelerator value before converting it to a digit. No reachable accelerator produces that value; the intent is not derivable. Two other places compute the accelerator cut-off with different bounds - one of them is wrong, and which is not determinable.
- The overweight margin, the actor-move dead zone in the map screen, and the condition strip's two band thresholds have no configuration entry and no discoverable derivation.
- The warning-threshold key table has seven entries and one is read. The other six name icons removed from the overlay, and shipped configuration still carries their values. Nothing consumes them.
- One weapon is special-cased onto a different icon atlas with no stated reason; and whether the engine's own elapsed-time line occupies the first statistics slot is decided by a proxy test the source itself labels a hack.
- A hint entry point and a development adjust mode are declared with empty or absent bodies. Whatever they displayed is gone.
- Unit confusion in three places: widths are measured in screen units and applied to canvas-unit rectangles. Intentional tuning around the shipped layouts or a bug is not determinable.
- The four server-admin "weather" buttons set four specific times of day that match the shipped environment definition; nothing records why those four.

### Design facts worth keeping (written into the twins)
- The map's metres-to-canvas transform and its drawing aspect flag; the map view's goal-driven planner (zoom out, move, settle) and its three animation constants.
- The buy screen's item state machine is what makes cancel reversible: a bought-then-sold item returns to the shelf, a sold-then-bought item returns to the player's own inventory.
- The overlay's ten-frame condition stride and its per-weapon misfire grading; the PDA's section-name page table and its script override point.

### Bugs found while recipifying
- The "other team" badge index reduces to a constant zero, so the other team's colour is unreachable.
- The overweight indicator's two upper branches are identical, so the middle colour never appears; the removed band's threshold is gone.
- The team-section guard asserts a line does NOT exist while its failure message says it does. One of the two is wrong, and the check is compiled out in shipping builds.

## src/xrGame/ai (stalker, trader, zombie, tushkano, phantom, telekinesis)
- The half-turn in the unprotected-area look state: it queries the LEAST covered heading and adds half a turn, which reads as facing toward cover. Whether the sign is a bug or whether the underlying query already returns the reverse heading cannot be decided from source; must be checked against a running original.
- A lift computed at entry of that same state and never used.
- The grace period in the extended move-to-point completion check has no derivation; only that the path builder's end-of-path flag is meaningless before a route exists.
- The four cover parameters in every extended move are identical for every creature and every situation, untunable from data, with no record of where they came from.
- The object-shove state's four constants, and why its acceptance cone tests pitch against the same half-angle as yaw.
- Telekinesis: the two height comparisons are strict against the same value, so the random-wobble branch is unreachable, and the hover band is centred above the nominal target height. Whether that offset was a band or an offset is not recoverable. The raise step is scaled then discarded, and the element-count scaling is an unexplained calibration. One update interval is defined, its use commented out, and the timestamp it maintained is still written every call.
- One weapon-type slot is skipped in both conversion tables with only a note that it is unused in release data; its meaning is lost.
- The grenade-throw sound fallback is registered with the same mask in both branches; looks like copy-paste, nothing confirms it.
- The trader's schedule bounds carry a source comment saying the relation to network latency is broken; the original relation is not recoverable.
- Two trader artefact-order entry points are stubs of a removed generated-quest feature that cannot be reconstructed.
- The phantom's export writes yaw twice and never writes bank; whether that is quantisation-era residue is unknown.
- The tushkano's state manager fetches a corpse and discards the result; needs verification against a running original that the query is genuinely pure.
- The tushkano's enemy rung has no default case, so an enemy whose danger rating is neither strong nor weak leaves no behaviour selected. Whether such a rating is reachable was not determined.

### Dead behaviour, stated as dead
- Four monster states are registered by no creature; one is dead twice over, because its header includes a sibling's inline file so its own is included by nothing, and its body is commented out anyway.
- A superseded telekinesis controller and a position-prediction header are in the build and included by nothing.
- A weighted-random helper is included once, never named, and draws from the C library's global generator rather than the engine's seeded one - a determinism hazard against conformance item 8.
- Shipped test scaffolding is live behaviour: the cat's threat state IS the debug stare-at-the-actor state, and the snork's enemy search IS a test-cover state, so the snork does not search - it walks to an assigned cover cell and stands there.
- A zombie head and spine rig is built with its callbacks commented out, so the rig never runs. One creature's special-parameter check is empty while the tree sets its input every tick. A retreat state's four authored distances are read by nobody and its distance-based completion is commented out, so retreat only ends on a timeout.
- The phantom's save and load are empty, so a phantom does not survive a save and reverts to its birth state; the spawn comment claiming otherwise is stale.
- The trader has no engine-side brain at all: its think entry point is empty.
- A grenade-reaction delay is multiplied out to zero. One walking-in-danger sound is registered and played nowhere; a friendly-fire injury line is loaded and never played.

### Bugs found while recipifying
- The weapon-accuracy dispersion multipliers are SWAPPED in both stand and crouch branches, so shipped stalkers are more accurate when not aiming.
- The fire-queue section override is never applied: its emptiness test is true for any non-empty string, so the section is always reset to the creature's own.
- The too-far-to-kill test returns false unconditionally, so stalkers engage at any range with any weapon.
- The recoil aim effector returns its input, so stalker aim is unaffected by recoil despite six guarded call sites.
- The advance-cover gate is a constant false, so the advance search always runs; it also replaces the cached cover without notifying subscribers or recomputing the cached value.
- The bone-protection reset overwrites its section argument before use.
- The multithreaded vision path is hard-disabled by a constant-false conjunct.
- A script-reachable binding binds the enemy-killed sound name to the interruption MASK rather than the sound identifier.
- Telekinesis clear paths erase entries without destroying the records (leak), and one accessor returns a polymorphic record by value (slicing).
- One zombie state fetches the squad command before null-checking the squad handle; it survives only on a data invariant, not a code one.

## src/xrGame/ui (HUD states, upgrade tree, menus)
- The per-game weapon-icon scale table, its pixel offsets and its two cell clamps are pure visual tuning, in code, underivable.
- Why the demo playback control closes on the crouch binding.
- One aspect factor is written out four separate times (and once as its reciprocal). The source itself notes it should be a shared constant; which places are deliberate asymmetries and which are oversights is not recoverable. One of them - an end-cap-only correction on a frame line - is in the shipped look.
- Two script-mode magic numbers stand for talk-screen shown and hidden, with no enumeration; both ends agree by convention.
- Assorted literal geometry with no derivation: the stacking gaps and margins in the upgrade info panel, a nudge in one frag list, the state strip and point-marker sizes in the upgrade tree, the ghost alpha in the quick-use bar, and two different texture segment counts for health and condition (both are art).
- Whether the menu spinner's detent click firing in the first tenth of the spin, and the first game reusing the second game's narrow loading layout, are deliberate or inverted.

### Nine files in this slice are not in the build
- Four are commented out of the build description; three more are commented out in their entirety. Their twins say so and record only the decisions that survived elsewhere.

### Incomplete rather than broken
- The log tab's batched row build has a public pump nobody calls, so a day past thirty entries shows thirty.
- An upgrade-texture verification routine is spelled with a typo and is never invoked.

### Bugs found while recipifying
- The upgrade-tree marker never stores the node its constructor is handed, so any pointer interaction with it dereferences nothing.
- The overlay's zone nearness factor always divides by a constant because an addition binds tighter than the conditional it was meant to select. Per-hazard feel radii are therefore ignored - and this IS the shipped detector feel.
- The server row loads the anti-cheat texture into the account-required icon slot.
- One capacity clamp has a lower bound above its upper bound.
- One time-period formatter's month arithmetic is wrong.
- One frame list reads its tile count off an already-overwritten path buffer; the shipped data falls back to the default.

## src/xrGame/ai/monsters (state machinery, pseudogiant, snork, rats, psi-dog)
- The state-identifier scheme counts attack members rather than allocating bits, so six unrelated members all satisfy the attack-camp membership test. Nothing asks today. "Heard a call for help" is also numbered inside the interesting-sound family, so it answers yes to both.
- Compiled-in timings with no derivation anywhere: the find-enemy delay, the flee cooldown, the attack-camp watch and creep pair, the ambush minimum and the cover annulus.
- Pseudogiant: the step radius, the shake-power divisor, the step effector's envelope divisor and axis ratios, the object sweep radius and two impulse coefficients.
- Rat: the nest-migration re-arm window (written twice), the food-per-bite ratio, the standing-position radius, and a corpse cost of food squared times distance with LOWER winning - which makes a better-stocked corpse LESS attractive. Whether that is a sign error or the foraging-spread mechanic it produces cannot be told.
- Why the rat idle voice got the top sound-mask bit while attack and eating share the one below, so a rat may chitter while eating but not while biting.

### Dead behaviour, stated as dead
- One rat state machine is commented out of the build AND would not compile if restored: it names renamed states and members and would collide with the live think entry point.
- A scanning ability is included by the burer but not inherited and never instantiated; its one survivor is a static flag now written by nobody. A psy-aura class has no derived class, no construction and no include outside its own implementation. A snork jump class is entirely commented out, superseded by shared machinery.
- The snork's enemy-search state is wired to a test-cover harness and the only line that would select it is commented out - a developer harness left in the shipped build.
- One attack predicate is declared and never defined, and the field it would use is zeroed at entry and never written, so the flee test's first guard is permanently false. Only the group variant implements it.
- The run-attack's path feasibility test is commented out, so the charge starts without checking there is room, and the computed target point is discarded. Three more of its fields are inert.

### Bugs found while recipifying
- The psi-dog's no-cover fallback sends the dog to mesh vertex zero: an arbitrary map corner, not a considered location.
- One rat activation path forces max speed then immediately overwrites it with the speed lottery; the equivalent elsewhere has the branches the right way round.
- A goal-time setter ignores its parameter, and the morale broadcast ignores its radius, so a rat death demoralises the whole group regardless of distance - the authored death distance is read and never used.
- The rat network record writes the coarse vertex twice and a distance twice, and import reads neither distance. Protocol fossil, frozen.

## src/xrGame/ai (bloodsucker, controller, psy-dog)
- Why the bloodsucker's own attack composite was disabled: the edit is deliberate, the code is complete, nothing records a reason. Dead with it: back-approach, wounded withdrawal, the reactive stalking loop.
- The chimera "come out" state's intended body is unrecoverable: the file that should hold it is a byte copy of its sibling.
- Bare in-code timings with no derivation: the camp relocation period, the wound threshold, the circling and behaviour-flip windows, the psy-dog aura's two windows, its radius and fade, the feed effect's two oscillations, the camera drag's ideal distance and wobble.
- The drag jump names a level-specific animation and bone as string literals in engine code.
- Two orphan constants above the controller's selector belong to a find-enemy branch that lives in the shared composite instead.

### Dead behaviour
- The controller's psychic-fire state is included by its attack composite but never instantiated, while its identifier is reused by the pack library. The controller sprint state is unregistered with its destination logic left as a comment.

### Bugs found while recipifying
- The vampire post-process effector's oscillation normalisation simplifies to dividing by one, so it spans the whole effect rather than the middle section it appears to target.

## src/xrGameSpy
- The matchmaking integration was never finished: two callbacks are empty and the public-address callback that would have driven server self-authentication is commented out with its path still present and uncalled.
- With the service gone, the three degradations are: accounts fall back to offline profiles, statistics to zeroes that scripts still read, and remote key authentication FAILS OPEN. The last is a security-relevant default, harmless only because the service it would have consulted no longer exists and the shipped build has no working transport anyway.
- Two recurring defects across the module: callback contexts held in stack frames, and unthrottled outage logging.

---

# Found by the rebuild tests

Three engineers were given this recipe alone — no source, no prior knowledge of the engine
— and asked to plan a rebuild of one module each in a language the recipe never names
(the collision database in Rust, the deferred renderer in Zig on WebGPU, the alife
simulation in Go). What they could not proceed without is listed below. Items that turned
out to be errors in this recipe have been corrected; these are the ones that remain.

## Gaps that remain

- **Smart terrain has no specification.** Every offline decision bottoms out in
  `task_for`, `suitability_for` and `enabled_for`. All three are called, all three are
  fatal on failure, and none is defined anywhere. The suitability metric, the capacity
  rule and the job record are absent, and conformance item 10 freezes their script
  signatures by name.
- **The alife tick has no rate.** Not the scheduler's period, not the switch radii, not
  the travel speeds, not the per-creature search interval. The machinery is specified
  exactly and how often it runs is nowhere, because those values live in shipped
  configuration this recipe treats as out of scope. For a weapon's damage that is the
  right call; for the clock of the simulation it is not.
- **The collision module's result type has no record block**, alone among its types, so
  the field order, the meaning of the distance for a non-unit direction, and which two of
  the three barycentric coordinates are reported are all unstated.
- **The renderer's packed geometry-buffer layout is the shipped default and is
  unspecified.** The console flag that selects it defaults on; what it packs where, in
  what encoding, and where the material id and hemisphere factor go once two channels are
  consumed, is never stated.
- **The transform that maps a unit sphere or cone onto a light's volume** is promised in
  two places in the light record's page and written in neither, though every lighting draw
  and every stencil-mask pass sets it.
- **The bump-map pair's suffix spelling is contradicted across four pages**, and they also
  disagree on which half of the pair carries height. Three consumers bind one spelling;
  the loader produces the other.
- **Several tie-breaks are undetermined**, each of which changes a built collision cache
  byte for byte: which triangle wins at exactly equal ray distance, which axis wins on a
  variance tie, whether the positive subtree is emitted before the negative one.
- **Three screen-coverage heuristics disagree** while one of them carries an invariant
  requiring that it match the others.
- **The graphics seam's "pluggable" verdict is tested only against APIs shaped like
  Direct3D 11.** A WebGPU rebuild hits five blockers the recipe never warns about:
  same-frame occlusion-query availability, using the accumulator as target and input at
  once, user clip planes for light shafts, the absence of shader reflection for the frozen
  sampler names, and the map-discard vertex path.

## Errors the tests found in this recipe, since corrected

Listed so the reader knows what kind of mistake to keep watching for.

- The preface specified the wrong tree build for the collision seam, and a result budget
  the module does not have.
- The plane type stated an inward frustum convention; the frustum is built outward.
- The collision triangle's material field was documented as 14 bits in one chapter and 16
  in another. It is 14.
- The glossary described offline combat, communication, trade and free wandering as part
  of the alife simulation. All four are commented out in the shipped engine.
- Conformance item 11 was not checkable as written.
- The sun's phase opened by calling it a directional light with no position; it is built
  as a point light 500 metres behind the camera, and the cascade fit depends on that.
- One multiplayer twin blamed the client for an inverted sprint rule that the server
  causes.
