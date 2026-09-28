# Alpine_Chase — Project Spec & Planning Prompt

> **How to use this file:** In Claude Code, enter **plan mode**, then say:
> "Plan Alpine_Chase. Think hard. Refer to @context/brainstorm.md, @context/art-reference.md and the frames in context/reference/. Outline the tech stack, technical architecture, data flow and module APIs. Save the approved plan to context/plan.md."

---

## 1. One-line pitch

A browser-based, single-player **free-roam skiing game on one stylised Matterhorn-inspired mountain**. It plays like *Steep* but slightly more grounded, and is rendered in the **painterly, anime-cinematic art style** of the reference video (`context/art-reference.md`, `context/reference/*.jpg`). The feel is upbeat, bright and fast.

## 2. Priorities (in order)

1. **Art direction and feel.** Match the reference as closely as real-time rendering allows. This is the core feature of iteration 1.
2. **Ski feel.** Movement, carving, jumps and crashes must feel great. The test mountain exists to tune this.
3. **Performance.** Target **40 FPS** in a desktop browser on the **reference device: the user's MacBook Air (Apple Silicon, 16 GB RAM, integrated GPU, fanless — so sustained-load thermal throttling must be considered)**. If 40 FPS forces a major art compromise, flag it and discuss; don't silently downgrade the look.
4. **Robust foundations** for later features (multiplayer, mobile, customisation, audio). Design for them, but don't build them.

## 3. Iteration 1 (MVP) scope

### In scope
- **One mountain**, a fixed hand-authored asset (the same every session), modelled on the **Matterhorn**. Gameplay beats accuracy: reshape freely for better lines.
  - A **~1 minute top-to-bottom** run on the main line at typical speed.
  - A few **test trails** (at least 3 distinct lines: groomed/cruisy, freeride/off-piste, park line).
  - **Terrain features:** kickers, rails/boxes, cliffs/drops, rocks, trees, cornices/wind lips, a halfpipe (small), step-downs, hips.
  - **Obstacles:** rocks, trees, crevasses, and a scripted/test avalanche zone. **No other skiers.**
  - The mountain is built **iteratively with the user**. They play, suggest changes, and Claude edits the mountain data. The pipeline must make this fast (see §8).
- **Helicopter spawn** at the summit. The player jumps out of the heli to start a run. Later the heli will be movable around the mountain, so design the spawn system for arbitrary drop points, but in iteration 1 it is fixed at the top.
- **Skis only.** Snowboarding comes after the ski core and art direction are nailed.
- **Default rider = the reference character** (spiky platinum-blond hair, white goggles with iridescent lens, navy oversized hoodie with a white splatter graphic, cream knit cuffs, baggy khaki pants, black gloves). Swap the board for skis; salmon/terracotta skis match the board colour. Customisation comes later, so keep rider appearance **data-driven** (outfit/colour config), not hard-coded.
- **Controls:** keyboard only for now. An input abstraction layer is needed so gamepad/touch can be added later without touching gameplay code.
- **Camera:** automatic cinematic third-person. It chases from behind/above, drops low near the skis at speed, and pulls wider in the air. No manual camera control in iteration 1.
- **Crashes:**
  - Small falls: the rider stumbles/falls and gets up quickly (short recovery animation).
  - Big crashes: skis detach and fly off, the rider **ragdolls**, then picks themselves up shortly after (skis snap back or are re-equipped).
- **Tricks and scoring:** a freestyle score system (see §6) with **medals unlocked at score thresholds**.
- **Challenges:** freestyle score challenges, shown as **markers on a map and visible in the world** while playing.
- **Day–night cycle:** a **30-minute real-time cycle** with short, not-too-dark nights (e.g. ~20 min day, ~3 min dusk, ~4 min moonlit night, ~3 min dawn). Night stays readable with blue moonlight, never pitch black.
- **Weather:** a toggle (settings option) for at least clear / snowfall / overcast-fog.
- **UI:** minimal and cinematic. No cluttered HUD. Trick names and score pop up briefly. The map is opened with a key. There is a small settings menu (weather toggle, quality).
- **Dev tuning panel** (dev builds only). Live sliders for physics/feel parameters, camera, lighting and post-processing, so the user can fine-tune feel while playing and values can be saved to config files.

### Out of scope for iteration 1 (design for, don't build)
- **Audio** (music, wind, carve sounds). **Next iteration and high priority**, because it's key to the reference's feel. Add an audio event hook layer now (gameplay emits events like `carve`, `land`, `crash`, `airborne`) so audio can plug in later.
- Snowboarding.
- Multiplayer (but see §7 for the foundations it needs).
- Mobile/touch/gamepad controls.
- Character customisation.
- Movable helicopter drop points.
- Races, gates, collectibles and other challenge types.
- Accounts/cloud saves. Progress (medals, best scores) saves locally in the browser.

## 4. Art direction requirements

Full breakdown: `context/art-reference.md`. Key frames: `context/reference/`.

- Stylised/toon shading with **coloured shadow ramps**: snow shadows are periwinkle/blue-violet, never grey.
- **Painterly post-process** (Kuwahara/oil-paint style filter or similar, plus subtle brush/paper texture). This is the single biggest lever for matching the reference.
- No heavy black outlines. Edges read through value contrast.
- Cobalt-to-cyan sky gradient, sun bloom/flare, strong atmospheric distance haze, and a **sea of clouds** below the summit.
- Chunky, faceted dark rock with painted planes. Smooth sculpted snow with wind lips.
- **Speed feel:** painterly powder spray from the ski edges, ski tracks left in the snow, wind-driven secondary motion on hair/hood/clothes, subtle speed lines/motion blur at high speed.
- The plan must include an early **"art spike"**: a single scene (terrain patch + rider stand-in + sky + post-process) whose job is to prove the look before gameplay is built on top. The user signs it off against the reference frames.

## 5. Ski physics & feel — "Steep, but 10% more realistic"

Quantify this with a **realism blend parameter**:

- For every feel parameter, define an **arcade value `A`** (Steep-like, estimated by feel) and a **realistic value `R`** (real-world skiing).
- The shipped value is `A + 0.10 × (R − A)`, exposed as a single global `realism = 0.10` slider in the dev panel, with per-parameter overrides.
- Parameters to include (at minimum):
  | Parameter | Arcade direction | Realistic direction |
  |---|---|---|
  | Gravity multiplier in air | higher (snappy) | 1.0 g |
  | Air/hang time on a standard kicker | longer | physically correct |
  | Spin rate in air (°/s) | fast | slower |
  | Air control (steering mid-air) | strong | near zero |
  | Top speed / terminal velocity | capped, forgiving | higher, drag-based |
  | Acceleration from standstill | quick | gradual |
  | Carve turn radius at speed | tight | wider (sidecut-based) |
  | Edge grip on ice/steep | very grippy | can wash out |
  | Landing angle tolerance (°) | wide | narrow |
  | Speed loss on sketchy landing | small | larger |
  | Crash threshold (impact force) | high | lower |
- Include telemetry in the dev panel (speed km/h, air time, spin rate, landing angle, G-force) so feel can be compared across tuning sessions.
- Physics must run at a **fixed timestep, decoupled from rendering** (important for feel consistency and multiplayer later).

## 6. Tricks & scoring

### Ski trick list (reference)
- **Straight airs / old-school:** straight air, spread eagle, daffy, twister, backscratcher, iron cross.
- **Grabs:** mute, safety, japan, tail, tip, critical, blunt, truck driver, octo grab.
- **Spins (flat/on-axis):** 180, 360, 540, 720, 900, 1080 (1260+ later). Switch take-off/landing variants.
- **Off-axis / inverts:** backflip, frontflip, cork 540/720/900, rodeo 540/720, misty 540/720, bio (later), double cork 1080 (later).
- **Rails/boxes (jibs):** 50-50 (straight), slide (sideways/"lipslide"/"boardslide" equivalents), 270 on, 270 off, switch-ups.
- **Ground/butter:** nose butter, tail butter, butter 180/360, switch skiing, powder/carve turns.
- **Terrain moves:** kicker jump, step-down, step-up, hip, gap, cliff drop, cornice drop, halfpipe airs (alley-oop later).

### Iteration 1 MVP trick set
Jump/ollie (pop), spins 180–1080 in both directions, backflip, frontflip, 6 grabs (mute, safety, japan, tail, tip, critical), grab + spin combos, nose/tail butter, rail 50-50 and slide, cliff/cornice drops, switch landing detection.

### Scoring
- Base points per trick, plus rotation, grab-hold duration, air time and landing quality (clean / sketchy / crash = 0 for that chain).
- **Combo chain multiplier** while tricks are linked without crashing or stopping.
- **Challenge zones** (map + world markers): hit a target freestyle score in a zone/line to earn **Bronze / Silver / Gold medals** at defined thresholds.
- Scoring rules must be **data-driven** (JSON/config), so they can be tuned without code changes.

## 7. Foundations for future features (design now, build later)

- **Multiplayer-ready:** keep simulation state separate from rendering. Put deterministic-ish fixed-step physics behind a clear `step(state, inputs, dt)` boundary. Player state must be serialisable. Inputs should be plain data (usable later for network replication/prediction). No global singletons for per-player state.
- **Mobile-ready:** input abstraction layer, resolution/quality scaling, UI that can reflow to touch later, and a performance budget (draw calls, triangles, texture memory) documented in the plan.
- **Customisation-ready:** rider appearance from config (meshes/colours/materials).
- **Snowboard-ready:** the rider/equipment model should allow a second equipment type later without rewriting the physics core (e.g. equipment defines edge/sidecut/stance parameters).
- **Audio-ready:** gameplay event bus (see §3).
- **Heli-ready:** a generic spawn/drop-point system.

## 8. Content & asset pipeline (to be finalised in planning)

Preliminary recommendation (the plan should confirm or improve it):
- **Mountain terrain:** start from a real **Matterhorn heightmap** (e.g. swisstopo swissALTI3D / SRTM DEM, free), crop and scale down to fit a ~1-minute run, then layer hand-authored edits (trail carving, feature placement, smoothing) stored as **data files in the repo**. That way Claude can change the mountain in code when the user gives feedback ("make this cliff smaller", "add a kicker here") and the result is reproducible.
- **Placed features** (kickers, rails, rocks, trees, halfpipe, crevasses) are defined in a level data file (positions/rotations/params), with procedural/parametric meshes for kickers, rails, halfpipe, etc.
- **User hardware constraint:** MacBook Air, 16 GB RAM, no dedicated GPU. ComfyUI is installed. Blender is **not** installed, and the user has no Blender experience. Local image-to-3D (TRELLIS/Hunyuan3D) is **not viable** on this machine (CUDA-only or far too slow). ComfyUI is fine for 2D work (SDXL-class models, ~1024px, slowish).
- **Rider:** generate concept/turnaround art matching the reference in **ComfyUI**. Then either (a) convert it with a **hosted image-to-3D service** (e.g. Tripo / Meshy / Hunyuan3D web, free tiers), or (b) start from a **CC0 rigged stylised base character** (e.g. Quaternius) restyled with our outfit meshes/materials. Auto-rig and animate via **Mixamo** (free). Export to glTF/GLB. The plan should pick one, with a fallback. The art spike can use a simple stand-in rider.
- **Blender (free) is recommended to install, but the user won't operate it.** Claude drives it **headlessly via Python scripts** (`blender --background --python script.py`) for clean-up, decimation, retargeting and GLB export, so asset steps are reproducible scripts in the repo.
- **Textures:** hand-painted-style snow/rock/cloth textures and skybox/cloud elements generated in **ComfyUI** (the user runs them from prompts/workflows that Claude provides and saves them into the repo). Also prefer **procedural/shader-generated** textures where possible, to reduce the asset burden.
- **Props:** CC0 stylised packs (e.g. Quaternius, Kenney) or ComfyUI → 3D, restyled by our shaders.
- All assets use **glTF/GLB**, compressed (Draco/Meshopt + KTX2) for web performance.

## 9. Tech stack direction (to be finalised in planning)

User requirements: browser is fine; fast to iterate; must scale to mobile and multiplayer later; powerful enough for upward expansion; Claude Code will write nearly all the code, so a **code-first engine** (not editor/GUI-driven) is strongly preferred.

Preliminary recommendation for the plan to evaluate:
- **Three.js (WebGPU renderer with WebGL2 fallback, TSL node shaders) + TypeScript + Vite.** Largest web-3D ecosystem, full shader control for the painterly look, instant hot reload, runs in mobile browsers, and can be wrapped with Capacitor for app stores later.
- **Rapier (WASM)** for physics/ragdoll: deterministic mode is useful for future multiplayer.
- **Vitest** (unit/integration), **Playwright or Puppeteer MCP** (browser/UI checks), **ESLint + Prettier**, **GitHub Actions** CI with a job named `ci`.
- **lil-gui/Tweakpane** for the dev tuning panel.
- Compare against Babylon.js and Godot 4 (web export) in the plan, and explain the choice.
- Hosting: a static host (GitHub Pages / Cloudflare Pages / Vercel), to be decided. It should be cheap or free.

## 10. What the plan must output

1. **Tech stack** with a justification for each choice: engine/renderer, language, physics, UI, testing, linting, build, hosting. Auth/email/database/object storage are **not needed in iteration 1**; state that explicitly and note what would be used later for multiplayer/accounts.
2. **Technical architecture:** module layout (e.g. `core/` sim loop, `input/`, `physics/`, `rider/`, `camera/`, `terrain/`, `world/features`, `tricks/`, `scoring/`, `render/` shaders + post, `ui/`, `devtools/`, `data/`), plus the boundary between simulation and rendering.
3. **Data flow:** input → sim step → events → scoring/UI/camera/render. Include the level-data → terrain/feature build pipeline.
4. **Module APIs:** key interfaces/types (sim state, input frame, events, trick definitions, level data schema, feel-parameter config).
5. **Performance budget** for 40 FPS.
6. A **milestone breakdown** that front-loads the art spike, then ski feel, then tricks/scoring, then world/challenges, day–night and weather. Include a rough time estimate per milestone.
7. **Risks and open questions** (painterly post-process cost, ragdoll feel, asset generation quality, heightmap licensing).

## 11. Working agreement

- Follow `context/workflow.md` (atomic GitHub issues, `/process-issue`, CI must pass).
- MVP first, then iterate.
- Give the user **time estimates and progress updates**, tracked in `context/progress.md`.
- The mountain and feel are tuned **with the user while they play**. Prefer data/config changes over code changes for tuning.
