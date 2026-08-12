# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial repository scaffolding with the demo-generation prompt (`space-invaders/prompt.md`).
- First "Alien Invasion — City Destroyer" WebGL demo: procedural cyberpunk city (New York, Paris, Berlin), destructible buildings with laser attacks, particle explosions, Web Audio sound effects, and OrbitControls + lil-gui.
- Alien victory state, searchlight batteries, scout UFOs, and city traffic.
- Two additional model-generated takes on the demo:
  - `gemini-3.5-flash/` — "Alien Invasion: City Destroyer" Three.js game (r128) with procedural cities, mothership laser attacks, particle engine, and Web Audio synth.
  - `gemini-3.6-flash/` — Alien Invasion 3D browser game with procedural city landmarks, neon mothership, custom particle physics, and raycasting-based laser targeting.
- City Defender 3D implementation plan (`space-invaders/ds-v4-flash/_ai/backlog/active/`).
- Monorepo knowledge base: root `README.md`, `AGENTS.md`, and `_ai/` (lessons learned, backlog, technical decisions).

### Changed

- Rewrote the "Alien Invasion" demo into the "City Defender 3D" arcade game (single-file `index.html`, Three.js r160 + GSAP), with player-controlled turret defense, waves, boss fights, and deterministic cheat API for testing.
- Moved the first project into the `space-invaders/ds-v4-flash/` folder and `prompt.md` into `space-invaders/` as part of the monorepo restructure.
- Documented the E2E Playwright setup for headless Chromium with SwiftShader (see `_ai/lessons_learned/`).
