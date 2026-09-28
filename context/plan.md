# Alpine_Chase — Technical Plan (Iteration 1)

> Once approved, this is saved verbatim to `context/plan.md`. It is the source of truth for the stack and architecture. **Do not re-litigate these decisions inside individual issues.**

## Context

Alpine_Chase is a browser-based, single-player, free-roam **skiing** game on one Matterhorn-inspired mountain. It plays like *Steep*, dialled 10% towards realism, and is rendered in the painterly anime-cinematic style of the reference clip (`context/art-reference.md`, `context/reference/*.jpg`). The repo holds only workflow/spec docs so far. This plan defines the stack, architecture, data flow, module APIs, performance budget and milestones. Phase 4 turns it into atomic GitHub issues.

Priorities: **(1) art direction**, (2) ski feel, (3) 40 FPS on an M5 MacBook Air in the browser, (4) foundations for multiplayer/mobile/customisation/audio/snowboard, designed for but not built.

---

## 1. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | **TypeScript (strict)** | Type safety across sim/render boundary; best tooling for Claude-written code |
| Engine / renderer | **Three.js (pinned version) `WebGPURenderer` + TSL node materials**, auto-fallback to WebGL2 backend | Code-first (no editor), full shader control for the painterly look, huge ecosystem. WebGPU ships in Chrome and Safari 26. The same TSL shaders compile to WebGL2 as a fallback. |
| Build / dev server | **Vite** | Instant HMR, including hot-reloading JSON data (mountain edits and feel tuning while playing) |
| Physics | **Rapier 3D (`@dimforge/rapier3d-compat`, WASM)** for collisions, ragdoll and detached skis. The **ski controller is custom code** that samples the heightfield analytically. | A custom controller gives exact control of feel and the realism blend. Rapier handles the hard rigid-body parts. Rapier also runs in Node (tests + a future server) and has a deterministic build for multiplayer later. |
| UI | **Preact + plain CSS** DOM overlay | Tiny (~4 kB), component-based, easy to reflow for touch later |
| Data validation | **Zod** schemas for every JSON data file | Hand-edited level/feel data fails loudly with clear errors |
| Dev tuning panel | **Tweakpane v4** (dev builds only, dynamically imported) | Sliders, live graphs and monitors for telemetry |
| Unit/integration tests | **Vitest** (node env for `sim/`, happy-dom for UI) | Fast, Vite-native |
| E2E / UI | **Playwright** (CI smoke tests, Chromium, WebGL2 fallback) + **Puppeteer MCP** (Claude's visual checks per workflow) | |
| Lint / format / types | **ESLint (flat config, typescript-eslint) + Prettier + `tsc --noEmit`** | ESLint also **enforces the sim boundary** (`src/sim/**` may not import `three`, DOM or `src/render/**`) |
| CI | **GitHub Actions**, single job named **`ci`**: install → lint → typecheck → unit/integration → build → Playwright smoke | Required for branch protection |
| Hosting | **GitHub Pages** via Actions on merge to `main` (repo is public, free) | Zero cost. Move to Cloudflare Pages if headers/bandwidth ever need it. |
| Asset formats | glTF/GLB with **Meshopt** geometry compression, **KTX2** textures (via `gltf-transform` CLI in `tools/`) | Small downloads, GPU-friendly |
| Asset tooling | **ComfyUI** (user runs Claude-authored workflows for 2D concept art, textures, cloud/sky paintings). **Blender headless** (`blender --background --python`), driven only by Claude's scripts. **Mixamo** for rig + fall/get-up clips. | Fits the MacBook Air: no local image-to-3D |

**Not needed in iteration 1:** auth/login, email, database, object storage, cloud VMs, runtime AI models. Progress saves to `localStorage` behind a `SaveStore` interface.
**Later:** authoritative multiplayer server in Node reusing `src/sim` unchanged (WebSocket/WebTransport, e.g. Colyseus or geckos.io). Accounts, cloud saves and leaderboards via **Supabase** (auth + Postgres + storage). Asset CDN via Cloudflare R2. Mobile app via **Capacitor** wrap.

**Rejected:** Babylon.js is capable, but has a smaller ecosystem and fewer stylised/TSL post-processing examples. Godot 4 web export is limited to the Compatibility renderer, C# can't export to web, and it's editor-centric (weak for Claude Code). Unity/Unreal are editor-bound, and their web builds are heavy or unavailable.

---

## 2. Architecture

### Core principle: simulation ⟂ presentation
- **`src/sim/`** is pure TypeScript: no Three.js, no DOM, no `Date.now()`/`Math.random()` (seeded RNG in state). It runs a **fixed 60 Hz step**. Its state is plain serialisable data. It is unit-testable in Node and reusable on a future server. ESLint enforces this.
- **`src/render/`, `src/camera/`, `src/ui/`** read **immutable snapshots** of sim state (previous and current tick). They interpolate by the loop's `alpha` and **never write to sim state**. Cosmetic-only systems live here: spray, tracks, spring bones, camera.
- **Communication:** input → sim via `InputFrame` (plain data). Sim → everything else via the `SimEvent[]` emitted each step (the audio hook for later, UI popups, camera shakes, FX triggers).

### Directory layout
```
src/
  main.ts                    bootstrap: load data → build world → start loop
  app/        Game.ts        wires sim/render/ui/input; loop.ts (fixed-step accumulator + interpolation alpha)
  sim/                       ── PURE, no three/DOM ──
    world.ts                 WorldState, createWorld(), step()
    events.ts                SimEvent union
    rng.ts, math/            seeded RNG; vec3/quat helpers on plain {x,y,z}/{x,y,z,w}
    terrain/                 heightfield.ts (bilinear height/normal/slope sampling), edits.ts (brush ops), build.ts
    rider/                   skiController.ts, airController.ts, modes.ts (FSM), landing.ts, feel.ts (realism blend)
    physics/                 rapierWorld.ts (init, heightfield + feature colliders), ragdoll.ts, looseSkis.ts
    features/                featureShapes.ts (kicker/rail/box/halfpipe/cliff params → colliders + ride surfaces)
    tricks/                  detector.ts (air rotation/grab/rail/butter tracking), classify.ts
    scoring/                 scoring.ts (chains, multipliers), challenges.ts (zones, medals)
    env/                     timeOfDay.ts (non-linear 30-min cycle), weather.ts
    spawn.ts                 drop points (heli), respawn
  input/                     InputFrame, KeyboardSource, bindings (data), InputSource interface
  render/
    Renderer.ts              WebGPURenderer setup, quality tiers, dynamic resolution
    lighting.ts              lighting keyframes → sun/moon/ambient/fog/sky uniforms
    materials/               snow.ts, rock.ts, toonCharacter.ts, foliage.ts (TSL; shared toon-ramp + coloured-shadow nodes)
    post/                    pipeline.ts, kuwahara.ts, brushTexture.ts, grade.ts, speedLines.ts
    sky/                     sky.ts (gradient + sun/moon), cloudSea.ts, distantPeaks.ts
    terrain/                 terrainChunks.ts (chunked LOD mesh from heightfield)
    features/                procedural meshes for kickers/rails/boxes/halfpipe/crevasses; instanced rocks/trees
    rider/                   riderView.ts, poses.ts (data-driven pose blending), ik.ts (2-bone legs), springBones.ts, ragdollView.ts
    fx/                      spray.ts (instanced painterly clumps), tracks.ts (ribbon trails), snowfall.ts
    heli/                    heliView.ts
    interpolate.ts
  camera/                    cinematicCamera.ts (rule-based rig: chase / low-speed / air-wide / crash)
  ui/                        Preact: App.tsx, TrickPopup, ScoreChain, MapView, WorldMarkers, Settings, MedalToast
  save/                      SaveStore interface + LocalStorageSaveStore
  devtools/                  panel.ts (Tweakpane), telemetry.ts, pins.ts, perfHud.ts
  schemas/                   Zod schemas for all data files
  data/
    feel.json  tricks.json  scoring.json  lighting.json  bindings.json  quality.json
    rider/default.json                        outfit/colours/equipment (customisation-ready)
    equipment/skis.json                       (snowboard.json later; same schema)
    levels/matterhorn/
      level.json            bounds, scale, base heightmap ref, drop points, trails (splines)
      terrain-edits.json    ordered brush ops applied on top of the base DEM
      features.json         kickers, rails, rocks, trees, halfpipe, cliffs, crevasses, avalanche zone
      challenges.json       challenge zones + medal thresholds
public/assets/              GLB/KTX2, heightmap PNG16, brush textures, painted cloud/sky textures
tools/
  terrain/                  DEM fetch/crop/scale → PNG16 (documented, one-off)
  blender/                  headless Python scripts (cleanup, decimate, retarget, export)
  comfyui/                  workflow JSONs + prompt files for concept/texture generation
  assets/                   gltf-transform compression scripts
  vite-dev-save.ts          dev-only Vite middleware: POST /__dev/save writes tuned JSON back to src/data
tests/  unit/  integration/  e2e/
```

### Key design decisions
- **Terrain = base DEM + ordered edit ops, applied at load (not pre-baked).** The base is the Matterhorn from swisstopo swissALTI3D (open data, attribution required). It is cropped and **rescaled to a ~2 × 2 km playable area with ~600–700 m vertical** at 1 m grid (2049² PNG16). The iconic horn stays as a steep, non-skiable backdrop above the heli drop, and the skiable flank sits below it. Edits (`raise`, `lower`, `flatten`, `smooth`, `carveTrail(spline, width, depth)`, `cliff`, `windLip`, `noise`) apply in <300 ms on load. Vite HMR re-applies them when `terrain-edits.json` changes, so **the user plays, asks for a change, and Claude edits JSON that hot-reloads**. The sim and the renderer consume the same `Heightfield`. A procedural pyramid+ridge-noise fallback exists if the DEM work stalls.
- **Iterative mountain building with the user:** dev key **P** drops a numbered *pin* (position + heading) that is saved to `devtools/pins` and shown in the world. The user says "add a kicker at pin 4" or "soften the drop between pins 6 and 7", and Claude edits `features.json` / `terrain-edits.json`.
- **Ski model (custom, sidecut-based):** gravity projected onto the slope, snow friction (μ depends on edge/brake/skid), aero drag ½ρC<sub>d</sub>A·v² (tuck shrinks A). Carve radius = `sidecutRadius · cos(edgeAngle)`. The velocity heading turns toward the ski heading, limited by edge grip. Excess lateral demand becomes a **skid** (speed scrub + spray event). Jumps: crouch-charge → pop impulse. Kickers launch from the lip normal. Air is ballistic with the feel gravity multiplier, plus input-driven spin/flip angular acceleration.
- **Realism blend:** every feel parameter is `{arcade, real}`, resolved as `arcade + realism·(real − arcade)` with `realism = 0.10` and optional per-param overrides (§4 APIs). The dev panel exposes the global slider, per-param overrides and telemetry (km/h, air time, spin °/s, landing angle, G).
- **Rider modes (FSM):** `heli → airborne ↔ grounded ↔ rail`, `→ smallFall → recovering → grounded`, `→ ragdoll → recovering → grounded`. Crash severity comes from impact speed along the normal, landing angle error and obstacle hits, with thresholds from feel params.
- **Ragdoll:** ~11 Rapier capsule bodies with limited joints, plus 2 free ski bodies that get an impulse on detach. After settling (low velocity or ~2.5 s) the rider blends from the ragdoll pose into a Mixamo get-up clip, and the skis fly back to the feet (cosmetic lerp).
- **Rider animation (no ski mocap exists):** **data-driven key poses** (neutral, tuck, carve L/R, crouch, pop, 6 grabs, spread eagle…) stored as bone rotations in JSON. They are blended procedurally from sim state (edge angle, crouch, speed, grab), with **2-bone IK** keeping feet planted on the skis and **spring bones** for hair tufts and hood (the windswept look). A small dev pose editor (Tweakpane) lets poses be tuned live. Mixamo supplies stumble/fall/get-up clips.
- **Rider asset route:** **Primary:** ComfyUI turnaround of the reference character → hosted image-to-3D (Tripo/Meshy free tier) → Blender headless cleanup/decimate (≤25k tris) → Mixamo auto-rig → GLB. **Fallback:** CC0 rigged stylised base (Quaternius) with outfit meshes and materials restyled. The art spike uses a capsule/mannequin stand-in.
- **Tricks:** the detector integrates rotation about the rider's yaw (spin), lateral (flip) and off-axis while airborne. It records grab slots/durations, rail contact, butter (pressed nose/tail at low edge). On landing it classifies against `tricks.json` (rotation rounded to the nearest 180° within tolerance, switch detection from velocity vs ski heading), then scores via `scoring.json` (base + rotation + grab hold + air time) × landing quality × combo multiplier. A crash zeroes the chain.
- **Day/night:** `env.timeOfDay ∈ [0,1)` advances over 30 real minutes through a piecewise map giving **~20 min day / 3 dusk / 4 moonlit night / 3 dawn**. `lighting.json` keyframes (sky top/horizon, sun/moon colour and intensity, shadow tint, fog, exposure) are interpolated. The night floor stays readable (blue moonlight).
- **Weather:** `clear | snow | overcast` in sim env (toggle in Settings), with a ~5 s blend. It affects fog density, sky saturation, light intensity and a camera-local GPU snowfall volume.
- **Camera (render-side, automatic cinematic):** springs toward a target derived from state. Chase from behind/above at low speed; lower and closer to the skis as speed rises; wider and higher when airborne (framing the landing); a side angle for rails; pull-back for ragdoll. Terrain collision avoidance, FOV kick with speed, subtle shake on landings from events.

---

## 3. Rendering & art pipeline (the look)

1. **Materials (TSL):** shared nodes `toonRamp(NdotL)` (3–4 soft bands) + `colouredShadow(shadowFactor)` lerping to periwinkle `#7C83C9`-ish in shadow (never grey) + a warm/cool fresnel rim.
   - **Snow:** off-white warm lit tone, blue-violet shadow, faint sparkle, slope/curvature-driven broad "painted plane" variation.
   - **Rock:** faceted charcoal/slate via flat normals + triplanar painted texture. Blend with snow by slope + edit masks.
   - **Character:** same ramp plus flat albedo colours from `rider/default.json`.
2. **Lighting & shadows:** one directional sun/moon with **cascaded shadows** (tight high-res cascade around the rider for crisp blue cast shadows, as in ref-05), plus hemisphere sky fill.
3. **Sky:** gradient (cobalt zenith → cyan haze horizon) driven by lighting keyframes, sun disc + bloom, a **cloud sea** plane below the summit (animated noise, toon-shaded, painted edges), and a ring of **distant peaks** (low-poly imposters from DEM surroundings).
4. **Atmosphere:** height fog + distance fog, blue-tinted, strong aerial perspective.
5. **Post pipeline (order):** scene pass (MRT colour + depth) → **bloom** → **painterly: generalized Kuwahara (8-sector) at 0.5–0.75× res**, recombined with edge-aware detail at full res → **brush-stroke / paper texture overlay** (screen-space, subtle, world-anchored where possible to avoid a "shower-door" look) → **colour grade** (lift/gamma/gain, saturation, warm highlights / cool shadows) → **speed lines / radial blur** above a speed threshold → FXAA.
6. **FX:** instanced painterly powder clumps (sprite atlas painted in ComfyUI) emitted from ski edges on carve/skid/land events. Ribbon ski tracks (fading, capped length). Snowfall volume.
7. **Art spike first (Milestone 1):** a terrain patch + stand-in rider + sky + clouds + post, with a free-fly camera and **side-by-side comparison against `context/reference/` frames** (dev overlay showing a chosen ref image). **The user signs off before any gameplay work.**

---

## 4. Data flow

```
Keyboard ──► KeyboardSource ──► InputFrame (plain data, per tick)
                                      │
loop.ts (accumulator, 60 Hz fixed) ───┤
                                      ▼
            sim.step(state, {player0: InputFrame}, dt) ──► new WorldState + SimEvent[]
              ├─ env (time of day, weather)
              ├─ rider FSM → ski/air controller ↔ Heightfield queries
              ├─ Rapier step (feature/obstacle contacts, ragdoll, loose skis)
              ├─ trick detector → scoring → challenges/medals
              └─ events: takeoff, land{quality}, carve{intensity}, skid, crash{severity},
                         trickLanded, comboEnded, railEnter/Exit, challengeEnter/Exit, medalEarned, heliJump
                                      │
             ┌──────────── snapshots (prev, curr) + alpha ────────────┐
             ▼                        ▼                   ▼            ▼
      render (interpolate)     camera rig          UI (Preact)   SaveStore (medals/bests)
      rider pose/IK/springs    (reads snapshot     popups, map,  [audio hook: subscribe to
      FX from events           + events)           markers        events — next iteration]

Load-time build:
 level.json + base DEM PNG16 + terrain-edits.json ──► Heightfield (shared) ──► Rapier heightfield + render terrain chunks
 features.json ──► featureShapes (sim colliders + ride surfaces) ──► render meshes/instances
 challenges.json ──► challenge zones (sim) ──► map + world markers (UI/render)
 (dev) JSON edits ─Vite HMR─► rebuild affected parts without reload; Tweakpane ─POST /__dev/save─► src/data/*.json
```

---

## 5. Module APIs (key types)

```ts
// input
interface InputFrame {
  steer: number;        // -1..1  (A/D)  grounded: edge/turn; air: spin
  pitch: number;        // -1..1  (W/S)  grounded: tuck/lean back+brake; air: front/back flip
  charge: boolean;      // Space held → crouch; release → pop/jump (also: jump out of heli)
  grab: GrabId | null;  // U/I/O/J/K/L → mute|safety|japan|tail|tip|critical (bindings.json)
  butter: -1 | 0 | 1;   // Shift+W / Shift+S nose/tail press
}
interface InputSource { poll(tick: number): InputFrame; dispose(): void } // keyboard now; gamepad/touch later

// sim
type PlayerId = string;
type RiderMode = 'heli'|'airborne'|'grounded'|'rail'|'smallFall'|'ragdoll'|'recovering';
interface RiderState {
  id: PlayerId; mode: RiderMode; modeTime: number;
  pos: Vec3; vel: Vec3; orient: Quat; angVel: Vec3;
  skiHeading: number; edgeAngle: number; crouch: number; switch: boolean;
  grab: { id: GrabId; held: number } | null;
  air: { time: number; spin: number; flip: number; offAxis: number; grabs: GrabRecord[] } | null;
  rail: { featureId: string; t: number; style: 'fifty'|'slide' } | null;
  chain: { tricks: LandedTrick[]; multiplier: number; score: number };
  equipment: EquipmentId;             // 'skis' now; 'snowboard' later
  ragdoll?: RagdollHandle;             // opaque handle into physics world
}
interface WorldState {
  tick: number; time: number; rngSeed: number;
  env: { timeOfDay: number; weather: Weather; weatherBlend: number };
  players: Record<PlayerId, RiderState>;
  challenges: Record<ChallengeId, { active: boolean; bestScore: number; medal: Medal | null }>;
}
function createWorld(level: LevelData, cfg: SimConfig): Promise<World>;   // loads Rapier, builds heightfield/features
interface World { state: WorldState; step(inputs: Record<PlayerId, InputFrame>, dt: number): SimEvent[]; snapshot(): WorldState; }

type SimEvent =
  | { t: 'takeoff'; player: PlayerId; speed: number }
  | { t: 'land'; player: PlayerId; quality: 'clean'|'sketchy'|'crash'; impact: number }
  | { t: 'carve' | 'skid'; player: PlayerId; intensity: number; pos: Vec3 }
  | { t: 'crash'; player: PlayerId; severity: 'small'|'big' }
  | { t: 'trickLanded'; player: PlayerId; trick: LandedTrick }
  | { t: 'comboEnded'; player: PlayerId; score: number; bailed: boolean }
  | { t: 'railEnter' | 'railExit'; player: PlayerId; featureId: string }
  | { t: 'challengeEnter' | 'challengeExit'; player: PlayerId; challengeId: ChallengeId }
  | { t: 'medalEarned'; player: PlayerId; challengeId: ChallengeId; medal: Medal }
  | { t: 'heliJump'; player: PlayerId };

// feel (feel.json)
interface FeelParam { arcade: number; real: number; override?: number; unit?: string; note?: string }
interface FeelConfig { realism: number; params: Record<FeelKey, FeelParam> }
type FeelKey = 'airGravityMul'|'spinRate'|'flipRate'|'airControl'|'topSpeed'|'dragCoeff'|'accelMul'
  |'sidecutRadius'|'edgeGrip'|'landingTolerance'|'sketchySpeedLoss'|'crashImpact'|'popImpulse'|'snowFriction';
function resolveFeel(cfg: FeelConfig): Record<FeelKey, number>; // arcade + realism*(real-arcade), override wins

// tricks & scoring (tricks.json / scoring.json)
interface TrickDef { id: string; name: string; kind: 'spin'|'flip'|'grab'|'rail'|'butter'|'drop';
  match: { spin?: number; flip?: number; grab?: GrabId; rail?: 'fifty'|'slide'; minAir?: number }; base: number }
interface ScoringConfig { rotationPer180: number; grabPerSecond: number; airPerSecond: number;
  landing: Record<'clean'|'sketchy', number>; comboStep: number; comboMax: number; comboTimeout: number }

// level data
interface LevelData { id: string; bounds: { size: number; vertical: number }; heightmap: string; gridRes: number;
  dropPoints: DropPoint[]; trails: TrailSpline[]; edits: TerrainEdit[]; features: Feature[]; challenges: ChallengeDef[] }
type TerrainEdit = { op: 'raise'|'lower'|'flatten'|'smooth'|'noise'; at: Vec2; radius: number; amount: number; falloff?: number }
  | { op: 'carveTrail'; spline: Vec2[]; width: number; depth: number; smooth: number }
  | { op: 'cliff'; at: Vec2; dir: number; width: number; height: number }
  | { op: 'windLip'; spline: Vec2[]; height: number };
type Feature = { id: string; type: 'kicker'|'rail'|'box'|'halfpipe'|'rock'|'tree'|'crevasse'|'cornice'|'avalancheZone';
  pos: Vec3 | Vec2; yaw: number; params: Record<string, number> };
interface ChallengeDef { id: string; name: string; zone: { spline: Vec2[]; width: number } | { center: Vec2; radius: number };
  medals: { bronze: number; silver: number; gold: number } }
interface DropPoint { id: string; pos: Vec3; yaw: number; kind: 'heli' }

// render / save
interface Snapshot { prev: WorldState; curr: WorldState; alpha: number; events: SimEvent[] }
interface SaveStore { load(): SaveData; save(d: SaveData): void }   // localStorage now; cloud later
```

---

## 6. Performance budget (40 FPS = 25 ms, M5 MacBook Air, sustained)

| Item | Budget |
|---|---|
| Sim (≤2 × 60 Hz steps/frame incl. Rapier) | ≤ 2.5 ms CPU |
| Render CPU (culling, submission, UI) | ≤ 6 ms |
| GPU: scene | ≤ 8 ms |
| GPU: shadows (cascades) | ≤ 3 ms |
| GPU: painterly post (Kuwahara @ 0.5–0.75×) + grade | ≤ 5 ms |
| GPU: bloom + FX + snowfall | ≤ 2.5 ms |
| Draw calls | ≤ 300 (instancing for rocks/trees/spray) |
| Triangles on screen | ≤ 1.5 M (terrain chunk LOD; rider ≤ 25 k) |
| Texture memory | ≤ 400 MB; KTX2 everywhere |
| Initial download | ≤ 30 MB |

- Internal render resolution defaults to **1× CSS pixels** (not retina). The painterly filter hides the softness, which is a free win.
- **Dynamic resolution scaling** holds ≥ 40 FPS (0.6×–1.0×).
- **Quality tiers** in `quality.json` (low/med/high). Low is the future mobile tier.
- A perf HUD (dev) shows frame time p50/p99 and 1% lows. There is a **10-minute soak test** for fanless thermal throttling.

---

## 7. Milestones (MVP-first, each = a group of atomic issues)

| # | Milestone | Contents | Est. (part-time) |
|---|---|---|---|
| M0 | **Foundations** | Issue 1 test suite (Vitest), Issue 2 CI (`ci` job: lint, types, tests, build, Playwright smoke), Issue 3 Puppeteer MCP; then Vite+TS+Three WebGPU scaffold, sim/render boundary lint rule, fixed-step loop, Zod data loading, GitHub Pages deploy | 3–4 days |
| M1 | **Art spike** ⭐ | Toon ramp + coloured shadow nodes, snow & rock materials, sky gradient + sun, cloud sea, distance/height fog, cascaded shadows, post pipeline (bloom, Kuwahara, brush overlay, grade), stand-in rider, free-fly camera, reference comparison overlay → **user sign-off** | 1.5–2 wks |
| M2 | **Mountain v0 + ski core** (= *playable MVP*) | DEM → PNG16 tool, heightfield + edit ops + HMR, chunked LOD terrain, Rapier heightfield, keyboard input, feel config + realism blend, ski controller (carve/skid/tuck/brake), jump/air, landing quality, cinematic camera, dev panel + telemetry + save-to-JSON, pins, heli drop spawn | 2 wks |
| M3 | **Rider** | ComfyUI concept + rider GLB pipeline (primary/fallback), outfit config, pose data + blending, 2-bone IK, spring bones, powder spray, ski tracks, speed lines | 1.5 wks |
| M4 | **Crashes** | Obstacle collisions, small fall + recovery, Rapier ragdoll, detached skis, get-up blend | 1 wk |
| M5 | **Tricks & scoring** | Air spins/flips, grabs, switch detection, rails/boxes, butters, drops, trick detector + classifier, scoring + combo chain, trick/score popups | 1.5 wks |
| M6 | **World & challenges** | Parametric kickers/rails/boxes/halfpipe/cliffs/cornices/crevasses, instanced rocks/trees, avalanche test zone, 3 trails, challenge zones + medals + local save, map view, world markers; **mountain tuning sessions with the user** | 1.5–2 wks |
| M7 | **Environment & polish** | 30-min day/night keyframes, moonlit night, weather toggle + snowfall, Settings menu, quality tiers + dynamic res, soak test/perf pass | 1 wk |

**Total ≈ 9–12 weeks part-time (~45–60 atomic issues).** This is up from the earlier 4–7-week guess. The main drivers are procedural rider animation (no ski mocap exists), ragdoll, and the art spike. The playable MVP (ride the mountain in the target look) lands at the end of M2, **~4–5 weeks** in.

**Iteration 2 (not in these issues):** audio (high priority), snowboard, movable heli, gamepad/touch, customisation.

---

## 8. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Painterly post too costly / "filter-over-3D" look | Half-res Kuwahara + dynamic res. Push most of the look into materials (toon ramp, coloured shadows, painted textures) so post is a finishing layer. Proven in M1 before gameplay. |
| Three.js WebGPU/TSL API churn | Pin the exact three version. Upgrade deliberately via an issue. |
| Rider quality from AI image-to-3D | CC0 rigged base fallback. The stand-in keeps other work unblocked. |
| No ski animation clips | Data-driven key poses + IK + spring bones. Mixamo only for falls/get-ups. |
| Ragdoll feels janky | Tune joint limits/damping in the dev panel. Keep ragdoll short (~2.5 s) and blend out early. |
| DEM licensing/processing | swisstopo open data with attribution in credits. Procedural fallback. |
| Safari WebGPU quirks | Chrome is the primary dev browser. Safari is checked per milestone. WebGL2 fallback. |
| Scope creep | Atomic issues with Out of Scope sections. Iteration-2 list kept separate. |

---

## 9. After approval (execution steps for this phase)

1. Save this plan verbatim to **`context/plan.md`**.
2. Update `context/progress.md`: Phase 2 ✅, and replace the estimate with the milestone table above (9–12 weeks; playable MVP ~4–5 weeks).
3. Commit (`Add technical plan for iteration 1`) and push to `main`.
4. Next phases: **Phase 3** `/init` (keep `CLAUDE.md` lean, and add the stack + commands once scaffolded), then **Phase 4** using `context/issue-generation-prompt.md`. The outline is presented for approval before any `gh issue create`.

## 10. Verification (how this plan will be validated as it's built)

- **Per issue:** Vitest unit tests (sim logic: heightfield sampling, edit ops, feel resolve, ski controller integration on a test slope, trick classification, scoring, challenges, time-of-day mapping), `npm run lint`, `tsc --noEmit`, and the Playwright smoke test (page loads, canvas renders, no console errors) all green locally and in the `ci` job.
- **Headless sim integration tests:** scripted inputs down the main trail reach the bottom in **55–65 s**. A standard kicker gives air time within the configured range. A 360 input sequence lands a classified "360".
- **UI/visual:** Puppeteer MCP screenshots for every UI or look change. In M1, compare against `context/reference/` frames with the user.
- **Performance:** perf HUD + 10-minute soak on the M5 Air holding ≥ 40 FPS p99 at default quality.
