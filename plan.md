# VR DIALS Reciprocal Lattice Viewer — Project Plan

## Goal

Rewrite `dials.reciprocal_lattice_viewer` (DIALS's wxPython/PyOpenGL reciprocal-space
viewer, laggy when rotating large reflection sets) as a WebXR app for standalone
Meta Quest 3, building on prior work: [X-ray Diffraction Simulator / Crystal Lab VR](
https://yangha7.github.io/X-ray_Diffraction_Simulator/vr_viewer.html), an idealized
textbook-equation reciprocal space demo already proven on Quest 3.

The new viewer visualizes *real* experimental DIALS output (strong spots, indexed/
integrated reflections, actual detector/goniometer geometry) rather than synthetic
textbook data.

## Why the DIALS viewer lags (and why that's fixable)

`dials.reciprocal_lattice_viewer` (`dials/src/dials/command_line/reciprocal_lattice_viewer.py`)
uses wxPython + PyOpenGL: old-style immediate-mode-ish rendering, per-frame Python-side
work, no modern GPU buffer geometry. This is a solved problem in WebGL terms — the
existing Crystal Lab VR demo already renders thousands of GPU-driven point sprites
(`THREE.Points` + custom vertex/fragment shader) at VR-friendly framerates.

## Key existing assets (don't rebuild these)

1. **Crystal Lab VR demo** (`vr_viewer.html`) — proven Quest 3 WebXR shell:
   - WebXR session setup, "Enter VR" button, controller models, laser-pointer raycasting
   - Trigger-drag sliders, drag-to-reorient with momentum
   - GPU shader-based point sprites for reciprocal lattice points
   - Canvas-texture panels for HUD/text (DOM overlays don't render in WebXR)
   - Instanced meshes for atoms, Ewald sphere wireframe, detector panel w/ canvas texture

2. **[toastisme/dials_browser_rlv](https://github.com/toastisme/dials_browser_rlv)** —
   existing browser port of the real DIALS viewer (desktop only, no VR):
   - Vite + Three.js r172 project
   - `.expt` files load directly as JSON
   - `.refl` files (DIALS `flex.reflection_table`) exported via `.as_msgpack()`,
     decoded client-side with `@ygoe/msgpack` + `dials_javascript_parser`
     (understands DIALS column layout: Miller indices, xyz obs/calc, panel, etc.)
   - Features: reciprocal cell display, reflection centroids, hover-for-hkl,
     calculated-vs-observed comparison, multi-experiment support
   - Also ships `dials_websockets_server_example.py` / `dials_http_server_example.py` —
     a live bridge that pushes `update_experiment`/`update_reflection_table` messages
     from a running DIALS/Python session to the browser (streaming-during-collection,
     not needed for v1 but architected for)

## Decisions locked in (2026-09-04)

| Question | Decision |
|---|---|
| Render target | **Standalone on Quest 3** (on-device mobile GPU, Snapdragon XR2 Gen 2) — no PC tether/Link streaming assumed |
| Data loading for v1 | **Static file load** — drag-and-drop a finished `.expt`/`.refl` pair, like `dials_browser_rlv` today. Live websockets streaming deferred to a later phase, not designed out |
| Base codebase | **Fork `dials_browser_rlv`** and graft WebXR onto its existing Three.js scene, inheriting its working `.refl`/`.expt` parsing instead of rewriting a DIALS data loader |
| Target dataset scale for v1 | **Start small** — a modest still/narrow-wedge dataset (~10³–10⁴ strong spots). Large fine-sliced dataset performance (~10⁵–10⁶ spots) is an explicit later phase, not a v1 design constraint |

## Central open risk

GPU point budget on Quest 3's mobile chipset for standalone rendering. Desktop
PyOpenGL being laggy at a given spot count says nothing reassuring about a mobile
GPU at the same count. This is a **benchmark**, not an assumption — deferred to
Phase 2 once the small-dataset MVP works, rather than speculatively over-designed
(e.g. premature LOD/binning) in Phase 1.

## Phases

### Phase 0 — Scaffolding
- Fork `dials_browser_rlv`, get its existing Vite dev build running locally
- Confirm drag-and-drop `.expt`/`.refl` loading works unmodified (validates the
  inherited data pipeline before any VR work starts)

### Phase 1 — WebXR MVP (small dataset, standalone)
- Add a WebXR session to the existing Three.js scene (`renderer.xr.enabled`,
  "Enter VR" button, controller models + laser-pointer raycasting — ported from
  `vr_viewer.html`)
- Test against a modest dataset (~10³–10⁴ spots) so performance isn't yet a variable
- Core v1 feature set (mapped from the real viewer's most-used options):
  - Reciprocal lattice points as GPU point sprites (reuse existing point shader;
    check whether the desktop viewer's point rendering can be reused directly or
    needs adapting)
  - Color/filter: indexed vs. unindexed vs. integrated (`display` option)
  - `d_min` resolution cutoff as a VR-manipulable control (thumbstick or
    trigger-drag slider)
  - Toggles: reciprocal cell edges, beam vector, rotation axis
  - Controller-raycast hover/select → canvas-texture panel showing hkl + resolution
    (VR analog of the desktop hover-for-hkl feature)
  - Grab-to-rotate / two-handed scale of the lattice (real hand manipulation —
    a place VR is strictly better than mouse interaction, not just parity)

### Phase 2 — Scale it up, deliberately
- Bring in a real fine-sliced dataset, benchmark on-device
- Whatever breaks first (framerate / load time / memory) determines the fix:
  image-range binning (draw only the currently-relevant `z`/image slice, like a
  time-scrubber), resolution-shell LOD, or batching efficiency improvements
- Avoid designing LOD speculatively before the actual bottleneck is known

### Phase 3 — Live streaming (deferred, not discarded)
- Wire in the websockets bridge (`dials_websockets_server_example.py`) to watch
  spots populate during an actual data collection
- Phase 1–2 should keep reflection-data ingestion decoupled from its source
  (drag-and-drop vs. socket push) so this doesn't require a rewrite

### Phase 4 — VR-native stretch goals
Things structurally hard on a 2D desktop viewer but natural in a headset:
- Side-by-side comparison of multiple experiments in physical space
- Combined reciprocal-space + real-space/detector view simultaneously
  (per the Crystal Lab VR demo's front/behind panel layout)
- Ewald-sphere overlay against observed spots for visual indexing diagnostics

## Access / resources needed from user

- **GitHub**: none needed yet to *start* — forking `dials_browser_rlv` and working
  locally doesn't require access to the user's account. Will need push access to
  the user's own GitHub (or a repo to push to) once ready to publish/deploy
  (e.g. via GitHub Pages, matching how the Crystal Lab VR demo is hosted).
- **`dials.lbl.gov` SSH access**: needed once Phase 0/1 needs a real `.expt`/`.refl`
  pair to test against, and definitely for Phase 2 (real fine-sliced dataset) and
  Phase 3 (live DIALS session to stream from). Not required to begin Phase 0
  scaffolding with the fork itself.

## Status

- 2026-09-04: Plan written. `dials_browser_rlv` (MIT) imported as the local git
  history's base commit; `npm install` and `npm run dev`/`npm run build` verified
  working. Fixed two pre-existing HTML bugs (stray commas in attribute lists) that
  broke under the current Vite/parse5 toolchain, blocking both dev and build.
  `ReciprocalLatticeViewerHeadless.html` is the entry point actually built/deployed
  per `vite.config.js` (`main`); `ReciprocalLatticeViewer.html` and
  `ReciprocalLatticeViewerHeadlessHttp.html` are alternate entry points not wired
  into the build yet — worth clarifying their intended roles before Phase 1 work
  picks one to extend with WebXR.
- Repo is a fresh local git repo (not yet pushed anywhere), remote `upstream` set
  to `https://github.com/toastisme/dials_browser_rlv.git` for reference/pulling
  future upstream fixes. Local git identity was set repo-scoped (not global) as
  `yangha@lbl.gov` / "Yang Ha" since none existed on this machine — change with
  `git config user.name`/`user.email` (no `--global`) if that's not right.

- 2026-09-04: Phase 1 implemented on `ReciprocalLatticeViewerHeadless.html` /
  `src/js/ReciprocalLatticeViewer.js`. Before extending it, checked how
  `isStandalone` gates drag-and-drop (`window.addEventListener('drop', ...)`
  only calls `addReflectionTable`/`addExperiment` `if (window.viewer.isStandalone)`)
  — Headless.html passed `false`, meaning drag-and-drop was actually **disabled**
  on the one file wired into the build, contrary to the README's headline
  feature. Flipped it to `true` and made the automatic `ws://127.0.0.1:50010/`
  connection opt-in via `?live=1` (was unconditional on load) so a standalone
  headset doesn't spend its session retrying a server that will never exist.
  This keeps the ingestion path (drag-and-drop vs. socket push) decoupled for
  Phase 3, per the Phase 3 note above.
  - Added an "Enter VR" (WebXR) path alongside the existing desktop view.
    The desktop `OrthographicCamera` + `OrbitControls` can't drive an immersive
    session (WebXR supplies its own per-eye perspective projection every frame;
    moving that camera by hand would fight headset tracking) — VR uses a
    separate stationary `PerspectiveCamera` rig, and the user grabs/turns the
    *content* instead (trigger-drag rotation, or thumbstick as a hands-free
    alternative) rather than moving the camera, which also avoids
    camera-motion sickness.
  - Content renders at **1/300 scale in VR** (`VR_WORLD_SCALE`): the desktop
    scene spans roughly a thousand units; at 1:1 metres that's vast next to
    the ~6cm eye separation used for stereo rendering, collapsing depth
    perception to nothing. `scene.add` is redirected to a scaled content
    group only for the VR session's duration (restored on exit) rather than
    rewriting the ~15 existing call sites that add directly to `window.scene`.
  - Desktop's on-demand render loop (`requestAnimationFrame` gated by a
    "did anything change" flag) doesn't work for VR, which needs a continuous
    per-frame loop regardless of scene changes (headset pose changes every
    frame) — VR uses `renderer.setAnimationLoop` on its own separate path;
    desktop's `animate()` now early-returns while `renderer.xr.isPresenting`.
  - Controller-ray hover shows hkl/resolution/expt id on an in-scene
    canvas-texture panel (desktop's hover info is a DOM overlay, which
    doesn't composite into an immersive session).
  - Verified via headless Chromium (Playwright, installed to the local
    playwright cache for this — not a project dependency): both `npm run dev`
    and `npm run build` are clean, page loads with zero console/page errors,
    `renderer.xr.enabled` is set, the rig + two XR controllers are created,
    and the websocket opt-in gate works both ways. `navigator.xr` isn't
    available in headless Chromium, so the actual VRButton-appears /
    on-headset-rendering path is **unverified** — needs a real Quest 3 check.

## Needs on-device verification (Quest 3) — not yet checked

- Does "Enter VR" actually appear and start a session at all.
- `VR_WORLD_SCALE` (1/300) and `VR_CONTENT_DISTANCE` (1.2m) defaults — first
  real look at whether the lattice lands at a comfortable size/distance.
- Controller-ray hover accuracy (hitting individual reflection points with
  the laser).
- Thumbstick zoom direction — the code comment flags it may be backwards
  (push-forward-to-zoom-in was a guess, not measured).
- Grab-rotate feel (single-controller wrist-orientation delta, no position
  component yet).
- Actual framerate/comfort with a real dataset loaded (still untested even
  at the "start small" ~10³–10⁴ spot scale from the target-dataset decision).

Next step: get a `.expt`/`.refl` pair (even a small/synthetic one) onto the
Quest 3's browser and actually try it, then tune the above against what's
actually seen on-device.

## Access needed from user (open)

- **GitHub**: no access needed to reach this point. Will need a repo to push to
  (yours, or a fork under your account) once ready to publish/deploy — not urgent yet.
- **`dials.lbl.gov` SSH**: not needed for Phase 0/1 with synthetic or small sample
  `.expt`/`.refl` files. Will matter once Phase 1 needs a first real dataset to
  drag-and-drop test against, and is required for Phase 2 (real fine-sliced data)
  and Phase 3 (live DIALS session). Send over a small sample pair, or access
  whenever convenient — not blocking right now.
