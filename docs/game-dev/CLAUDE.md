# Writing browser games that are fast, first time, every time

Copy this file into the root of a new game project (as `CLAUDE.md`). It is the distilled
experience of building Ouro (a slither.io-style game: C→WebAssembly simulation, WebGPU + WebGL
renderers, a Rust game server), including the mistakes, so the next game starts where this
one ended instead of rediscovering it.

## 0. The rules that matter most
1. **Measure on the real device, before and after every change.** Build the performance panel
   (section 2) on day one. A change that was not measured on the target hardware is a guess.
2. **The test machine lies about graphics.** Headless browsers draw in software: they have no
   fast clear, punish things real GPUs don't care about, and hide what real GPUs do care about.
   Use them for correctness and for *ratios between passes*, never for "this will save X ms".
   (Ouro: the floor got 10× faster in software and exactly 0% faster on the owner's laptop.)
3. **Pixels written are the GPU cost on laptops and phones**, not the maths per pixel. Every
   full-screen pass, overlay, vignette or blur costs ~1–1.5 ms on an integrated GPU. Draw as
   few pixels as possible; let the screen clear provide the background.
4. **Say what you don't know.** Report real numbers, name the ones that are estimates, and
   undo changes that didn't help. A wrong "75% faster" costs more trust than the change gains.
5. **Decide the fixed things first**: world size, entity counts, target devices, frame budget
   (e.g. "GPU ≤ 2 ms, CPU ≤ 1 ms per frame on an Intel iGPU laptop, ≤ 20 MB"). Changing world
   size later broke integer types, grids and spawn rules in two codebases.

## 1. Before writing code: a one-page plan
- Target devices and browsers (and their worst one: usually a phone or an Intel iGPU on Edge).
- Frame budget (CPU ms, GPU ms) and memory budget (MB). Write them in this file.
- What runs where: simulation (WASM/JS/server), rendering (WebGPU with a WebGL 2 fallback),
  UI (plain DOM). Online? Then the server owns the truth from day one (section 5).
- Sizes: world radius, max entities, max food/particles. Pick integer types that fit 2× that
  (Ouro's 16-bit positions and `short` grid cells both overflowed when the world doubled).
- The test plan: which automated checks prove each part works (section 7).

## 2. Build the measuring tools first (day one)
A key (e.g. **P**) toggles a panel that shows, refreshed twice a second:
- fps; CPU ms per frame split by phase (simulation, prepare, draw submission, DOM);
- GPU ms per pass (WebGPU timestamp queries; WebGL `EXT_disjoint_timer_query_webgl2`). Time
  only **1 frame in 8**: splitting a frame into timed passes costs time itself. In WebGL, give
  uploads and the clear their own timers, or they hide inside the first pass;
- RAM: JS heap (`performance.memory`, rounded in Chrome/Edge: mark it "≈"), the game's own
  data (WASM memory size), graphics memory the game allocated (count it when you create
  buffers/textures), screen images (canvas w × h × 4 × ~3, an estimate);
- counts: entities drawn, food in view, network ping/bandwidth if online;
- a one-shot simulation benchmark (µs per step, timed over 100+ steps in one go).
Ask the owner/players for screenshots of this panel; it is the only real measurement.

## 3. Simulation
- Fixed timestep (e.g. 60 Hz), decoupled from frame rate; draw with interpolation (`alpha`).
- C (or Rust) compiled to WebAssembly with **static memory** (no growth, no malloc in the hot
  path): JS keeps zero-copy typed-array views into it forever.
- Spatial hash grids for collisions and food; cell indices are `i32`, never `short`.
- Level of detail: entities far from the player update less often (every 4th step).
- After any size/limit change: run the simulation natively under ASan/UBSan for minutes of
  game time (gcc `-fsanitize=address,undefined`).
- If the game exists both offline and online, the rules live in **one** file, or in two copies
  that a script diffs function by function after every change.

## 4. Rendering (WebGPU first, WebGL 2 fallback, one shared design)
- **One upload per frame**: the simulation writes all per-frame data (uniforms, instance
  data) into one block of WASM memory; the renderer copies that block once. No per-frame JS
  allocations (reuse arrays and descriptors).
- **Let the GPU build geometry** ("vertex pulling"): upload compact data (positions as 16-bit
  integers), compute quads/ribbons in the vertex shader. Snakes: one header per snake, trail
  points in a texture/storage buffer, only new points uploaded.
- **Background = the clear colour.** Decorations (grid dots, edges) are tiny flat-colour shapes
  computed in the vertex shader; no pixel does maths or blending for the floor.
- Discard pixels that add nothing (outside a sprite's circle) instead of blending zeros.
- Never darken/dim with a full-screen overlay (CSS gradients over the canvas count!): draw the
  background darker instead (free).
- No automatic resolution scaling unless the owner wants it; offer a Quality setting.
- Half precision (f16) in WebGPU shaders where supported: a real win on mobile GPUs.
- Keep both renderers in step: every visual feature in both, checked by screenshots of both.
- Handle device loss (phones drop the GPU when the tab is hidden): recreate and re-upload.

## 5. Online games
- Server-authoritative simulation; the page **predicts its own player** with exactly the same
  movement maths (fixed steps, same turn rates); any mismatch shows as sideways pulls.
- Stamp each input with the step it was made for; the server applies it there (rewind and
  replay a few steps) instead of when it arrived. Players then turn where they turned.
- Other players: interpolate behind a **jitter buffer sized by a percentile** (e.g. 95% of the
  last 10 s, capped), not by the worst stall; carry entities on through stalls and blend errors
  out over ~80 ms. Send headings/targets so the page can predict, not guess.
- Snapshots: deltas (only what changed, small integers), full state only when needed.
- Delay compensation in hit tests for players; never for bots ramming players.
- Server: single-threaded loop is fine at this scale (threads measured slower); keep socket
  read buffers small (a 128 KB zero-filled buffer cost 70% of the CPU).
- Deploy pipeline from the start: CI deploys on push using repository **secrets**; never put
  passwords, keys or server addresses in the repo. Expose `/status` with the running version so
  you can see what is live. Measure what players actually run, not what is in the repo.

## 6. Code hygiene that prevents real bugs
- Never put a `//` comment in the middle of a line in JS or shader strings (it silently
  comments out the rest: happened twice). Run `node --check` on every JS file after editing.
- When the same lines exist twice (two copies of a function), check where a search-and-replace
  landed.
- UI changes go in every page variant (offline and online), rule changes in every simulation copy.
- Experiments go behind a switch (URL parameter or a settings button, not a link with `&`
  that may get mangled), measured, then either made the default or deleted.
- Keep a project notes file (this file's sibling) with: measured numbers, what was tried and
  did not help, and every pitfall hit. Update it in the same commit as the change.

## 7. Verify automatically, every time
- Browser tests (Playwright): load each page in each renderer, play, die, restart; fail on any
  page error.
- Pixel probes: after drawing, read the canvas at the spot where the simulation says an object
  is (proves positions, signs and decoding in all four quadrants of the map).
- Brute-force coverage checks for culling/grid maths (random cameras vs a list of what should
  be visible) before trimming any safety margin.
- Native sanitizer runs of the simulation; for servers, a load test with many fake clients.
- Compare old vs new build side by side (same scenario, several runs, report the median).

## 8. Order of work for a new game
1. Plan (section 1) and write budgets here.
2. Perf panel + test harness + deploy pipeline (sections 2, 5, 7).
3. Simulation with fixed sizes, sanitizer-clean.
4. One renderer end to end (WebGPU), then the fallback, both from the same design.
5. Online play (if any): authority, prediction, jitter buffer.
6. Only then features and polish, each measured on the real device before it ships.
