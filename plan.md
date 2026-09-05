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

- 2026-09-04: First real data. Pulled a small sample from
  `dials.lbl.gov:/net/dials/raid1/yangha/Tutorial/agent_test` (a 1200-image,
  full-scale insulin rotation dataset — its `indexed.refl`/`integrated.refl`
  are 86–122MB, well beyond our Phase 1 "start small" target) and used
  `dials.slice_sequence` there to cut a 30-image/1316-reflection subset,
  saved locally as `sample-data/indexed_1_30.expt`/`.refl` (kept in-repo,
  ~2MB, so re-testing doesn't need SSH access again). Loading it surfaced
  drag-and-drop was completely broken — three separate upstream API drifts
  between `ReciprocalLatticeViewer.js` and its unpinned
  `dials_javascript_parser` dependency, all pre-existing and unrelated to
  the VR work, just never exercised before because drag-and-drop was
  disabled on the one page wired into the build (see the Phase 1 entry
  above). Fixed: a renamed parser method, a reflection-position-building
  path built against a data shape the parser no longer produces (routed
  through the same, already-correct code path the live-streaming feature
  uses instead of patching the stale one), and a missing `this.crystals`
  populate step in the parser's file-based experiment loader (compensated
  for in our own calling code rather than patching the vendored dependency).
  Also fixed a related non-blocking display bug found by the same audit
  (`displayNumberOfReflections` read a property that doesn't exist).
  Verified via a simulated drag-and-drop in headless Chromium: zero errors,
  and the loaded reflection count (1316) matches `flex.reflection_table`'s
  own count for the same file exactly.
  - **Known, not fixed**: `addCalculatedIntegratedReflectionsFromData` (part
    of the not-yet-built live "calculated reflections" streaming feature)
    has the same stale-data-shape assumption as the bug above, plus
    references to `this.expt.scan`/a bare `goniometer` that don't resolve
    on `ExptParser`. It has no current caller (dead code), so left as-is —
    needs the same "route through the modern flat-getter API" treatment
    when Phase 3 (live streaming) actually gets built.
  - Given how much silent API drift turned up from one previously-untested
    code path, worth treating any *other* not-yet-exercised feature (mesh
    generation, crystal view, calculated/integrated overlays, multi-experiment
    display) as unverified until actually tried, not just "should work because
    the code is there."

- 2026-09-04: Swapped the loading mechanism for the live deployment, for
  testing convenience. HTML5 drag-and-drop is a desktop mouse paradigm with
  unknown support in the Quest Browser and fiddly to test either way, so
  rather than debug that on-device right now, the deployed page **auto-loads
  `sample-data/indexed_1_30.expt`/`.refl` on open**, through the same
  `addExperiment`/`addReflectionTable` calls drag-and-drop already used (no
  new data-handling logic — copied to `resources/sample-data/` so Vite's
  `publicDir` bundles it into the deployed build). This is explicitly
  **temporary test scaffolding**, flagged as such at the call site, not the
  real load-data feature — **the live URL no longer starts on an empty
  "drag files here" scene for anyone who visits it**; it always shows this
  one sample dataset until the scaffolding is replaced. When a real
  loading UI gets built (a file picker button is the likely answer, since
  it works via the Android file picker on-device, unlike drag-and-drop),
  this block should be removed outright, not left alongside it.

- 2026-09-04: First look with real data on-device: "a lot of surfaces"
  instead of a point cloud. Root cause was `THREE.PointsMaterial.size` --
  a raw shader uniform, not a transform, so it doesn't shrink with
  `VR_WORLD_SCALE` (which works by scaling an ancestor group) the way point
  *positions* do, and it's only distance-attenuated under a perspective
  projection (three.js's shader skips that for the desktop
  OrthographicCamera entirely) — so the existing size, tuned for a camera
  where it has no distance dependence, blew up into screen-filling quads
  once real perspective distance attenuation applied at VR's much closer
  viewing range. Fixed by swapping `material.size` to a separate
  `VR_POINT_SIZE` constant (0.03, ~3cm) on VR session start and restoring
  it on exit, mirroring the content-group scale lifecycle. Verified the
  enter/exit lifecycle directly (dispatching sessionstart/sessionend on
  renderer.xr — doesn't need real hardware since the handlers only touch
  the THREE.js scene graph): sizes correctly flip 10 → 0.03 → 10, zero
  errors. **`VR_POINT_SIZE` itself is still an untested guess** — mechanism
  is now correct, but whether 3cm is actually the right size wants an
  on-device look.

## Needs on-device verification (Quest 3) — not yet checked

- ~~Does "Enter VR" actually appear and start a session at all~~ — confirmed
  working (empty scene, visible controller laser).
- ~~Does the auto-loaded reflection data render as points, not giant
  quads~~ — was broken (see above), now fixed pending a re-check on-device.
- Whether `VR_POINT_SIZE` (0.03) is actually a good point size once seen
  in the headset, or needs tuning up/down.
- 2026-09-04: reported miller-index hover label "not very consistent."
  Found the actual bug: `updateVRHover` only ever read `vrControllers[0]`
  (one arbitrary hand, per whatever order WebXR enumerates input sources —
  not necessarily consistent or tied to a specific hand), while
  grab-rotate already checks both controllers. Pointing with the "other"
  hand, or switching hands, would look exactly like a flaky feature with
  no way to tell from inside the headset. Fixed to loop over both
  controllers, same pattern as grab-rotate. **Still needs an on-device
  recheck** — could only verify the animate loop runs without erroring,
  not that hover now actually works reliably (headless Chromium has no
  real XR input sources to test the hit-detection itself against).
  Also worth watching for: whether the reflection points (now ~3cm) are
  just inherently fiddly to aim a laser at precisely, separate from the
  which-hand bug — if hovering still feels unreliable after this fix,
  that's the next thing to check (may want a larger raycast tolerance
  specifically for VR).
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

## Deployment

- **Repo**: [github.com/yangha7/VR_DIALS](https://github.com/yangha7/VR_DIALS)
  (`main` branch has the full history above; `gh-pages` branch is the built
  output, deployed via the inherited `npm run deploy` → `gh-pages -d dist`).
- **Live URL**: https://yangha7.github.io/VR_DIALS/ReciprocalLatticeViewerHeadless.html
  (same pattern as the existing `X-ray_Diffraction_Simulator` demo — served
  from the built filename, not a bare `index.html` at the repo root). GitHub
  Pages needed enabling once manually in repo Settings → Pages (source:
  `gh-pages` branch, root) — pushing to that branch alone didn't turn it on
  for a brand-new repo.
- Verified 2026-09-04 via headless Chromium against the live URL: HTTPS/secure
  context confirmed (required for WebXR to activate at all), all assets
  resolve, zero console/page errors, XR scaffolding initializes. Still
  untested on an actual Quest 3 — see the on-device checklist above.
- To redeploy after further changes: `npm run build && npm run deploy` from
  this directory, then re-check the live URL (Pages typically updates within
  well under a minute of the `gh-pages` branch changing).

## Access needed from user (open)

- **GitHub**: no access needed to reach this point. Will need a repo to push to
  (yours, or a fork under your account) once ready to publish/deploy — not urgent yet.
- **`dials.lbl.gov` SSH**: not needed for Phase 0/1 with synthetic or small sample
  `.expt`/`.refl` files. Will matter once Phase 1 needs a first real dataset to
  drag-and-drop test against, and is required for Phase 2 (real fine-sliced data)
  and Phase 3 (live DIALS session). Send over a small sample pair, or access
  whenever convenient — not blocking right now.
