# Roadmap

What Guarana is working on, in order. Each item is written like an issue: the problem, the plan,
and what "done" means. There are no dates: an item is done when its evidence says so.

- **Now:** in progress.
- **Next:** planned after the current work. Help is welcome now.
- **Later:** wanted, not started.

A performance item is done only with valid benchmark rows that show the gain, made as described in
[docs/BENCHMARKING.md](docs/BENCHMARKING.md). Work that already exists upstream is ported or helped
along, not rewritten: upstream links point to [Lichii](https://github.com/zortos293/Lichii). To
propose an item, open a [discussion](https://github.com/karimalsalah/guarana/discussions) with the
problem and how you would measure it.

| | Item | Area |
|---|---|---|
| **Now** | [N1. Vblank-locked presenter](#n1-vblank-locked-presenter-for-generated-frames) | Frame generation, pacing |
| | [N2. Intel pipeline-cache crash](#n2-intel-driver-crash-when-the-pipeline-cache-is-saved-during-builds) | Shaders, Intel |
| | [N3. Test builds for every PR, nightlies](#n3-test-builds-for-every-pull-request) | CI |
| | [N4. Switch registry](#n4-switch-registry) | Docs, tooling |
| **Next** | [X1. Shader translation off the GPU thread](#x1-shader-translation-off-the-gpu-thread) | Shaders, stutter |
| | [X2. Persisted SPIR-V](#x2-persisted-spir-v) | Boot, shaders |
| | [X3. Non-blocking boot and memory-mapped caches](#x3-non-blocking-boot-and-memory-mapped-caches) | Boot |
| | [X4. Cheaper Intel pipeline compiles](#x4-cheaper-intel-pipeline-compiles) | Shaders, Intel |
| | [X5. Pipeline-library fast-link](#x5-graphics-pipeline-library-fast-link) | Shaders, stutter |
| | [X6. Shader warm-up](#x6-shader-warm-up) | Shaders, stutter |
| | [X7. AMD Radeon iGPU path](#x7-amd-radeon-igpu-path) | Devices, AMD |
| **Later** | [L1. LSE atomics in the JIT](#l1-lse-atomics-in-the-jit) | CPU |
| | [L2. Handheld mode and hotkeys](#l2-handheld-mode-and-hotkeys) | Frontend |
| | [L3. Portable mode out of the box](#l3-portable-mode-out-of-the-box) | Frontend, packaging |
| | [L4. Auto-update](#l4-auto-update) | Frontend, packaging |

## Now

### N1. Vblank-locked presenter for generated frames

`area: present` · `switch: EDEN_NX2_FLUID=1 (experimental, in progress)`

**Problem.** With frame generation on, v20 reads 26.7-28 game fps in Ocarina of Time, so a real
frame arrives every 36-37 ms: more than two 16.67 ms refreshes and fewer than three. Shown frames
cannot all stay up for the same time, and the result is visible micro-stutter.

**Plan.** A presenter locked to the display's vblank, so every shown frame, real or predicted, stays
up for exactly one 16.67 ms refresh at 60 Hz. It ships first as an experiment behind
`EDEN_NX2_FLUID=1`.

**Done when.**
- A present-level capture in the Hyrule Field scene with frame generation on shows every shown frame
  lasting one refresh at 60 Hz.
- Game fps in the same scene is not lower than v20 (valid [bench rows](docs/BENCHMARKING.md)).
- It is on by default and the switch is gone, once rows from the Arc 140T and at least one other
  device agree.

**Help wanted.** Shown-frame captures on 60 Hz and high-refresh displays.

### N2. Intel driver crash when the pipeline cache is saved during builds

`area: shaders` · `vendor: Intel` · driver report:
[IGCIT #1614](https://github.com/IGCIT/Intel-GPU-Community-Issue-Tracker-IGCIT/issues/1614)

**Problem.** Intel's Vulkan driver can crash when the pipeline cache is saved while other threads
are still building pipelines into the same cache.

**Plan.** Never save the cache while a build into it is in flight, without making the game wait for
the save. Fix in progress.

**Done when.**
- A stress run that saves the cache during heavy shader building no longer crashes on Arc.
- Bench rows show no new frame-time spikes, and the cache still loads on the next boot.

### N3. Test builds for every pull request

`area: ci`

**Problem.** Testers and agents without a local toolchain cannot try a change until it is merged.

**Plan.** `build-check.yml` already builds every pull request. Attach the build as a downloadable
artifact and publish nightly builds from the default branch.

**Done when.** Every PR links a build that runs on a clean Windows machine, and a nightly build
appears on the [releases page](https://github.com/karimalsalah/guarana/releases).

### N4. Switch registry

`area: docs` · `area: tooling`

**Problem.** `EDEN_NX2_*` switches are read in many places across `src/`. Players and agents cannot
tell which ones are measured, which are experiments, and which are diagnostics.

**Plan.** One table in [docs/switches.md](docs/switches.md): name, default, effect, scope, status and
the evidence behind it.

**Done when.** Every switch read in `src/` is listed, and CI flags a new switch that is missing from
the table.

## Next

### X1. Shader translation off the GPU thread

`area: shaders` · upstream: [Lichii #1378](https://github.com/zortos293/Lichii/issues/1378),
[Lichii #1379](https://github.com/zortos293/Lichii/issues/1379)

**Problem.** On a pipeline cache miss, the GPU back thread still reads the shader environments and
translates the programs to IR itself before the draw can go on. SPIR-V emission and the driver
compile already run on worker threads
([docs/nx2/async-native-pipelines.md](docs/nx2/async-native-pipelines.md)).

**Plan.** Move the remaining translation off the GPU thread without discarding draws and without
reading guest state unsafely. Port or help land the upstream work rather than writing a second
version.

**Done when.** With a fresh cache on the same route, `EDEN_NX2_STUTTER_TRACE=1` logs fewer and
shorter hitches whose top cause is a pipeline built on the GPU thread, and the image is unchanged.

### X2. Persisted SPIR-V

`area: boot` · `area: shaders` · upstream: [Lichii #1366](https://github.com/zortos293/Lichii/issues/1366)

**Problem.** Translated SPIR-V is not kept across restarts, so every pipeline built after a restart,
cached or not, has to translate its shaders again first.

**Plan.** Persist the translated SPIR-V next to the pipeline cache, keyed so that a stale entry is
rebuilt, and load it instead of translating again.

**Done when.** The boot phases in `Nx2Boot` lines (`EDEN_NX2_STUTTER_TRACE=1`) show less time
loading shaders on a warm cache, on the same machine, with identical pipelines.

### X3. Non-blocking boot and memory-mapped caches

`area: boot`

**Problem.** The game's first frame waits for boot work, loading the disk caches among it, that
could overlap with the game starting.

**Plan.** Let the game start while that work finishes in the background, and memory-map cache files
instead of reading and copying them.

**Done when.** Time from launch to the first game frame drops on a warm cache, measured with
`Nx2Boot` lines on the same machine, with no new hitches after boot.

### X4. Cheaper Intel pipeline compiles

`area: shaders` · `vendor: Intel`

**Problem.** Intel's driver takes 0.4-1 s to compile one pipeline, which is where first-visit
hitches come from.

**Plan.** Measure what drives the compile time (`EDEN_NX2_PIPELINE_BUILD_TRACE` for build times,
`EDEN_NX2_PIPELINE_STATS` for what the driver's compiler made), then change what we hand the driver
where its output stays the same.

**Done when.** The median build time per pipeline on Arc drops for the same shader set, with
unchanged screenshots and frame captures.

### X5. Graphics pipeline library fast-link

`area: shaders` · upstream: [Lichii #1173](https://github.com/zortos293/Lichii/issues/1173),
[Lichii #1177](https://github.com/zortos293/Lichii/issues/1177)

**Problem.** The first draw that needs a new pipeline waits for its full compile.

**Plan.** Where the driver supports `VK_EXT_graphics_pipeline_library`, link pre-built stages
quickly for the first draw and switch to the fully optimised pipeline once it is ready.

**Done when.** First-visit hitch time drops on a supporting driver, the image is unchanged, and
drivers without the extension keep today's path.

### X6. Shader warm-up

`area: shaders` · upstream: [Lichii #1183](https://github.com/zortos293/Lichii/issues/1183)

**Problem.** Most pipelines are built the moment they are first drawn.

**Plan.** Build pipelines that are known to be needed in the background before gameplay reaches
them, building on upstream's `shader_warm` precaching.

**Done when.** A scripted route on a fresh cache shows fewer first-visit hitches in the stutter
trace, without a longer boot.

### X7. AMD Radeon iGPU path

`area: devices` · `vendor: AMD`

**Problem.** Radeon 780M, 880M, 890M and 8060S devices are untested here, and AMD drivers differ
from NVIDIA's in ways upstream has documented ([docs/nx2/amd.md](docs/nx2/amd.md)). The only number
so far is a third-party report of about 30 fps in Ocarina of Time on an 890M
([Lichii #1433](https://github.com/zortos293/Lichii/issues/1433)).

**Plan.** Collect bench rows from real AMD handhelds, check every Intel-tuned switch
(`EDEN_NX2_GM=11`, `EDEN_NX2_POWER_POLICY=igpu`) on them, and fix AMD-only problems upstream first.

**Done when.** Valid rows from at least two AMD iGPU devices, a recommended AMD config in
[docs/DEVICES.md](docs/DEVICES.md), and its known issues listed there.

## Later

### L1. LSE atomics in the JIT

`area: cpu` · `component: dynarmic`

**Problem.** Games use ARMv8.1 LSE atomic instructions, and how the JIT translates them shows up in
guest CPU time.

**Plan.** Emit host atomic instructions for the guest's LSE atomics in dynarmic's x86-64 backend
wherever the semantics match exactly.

**Done when.** Dynarmic's tests pass and `EDEN_NX2_GUEST_THREAD_STATS=1` shows less JIT time in a
scene that leans on atomics, with game fps not lower.

### L2. Handheld mode and hotkeys

`area: frontend`

**Problem.** Windows handhelds are played with a controller; the desktop interface expects a mouse
and keyboard.

**Plan.** A controller-first mode: the game list and the common settings navigable with a pad, and
hotkeys for frame generation, a frame-time overlay, pause and quit.

**Done when.** A handheld player can set up and play without a keyboard, mouse or touch screen.

### L3. Portable mode out of the box

`area: frontend` · `area: packaging`

**Problem.** A `user` folder next to the program already makes an install portable, but
`nx2_switches.txt` is read only from the program's folder and from `%APPDATA%\lichii`, so a portable
install cannot keep its own switches.

**Plan.** Read `nx2_switches.txt` from the portable `user` folder too, and document portable use as
a first-class way to install on handhelds and USB drives.

**Done when.** A fresh portable install keeps settings, saves, caches and switches in its own folder
and writes nothing to `%APPDATA%`.

### L4. Auto-update

`area: frontend` · `area: packaging`

**Problem.** Updating means downloading and extracting a zip by hand.

**Plan.** Start from the update checker and installer already in `src/frontend_common` (off in
release builds): an opt-in check against this repository's releases that downloads, verifies and
installs the new build in place, keeping settings and caches.

**Done when.** An older build updates itself to the latest release after the player agrees, and the
check can be turned off.

## Shipped

See [CHANGELOG.md](CHANGELOG.md).
