# AGENTS.md

Monorepo of prompt-driven AI experiments (WebGL games/demos). Every top-level folder is a self-contained project with its own `_ai/` knowledge base; the root `_ai/` is the monorepo-level KB.

## Layout & docs gotchas

- Current first project: `space-invaders/ds-v4-flash/` (the game "City Defender 3D"). The root `README.md` and the project's plan file still reference the pre-move layout (`ds-v4-flash/`, `prompt.md` at repo root, workspace root `/home/marc/devel/space-invaders-ai/`) — trust on-disk paths over those docs.
- `space-invaders/prompt.md` is the original demo-generation prompt, context only.
- Cross-project knowledge lives in `_ai/lessons_learned/`: add `YYYY-MM-DD-topic.md` and update the index table in `_ai/lessons_learned/README.md`.
- Per-project `_ai/` convention: implementation plans in `_ai/backlog/active/`, final reports in `_ai/backlog/reports/`, decisions in `_ai/technical_decisions/`.

## ds-v4-flash — City Defender 3D

- The entire game is ONE file: `space-invaders/ds-v4-flash/index.html` (~4500 lines). Hard constraint from the implementation plan: single self-contained file, no build step, no packages. Three.js r160 + GSAP load via CDN import map; OrbitControls and lil-gui are imported dynamically only in demo mode.
- Run: `python3 -m http.server 8080` from the project dir (also works from `file://`).
- URL params: `?demo=1` = legacy passive demo (OrbitControls + lil-gui); `?preserve=1` = keep drawing buffer (needed for screenshots).
- No test runner exists. Verify in the browser console: game mode exposes `window.app` (`THREE`, `scene`, `camera`, `GameState`) and `window.app.cheat` with `spawn(type,n)`, `boss()`, `killBoss()`, `damageAll()`, `damageBuilding(id,amt)`, `score(n)`, `win()`, `lose()`, `state()`, `fire(on)` — use these to reach boss/victory/game-over states deterministically instead of playing through.
- High score persists in `localStorage` key `city-defender-3d.highscore`; score is clamped to ≥ 0.

## E2E / headless testing (hard-earned — see `_ai/lessons_learned/2026-08-12-city-defender-e2e-playwright.md`)

- No Playwright browser binaries on this system (Arch): launch system Chromium at `/usr/lib/chromium/chromium` with `--use-angle=swiftshader --enable-unsafe-swiftshader --no-sandbox`.
- Use `waitUntil: 'domcontentloaded'` with a long timeout — CDN fonts/module scripts block `load` for 30–60 s headless.
- Never assert on wall-clock sleeps: the game clamps `dt` to 0.05, so game time drifts from wall time at headless FPS. Poll for state every ~250 ms with generous timeouts.
- Aim at 3D targets in tests by projecting world positions to screen NDC then pixels; skip targets behind the camera (`p.z > 1 || p.z < -1`).
- If a test file gets mangled by repeated patch attempts, rewrite it from scratch rather than patching again.