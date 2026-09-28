# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workflow — read first, every session
- Follow `context/workflow.md` for all work. Check `context/progress.md` to see the current phase.
- `context/` is the source of truth: `brainstorm.md` (spec), `plan.md` (stack/architecture/APIs — don't re-litigate), `art-reference.md` + `reference/` (art direction — top priority), `issue-generation-prompt.md`.
- Build issues with `/process-issue <number>`. One issue → one branch → one PR. Commit per logical step, reference the issue number, push after every commit.
- Per-issue plans go in `scratchpads/`.
- New bugs/features → brainstorm, then create a GitHub issue. No ad hoc changes.

## Stack
Browser game: TypeScript + Vite + Three.js (`WebGPURenderer`, TSL shaders, WebGL2 fallback) + Rapier (WASM) + Preact UI + Zod data schemas. Tests: Vitest + Playwright; lint: ESLint + Prettier + `tsc --noEmit`. CI job is named `ci`.

## Commands
_No code yet — add build/dev/lint/test (incl. single-test) commands here once the M0 scaffold lands._

## Architecture (big picture — details in `context/plan.md`)
- **`src/sim/` is pure and deterministic:** no `three`, no DOM, no `Date.now()`/`Math.random()`; fixed 60 Hz `step(inputs, dt)` → serialisable `WorldState` + `SimEvent[]`. ESLint enforces the boundary. This keeps sim testable in Node and reusable by a future multiplayer server.
- **Render/camera/UI only read interpolated snapshots** of sim state and react to `SimEvent`s; they never write sim state. Cosmetic systems (spray, tracks, spring bones, camera) live render-side.
- **Data-driven tuning:** feel (realism blend `arcade + realism·(real − arcade)`, realism = 0.10), tricks, scoring, lighting, bindings and the whole mountain (`src/data/levels/matterhorn/`: base DEM + ordered `terrain-edits.json` + `features.json` + `challenges.json`) are JSON validated by Zod and hot-reloaded. Prefer editing data over code when tuning with the user.
