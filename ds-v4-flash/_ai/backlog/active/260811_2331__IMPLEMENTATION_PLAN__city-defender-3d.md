---
filename: "_ai/backlog/active/260811_2331__IMPLEMENTATION_PLAN__city-defender-3d.md"
title: "City Defender 3D — turn the Alien Invasion WebGL demo into a playable arcade wave-survival game"
createdAt: 2026-08-11 23:31
createdBy: "opencode [deepseek-v4-flash-free]"
updatedAt: 2026-08-11 23:31
updatedBy: "opencode [deepseek-v4-flash-free]"
status: draft
priority: high
tags: [threejs, webgl, game, arcade, single-file, javascript]
project: ds-v4-flash
estimatedComplexity: complex
documentRevision: 1
documentType: IMPLEMENTATION_PLAN
---

# City Defender 3D — Implementation Plan

> **Plan file:** `_ai/backlog/active/260811_2331__IMPLEMENTATION_PLAN__city-defender-3d.md`

## 1. Problem Statement

The current codebase (`ds-v4-flash/index.html`, ~2717 lines) is a *passive* WebGL demo: a procedurally generated city (New York / Paris / Berlin) is destroyed by an autonomous alien mothership while the user watches through OrbitControls and a lil-gui panel. It is not a game — there is no player agency, no win/lose condition, no score, and no persistence.

The goal of this plan is to evolve it into **"City Defender 3D"**, a playable single-file arcade game in the spirit of classic Space Invaders:

- A player-controlled **drone** auto-pilots a patrol route around the city; the player aims with the mouse (crosshair / Fadenkreuz) and fires.
- **4 enemy types**: Scout UFOs, Bombers (drop bombs on buildings), Dive Drones (ram the player and stun the cannon), and a **Mothership boss** with laser-barrage phases.
- **Wave survival structure** (~8–10 waves + boss). City has HP (buildings); losing all buildings = game over. Defeating the boss = victory.
- **Pure arcade scoring**: points per kill, minus points for friendly fire (accidentally shooting buildings or city traffic cars).
- City selected at the start menu (one city per run; buildings persist damage — they dim, char, lean, and become rubble instead of vanishing).
- Full polish layer: WebAudio-synthesized SFX (no audio files), start-menu, wave banners, HUD, game-over/victory screens, high score persisted in `localStorage`.
- **Constraint: stay a single self-contained `index.html`** (Three.js + GSAP via CDN), preserving the existing demo's visual DNA (neon cyberpunk, fog, particle systems, city generators).

## 2. Implementation Notes

### 2.1 Relevant root directories & files

```
/home/marc/devel/space-invaders-ai/            # workspace root (git repo)
  prompt.md                                    # original demo prompt (context only)
  ds-v4-flash/                                 # project root
    index.html                                 # THE ONLY SOURCE FILE (2717 lines)
    README.md                                  # stub — must be rewritten
    CHANGELOG.md                               # must gain a new version entry
    _ai/backlog/active/                        # this plan lives here
    _ai/backlog/reports/                       # final report goes here
```

### 2.2 Current `index.html` architecture (what we build on)

| Section (approx. line) | Content |
|---|---|
| 129–307 | `CONFIG`, seeded RNG (`mulberry32`), texture factories (`makeFacadeTexture`, `makeGlowTexture`, `makeSkyTexture`, `makeGridTexture`) |
| 311–432 | Renderer, fog (FogExp2), camera, **OrbitControls** (to be removed/disabled), sky, moon light, stars |
| 432–574 | City base, ground grid, searchlight battery pools |
| 574–682 | Graveyard / city-blocking pools |
| 746–1102 | `CityGenerator` — New York / Paris / Berlin builders (REUSE AS-IS) |
| 1106–1252 | `Mothership` — disc/ring structure, rotation, targets (REUSE core, extend for boss) |
| 1256–1360 | `ScoutShip`, `ScoutFleet` (REUSE visual, re-skin as enemy) |
| 1395–1462 | `BeamPool` — laser beam cylinders |
| 1495–1609 | `ImpactFX` — rings, flashes, lights |
| 1614–1787 | `ParticlePool` — fire/smoke particles |
| 1806–1933 | `CarRig` — city traffic cars (friendly-fire target) |
| 1937–1978 | `DotPool` – small dot sprites (used for windows/bullets) |
| 1982–2577 | `Game` class — master orchestration, auto-attack loop |
| 2578–2683 | lil-gui panel wiring (to be replaced by real game UI) |
| 2683–2717 | Boot, resize, RAF loop |

### 2.3 Key technical decisions baked into this plan

1. **No build step, no modules.** Everything stays in `index.html`; the game must run from `file://` (double-click) and over `http://localhost`.
2. **Remove OrbitControls + lil-gui** from the play path (they belong to the old demo). GSAP stays (menu/wave transitions).
3. All mutable gameplay state lives in dedicated classes with narrow responsibilities (SRP): `Game` becomes a thin state coordinator; combat, AI, scoring, audio, and UI each get their own class.
4. All performance-critical effects continue to use the existing pool pattern (`BeamPool`, `ParticlePool`, `DotPool`). Bullets get a new `BulletPool`; buildings go through a new `CityIntegrity` facade instead of being deleted directly.
5. **SOLID mapping** (explicitly followed):
   - **SRP**: `GameStateMachine` owns transitions; `ScoreKeeper` owns points; `WaveDirector` owns composition; `AimController` owns input/raycast; `PlayerDrone` owns motion; each enemy class owns its AI.
   - **OCP**: new enemy types extend an `Enemy` base without touching `Game`; waves are composed from a descriptor table.
   - **LSP**: `BossMothership`, `ScoutEnemy`, `BomberEnemy`, `DiveDroneEnemy` all satisfy the `Enemy` interface (update/damage/destroy/spawn FX).
   - **ISP**: `AimController` exposes only `update(ray, dt)`; it never touches score or HUD directly — it emits hit events.
   - **DIP**: `Game` depends on abstractions (`Enemy`, pooled effects) — never on concrete Three.js mesh manipulation from outside those classes.

### 2.4 Commands / testing

No build or test runner exists. Verification is manual browser testing:

```bash
# serve locally (recommended so localStorage + CDN work identically)
cd /home/marc/devel/space-invaders-ai/ds-v4-flash && python3 -m http.server 8080
# then open http://localhost:8080
```

- Before each phase: open the page, check the DevTools console for errors.
- After phases with gameplay: run the manual test checklist from the phase's Verification section.

## 3. Phase Architecture Overview

```
MENU -> WAVE_BANNER -> PLAYING -> (GAME_OVER | VICTORY)
          ^                         |
          +---- next wave ----------+
```

Phases are ordered so the game is playable incrementally: shell first, then the drone, then enemies, then damage model, then the boss, then audio/polish, then documentation & report.

---

## Phase 1 — Game Shell: states, screens, high score

**Objective:** Transform the demo boot path into a stateful game with a start menu, city selection, HUD, and game-over/victory endpoints. Nothing is playable yet; the demo visuals keep running behind the menu as an attract mode.

### Tasks

1.1 Add a `GameStateMachine` (MENU / WAVE_BANNER / PLAYING / GAME_OVER / VICTORY) with `setState(state, payload)` and a small DOM overlay controller.

1.2 Replace the lil-gui panel wiring (~lines 2578–2683) with a hidden-on-demand HUD: score, wave, city-HP bar, "CANNON STUNNED" flash — all DOM elements, styled to the neon theme.

1.3 Build the **start menu overlay** (title "CITY DEFENDER 3D", subtitle, three city buttons, high-score readout, START button). City buttons pre-select the city; START triggers city generation + `MENU -> WAVE_BANNER`.

1.4 Add a `ScoreKeeper` class: `add(points)`, `subtract(points)`, `streak`-free pure score, `getHighScore()`/`saveHighScore()` through `localStorage`, and a `reset()` for new runs.

1.5 Wire `GAME_OVER` and `VICTORY` screens with score + high score + "Play again" (reload state, return to MENU).

1.6 Keep the old mothership destruction demo available behind a hidden dev toggle (`?demo=1` URL param re-enables OrbitControls + auto-destruction) so nothing is lost during the transition.

### Code sketches

[MODIFY] `index.html` — state machine + score keeper (inserted before the `Game` class):

```js
const GameState = Object.freeze({
  MENU: 'MENU',
  WAVE_BANNER: 'WAVE_BANNER',
  PLAYING: 'PLAYING',
  GAME_OVER: 'GAME_OVER',
  VICTORY: 'VICTORY',
});

class GameStateMachine {
  constructor() { this.state = GameState.MENU; this.handlers = {}; }
  on(state, fn) { this.handlers[state] = fn; }
  setState(state, payload) {
    this.state = state;
    if (this.handlers[state]) this.handlers[state](payload);
  }
}

class ScoreKeeper {
  constructor() { this.score = 0; this._key = 'city-defender-3d.highscore'; }
  add(p) { this.score += p; }
  subtract(p) { this.score = Math.max(0, this.score - p); }
  reset() { this.score = 0; }
  getHighScore() { return Number(localStorage.getItem(this._key) || 0); }
  saveHighScore() { localStorage.setItem(this._key, String(Math.max(this.getHighScore(), this.score))); }
}
```

### Deliverables

- `index.html` modified: states, menu, HUD skeleton, score, screens, `?demo=1` fallback.

### Verification

1. Open `http://localhost:8080` → start menu shows, city buttons highlight, high score reads 0.
2. Click START with no enemies existing yet → wave banner shows "WAVE 1", then joins PLAYING with the demo city intact and HUD visible.
3. Force `setState(GameState.GAME_OVER)` from console → game-over screen shows score; localStorage contains `city-defender-3d.highscore` after save.
4. Open `?demo=1` → old OrbitControls + auto-destruction demo still runs.

---

## Phase 2 — Player Drone, aim & firing

**Objective:** Add the playable core: an auto-piloting `PlayerDrone` with a chase camera, mouse-aim crosshair, raycast shooting, bullet pooling, and friendly-fire penalties against buildings and traffic (minus points).

### Tasks

2.1 Build `PlayerDrone` (procedural jet: cone fuselage, box wings, glowing engine emitters + `ParticlePool` engine trail). It flies a closed circular patrol path around the city (radius ≈ `cityRadius * 1.15`, altitude ≈ 70), banking into turns via look-at + slerp.

2.2 Replace `OrbitControls` usage during PLAYING with a **chase camera**: `camera.position = drone.position - forward * 26 + up * 8`, look-at slightly ahead of the drone; smooth lerp. OrbitControls remains active only in `?demo=1` mode.

2.3 Add `AimController`: a CSS crosshair that follows the mouse; renders a `Raycaster` ray from camera through the cursor. Mouse-down (hold) = auto-fire with a fire-cooldown (e.g. 0.12 s). Exposes `emitHit(ray)` semantics — firing is decided by the combat layer, not the input class (ISP).

2.4 Add `BulletPool`: pooled glowing tracer projectiles (reuse `DotPool`/glow texture pattern) flying from near the drone toward the aim point; 250–350 units/s; they despawn on contact.

2.5 **Combat layer** `CombatController`:
   - Raycast hit **enemy first** (later phases) → damage, spark FX (`ImpactFX`), score via `ScoreKeeper`, `ParticlePool` burst.
   - Raycast hit **building facade** → bullet sparks, `ScoreKeeper.subtract(FRIENDLY_BUILDING_PENALTY)`.
   - Raycast hit **car** (`CarRig` BVH/box approx.) → car pops (explosion + disable), `ScoreKeeper.subtract(FRIENDLY_CAR_PENALTY)`.
   - Enemies not yet present: bullets simply travel to max distance and despawn.

2.6 **Stun mechanic stub**: `CombatController.setStunned(seconds)` — disables firing, shows the HUD "CANNON STUNNED" flash; used by Dive Drones in Phase 3.

### Code sketches

[MODIFY] `index.html` — drone, aim, combat (inserted before `Game` class):

```js
class PlayerDrone {
  constructor(scene, radius, altitude) {
    // procedural jet mesh; engine glow sprites (additive)
    this.speed = 55;
    this.radius = radius; this.altitude = altitude;
    this.angle = Math.random() * Math.PI * 2;
    this.fwd = new THREE.Vector3(1, 0, 0);
  }
  update(dt, time) {
    this.angle += (this.speed * dt) / this.radius;
    const a = this.angle;
    this.position.set(Math.cos(a) * this.radius, this.altitude, Math.sin(a) * this.radius);
    // bank into the turn: lookAt center, then roll slightly by sin(a)
    this.roll = Math.sin(a) * 0.35;
  }
}

class AimController {
  constructor(crosshairEl) { this.crosshair = crosshairEl; this.ray = new THREE.Raycaster(); }
  onMove(x, y, ndcX, ndcY) { /* move crosshair element */ }
  setCamera(camera) { /** } // camera dependency injected at runtime
}

const SCORE_TABLE = Object.freeze({
  SCOUT: 100, BOMBER: 250, DIVE_DRONE: 150,
  BOMB_INTERCEPTED: 100, BOSS: 5000,
  FRIENDLY_BUILDING: -200, FRIENDLY_CAR: -50,
});
```

### Deliverables

- `index.html` modified: drone mesh + autopilot, chase cam, crosshair, bullets, combat raycast, friendly-fire penalties, stun stub.

### Verification

1. Start a run → drone takes off, circles the city smoothly; camera follows, no OrbitControls.
2. Hold mouse → tracer bullets fire at the crosshair; holding further keeps firing with cooldown.
3. Shoot a building → sparks, score drops by 200 (HUD updates). Shoot a car → car explodes, −50.
4. Console: no errors; `?demo=1` still works.

---

## Phase 3 — Enemies & Wave Director

**Objective:** Add Scout, Bomber, and Dive Drone enemies plus a `WaveDirector` that composes escalating waves. Enemies die to bullets; bombers harm the city (Phase 4 consumes that damage); dive drones stun the cannon.

### Tasks

3.1 Introduce an `Enemy` base class (SRP/OCP anchor): `{ hp, scoreValue, mesh, update(dt, time, world), damage(amount), onDestroyed() }` with a protected `onDestroyed` hook spawning `ParticlePool` explosion + `ImpactFX` flash + score.

3.2 `ScoutEnemy`: slow circling saucer (re-skin existing `ScoutShip` geometry); fires no projectiles in early waves, gains a slow pea-shooter at wave 4+ (damage negligible to player — purely cosmetic).

3.3 `BomberEnemy`: crosses the city at moderate altitude; on reaching a random building it drops a **bomb** (glowing falling sphere). Bombs are destructible projectiles (intercept = +100). Missed bombs detonate on impact with a building → calls `world.cityIntegrity.damageBuilding(buildingId, dmg)` (API is ready in Phase 4, stubbed in 3.x to log only).

3.4 `DiveDroneEnemy`: fast, waits until player is within ~120 units, then boosts toward the drone; on contact → `world.combat.setStunned(3)` + explosion of both drone-corner sparks (drone itself survives visually with flicker), enemy dies.

3.5 `WaveDirector`:
   - `buildWave(n)` returns a spawn list from a descriptor table with counts/HP/speed scaling: e.g. `WAVE_1: [4×SCOUT]`, `WAVE_2: [5×SCOUT, 2×BOMBER]`, `WAVE_3: [6×SCOUT, 3×BOMBER, 2×DIVE]`, … `WAVE_8: [12×SCOUT, 6×BOMBER, 5×DIVE]`, `WAVE_9: [BOSS]` (boss consumes Phase 5).
   - Spawns outside the fog ring and drifts in; wave considered active while any enemy remains; then `PLAYING -> WAVE_BANNER(next)`.
   - Scale factor S = 1 + (n−1) * 0.12 applied to hp and count.

3.6 An `EnemyRegistry` (simple array inside `Game`) collects enemies; `CombatController` raycast now tests enemy meshes first (Phase 2.5 step becomes live).

### Code sketches

[MODIFY] `index.html` — enemy base + director:

```js
class Enemy {
  constructor(scene, hp, scoreValue) { this.hp = hp; this.scoreValue = scoreValue; this.alive = true; }
  update(dt, time, world) { /* override */ }
  damage(amount) {
    this.hp -= amount;
    if (this.hp <= 0) { this.onDestroyed(); return true; }
    return false;
  }
  onDestroyed() { this.alive = false; /* FX + score hook */ }
}
```

### Deliverables

- `index.html` modified: base `Enemy`, 3 enemy classes, bombs as projectiles, `WaveDirector` with 9-wave table, registry + combat integration.

### Verification

1. Wave 1 spawns 4 scouts; bullet hits kill them: explosion FX + score +100 each.
2. A bomber drops a bomb that lands on a building → console log of future damage; bomb shot mid-air → +100.
3. A dive drone that reaches the drone → HUD "CANNON STUNNED", firing blocked ~3 s, then re-enables.
4. After last enemy of a wave dies, the wave banner announces the next wave.

---

## Phase 4 — City Integrity & loss condition

**Objective:** Give the city real HP. Buildings accumulate layered damage states (intact → windows dimmed → charred/leaning → rubble) instead of being instantly deleted; total damage feeds the HUD city-HP bar; 0 HP = GAME_OVER.

### Tasks

4.1 New `CityIntegrity` class owning the mapping `buildingId -> { hp, maxHp, state }`:
   - Each `CityGenerator` building registers an id + HP (e.g. 100 for small, 300 for landmarks, Eiffel/TV-tower 600).
   - `damageBuilding(id, amount)` decays state at thresholds 75/40/10 %.
   - State visuals: swap facade texture to a dim variant → darken material + slight random lean (rotate z/x by ≤ 5°) → shrink `scale.y` to 0.35 + charred color. Final state triggers the existing destruction/`ParticlePool` + `ImpactFX` collapse, with the building ending as a small rubble mound (never fully deleted — keeps the skyline readable).
   - Performance: state changes are one-shot mesh/material edits; leaning is applied once, not animated per-frame.

4.2 "Graveyard" reuse: re-purpose the existing graveyard block pools for rubble mounds.

4.3 `CityIntegrity.getHealth()` → 0..1 normalized; HUD bar reflects it. GAME_OVER when it hits 0 (slow-mo via reduced time scale + sky flash, then state screen — use existing alien-victory-state visuals as the frame).

4.4 Landmarks (Eiffel, Empire State, TV Tower) emit a one-time low alarm flash when they first enter charred state (audio comes in Phase 6; visual flash only for now).

4.5 Bomber bombs and (later) boss beams are the only damage sources; player bullets never damage buildings (they only cost points).

### Code sketches

[MODIFY] `index.html` — integrity facade:

```js
class CityIntegrity {
  constructor() { this.buildings = new Map(); this.health = 1; }
  register(id, maxHp) { this.buildings.set(id, { hp: maxHp, maxHp, state: 0 }); }
  damageBuilding(id, amount) {
    const b = this.buildings.get(id); if (!b) return;
    b.hp = Math.max(0, b.hp - amount);
    const next = b.hp / b.maxHp; // thresholds -> state 0..3 visuals
  }
  getHealth() {
    let s = 0, m = 0;
    for (const b of this.buildings.values()) { s += b.hp; m += b.maxHp; }
    return m > 0 ? s / m : 0;
  }
}
```

### Deliverables

- `index.html` modified: integrity model, damage-state visuals, HUD bar, rubble reuse, GAME_OVER trigger.

### Verification

1. Fire bombs (from Phase 3) at a building → window texture dims, then char/lean, then collapse into rubble; HP bar drops proportionally.
2. Reduce all buildings to rubble via console `cityIntegrity.damageBuilding(id, 99999)` loop → GAME_OVER screen appears with score.
3. Player bullets hitting buildings do NOT damage them (score penalty only — unchanged from Phase 2).

---

## Phase 5 — Boss: the Mothership

**Objective:** Turn the existing `Mothership` into a real boss fight: HP, three laser-barrage phases, destructible, victory screen on defeat.

### Tasks

5.1 Extend `Mothership` → `BossMothership extends Enemy`:
   - HP ~8000; HUD boss HP bar (distinct color) appears above city HP bar.
   - Phase A (66–100 %): slow singles — fires existing `BeamPool` beams at random buildings; each beam deals `CityIntegrity` damage scaled to tower class.
   - Phase B (33–66 %): double-barrage + first **drone swarm summon** (2× dive drones per summon, via `WaveDirector.spawnExtra('DIVE_DRONE', 2)`).
   - Phase C (<33%): triple beams + shroud of scouts; boss slowly descends toward the city.
   - Phase transitions trigger shockwave FX + alarm flash.
5.2 Anti-dodge check: boss beams must be dodgeable at range — beam travel time ~1.2 s with a telegraph: the target building flashes red 0.6 s before impact (reuse `ImpactFX` flash on the building).
5.3 On defeat: massive multi-ring `ImpactFX` + ship falls and detonates at ground level; `ScoreKeeper.add(5000)`; state → VICTORY; high score saved.
5.4 VICTORY screen: score, high score, "Defend Again" (→ MENU) and "Replay" (same city, wave 1).

### Code sketches

[MODIFY] `index.html` — boss wiring:

```js
class BossMothership extends Enemy {
  constructor(scene, cityRadius) {
    super(scene, 8000, SCORE_TABLE.BOSS);
    this.phase = 'A';
  }
  update(dt, time, world) {
    const hpFrac = this.hp / 8000;
    if (hpFrac < 0.33) this.phase = 'C';
    else if (hpFrac < 0.66) this.phase = 'B';
    // phase-driven barrage scheduling (telegraph flash -> BeamPool -> cityIntegrity.damageBuilding)
  }
  onDestroyed() { /* super.onDestroyed(); fall+detonate sequence; world.gameMachine.setState(GameState.VICTORY) */ }
}
```

### Deliverables

- `index.html` modified: boss class, HP bar, phases/telegraphs, victory flow.

### Verification

1. Wave 9 spawns the mothership with a boss HP bar; beams are telegraphed (red building flash) then hit buildings dealing damage.
2. Phase thresholds trigger transition FX; summons appear in B and C.
3. Reduce boss HP to 0 → detonation sequence → VICTORY screen with +5000 and high score saved.

---

## Phase 6 — Audio, final HUD & polish

**Objective:** WebAudio-synthesized SFX, audio-context unlock on first interaction, final HUD polish, and end-to-end balance sanity pass.

### Tasks

6.1 `AudioManager` class (`WebAudio`, no files):
   - `zap` (fire — short square blip ~880 Hz, 0.08 s), `explosion` (noise burst + low sine drop), `alarm` (two-tone saw wobble), `wave_fanfare` (rising arpeggio), `boss_hit` (metallic ping), `stun` (descending glissando), `victory`/`gameover` stingers.
   - Master gain, `AudioContext` created lazily on first pointer/key press (browser autoplay policy).
6.2 Wrap the HUD: score + wave + city HP + boss HP + stun flash get a shared update tick from `Game` (single DOM read per frame where cheap; only set text when changed).
6.3 Screens polish: menu title animation (GSAP float), wave banner slide-in + out, game-over/victory glass panels; consistent neon CSS palette (reuse existing accents).
6.4 Balance ping: tune `SCORE_TABLE`, wave table, boss HP, bomb damage from a quick playthrough; record any tweaks in `CONFIG` (keep all in `CONFIG` — single source of truth).
6.5 Remove the lil-gui script tag + logic (except `?demo=1` path).

### Code sketches

[MODIFY] `index.html` — audio manager:

```js
class AudioManager {
  constructor() { this.ctx = null; this.master = null; }
  unlock() {
    if (!this.ctx) {
      this.ctx = new (window.AudioContext || window.webkitAudioContext)();
      this.master = this.ctx.createGain(); this.master.gain.value = 0.4;
      this.master.connect(this.ctx.destination);
    }
    if (this.ctx.state === 'suspended') this.ctx.resume();
  }
  play(name) { /* switch on oscillator/noise recipes */ }
}
```

### Deliverables

- `index.html` modified: audio manager + recipes, HUD/tick polish, menu animations, balance pass.

### Verification

1. First click unlocks audio; every action (fire, kill, bomb, stun, fanfare, boss hit) produces distinct sound.
2. HUD updates beat-by-beat without flicker; wave banner animates in/out.
3. Full run (console-cheat if needed) hits both endings without console errors. Sanity: score can never go below 0.

---

## Phase 7 — Documentation & Implementation Report

**Objective:** Update user documentation and write the implementation report (mandated final phase).

### Tasks

7.1 [MODIFY] `README.md` — full rewrite: what the game is, controls (mouse aim, hold to fire, drone autopilot), how to run (`python3 -m http.server 8080` or double-click), wave/boss overview, score table incl. friendly-fire penalties, `?demo=1` hint.

7.2 [MODIFY] `CHANGELOG.md` — add a `0.2.0 — City Defender 3D` entry summarizing the game transformation (features, structure).

7.3 [NEW FILE] `_ai/backlog/reports/260811_2331__IMPLEMENTATION_REPORT__city-defender-3d.md` — per the report template in the frontmatter spec of this plan's instructions: Summary, Files Changed, Key Changes, Technical Decisions, Testing Notes, Usage Examples, Documentation Updates, Next Steps; status `completed` or `partial` with honest coverage.

7.4 [NEW FILE] no — final git commit is left to the user (do NOT commit unless asked).

### Report frontmatter template

[NEW FILE] `_ai/backlog/reports/260811_2331__IMPLEMENTATION_REPORT__city-defender-3d.md`:

```yaml
---
filename: "_ai/backlog/reports/260811_2331__IMPLEMENTATION_REPORT__city-defender-3d.md"
title: "Report: City Defender 3D — arcade wave-survival game"
createdAt: YYYY-MM-DD HH:mm
createdBy: "opencode [deepseek-v4-flash-free]"
updatedAt: YYYY-MM-DD HH:mm
updatedBy: "opencode [deepseek-v4-flash-free]"
planFile: "_ai/backlog/active/260811_2331__IMPLEMENTATION_PLAN__city-defender-3d.md"
project: "ds-v4-flash"
status: completed
filesCreated: 2
filesModified: 3
filesDeleted: 0
tags: [threejs, webgl, game, arcade, single-file, javascript]
documentType: IMPLEMENTATION_REPORT
---
```

### Deliverables

- `README.md` rewritten, `CHANGELOG.md` updated, `_ai/backlog/reports/260811_2331__IMPLEMENTATION_REPORT__city-defender-3d.md` written.

### Verification

1. `README.md` renders correctly on GitHub/gitlab markdown; commands in it work from a fresh clone.
2. Report file exists at the exact path above with all required sections and honest status.
3. Play a full game: menu → city → 9 waves → boss → VICTORY; and one loss run → GAME_OVER; both with zero console errors.

---

## Appendix A — Global Verification Checklist (run after each phase)

- [ ] Zero console errors at all states (MENU, banner, play, game over, victory).
- [ ] 60 fps on a mid-range machine (check `renderer.info` draw calls stay reasonable; pools capped).
- [ ] No memory leak when switching from defeat → new run (buildings re-generated; pools reused).
- [ ] `?demo=1` legacy demo still operational until final cleanup decision.
- [ ] Score never negative; high score persists across reloads.

## Appendix B — Out of scope (YAGNI)

- Multiplayer / online leaderboards
- Shop or upgrade systems (pure arcade by design)
- Mobile touch controls / gamepad
- Audio files or external assets (all procedural)
- Separate modules or build tooling (single-file constraint)