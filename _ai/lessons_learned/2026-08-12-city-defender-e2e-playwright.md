# [2026-08-12] - City Defender 3D: E2E Verification of a Three.js Game

## Context
Built a complete single-file 3D game (City Defender 3D, `ds-v4-flash/index.html`, ~4500 lines: Three.js + GSAP + vanilla JS game loop) and verified it end-to-end with a Playwright smoke test that plays through both endings (game over + victory), the boss fight, score persistence, stun, friendly-fire penalties, and the legacy demo mode.

## Challenge
- No Playwright-managed browser binary available; `npx playwright install` failed with `apt-get: command not found` (Arch/Manjaro system, not Debian).
- Headless software-GL (SwiftShader) rendered frames so slowly that fixed-sleep test assertions failed intermittently.
- Multiple real bugs surfaced only through the automated test: a `.clone()` crash on fire, stun never working (`NaN`), bullets never killing enemies, console warnings.
- Debugging instrumentation patches to the test file kept getting mangled by repeated sed/python patch attempts.

## Discovery/Solution
- **Find an existing browser with `ps` instead of installing one.** `ps aux | rg -i "chrom|firefox"` revealed system Chromium at `/usr/lib/chromium/chromium`. Point Playwright at it via `chromium.launch({ executablePath: '/usr/lib/chromium/chromium', args: ['--use-angle=swiftshader','--enable-unsafe-swiftshader','--no-sandbox'] })`. No download, no apt.
- **`waitUntil: 'load'` hangs in slow headless** — module scripts + CDN fonts block `load`/`domcontentloaded` for 30-60s. Use `waitUntil: 'domcontentloaded'` with `timeout: 120000`.
- **Poll for game state instead of sleeping.** gsap/JS timers run on wall clock but the game's own `dt` is clamped (0.05), so world-time vs wall-time drift wildly under slow FPS. A `waitForState(page, state, timeout)` polling loop (every 250ms) made all state-transition assertions robust.
- **Expose a cheat API + `THREE` on `window.app`** for deterministic tests (fire/boss/killBoss/damageAll/score/state). Made the boss fight, game-over, and victory paths reachable in seconds instead of playing minutes.
- **Aim at 3D targets in tests by projecting to screen:** `pos.project(camera)` → NDC → pixel coords → `page.mouse.move(x, y)`. Skip targets behind the camera (`p.z > 1 || p.z < -1`). This exercises the real AimController/mousemove path.

## Key Takeaways
- **Three.js gotcha:** a `THREE.Raycaster` holds its direction at `ray.ray.direction` (it wraps a `THREE.Ray`), not `ray.direction` — this caused a "Cannot read properties of undefined (reading 'clone')" crash in `fireShot`.
- **Mirror game time explicitly.** `Game` had `this.time`, but `CombatController.isStunned()` read `this.world.time` — never initialized → `Math.max(undefined, x)` = `NaN` → stun silently dead. Fix: `world.time = 0` in init and `world.time = this.time` every `Game.update()`. A debug probe asserting `world.time` is a number catches this instantly.
- **Muzzle parallax misses small hulls.** Aiming bullets down the camera ray from a muzzle offset (8-12 units) misses ~1-unit-wide targets at 150-240u range. Fix: raycast the crosshair ray against enemies/bombs and aim at the actual hit point (fallback to a 600u point) — this also gives "shoot what you see" feel.
- **Never splice while iterating pooled arrays:** `for (const b of this.active) { ...this.release(b) }` skips entries. Always iterate `this.active.slice()`.
- **`MeshBasicMaterial` ignores `emissive`/`emissiveIntensity`** — produces `THREE.Material: 'emissive' is not a property of THREE.MeshBasicMaterial` warnings. Use plain `color` or switch to `MeshStandardMaterial`.
- **Headless FPS ≠ real FPS:** banner timing, stun expiry, wave clear all took 2-3x longer in SwiftShader headless than design values; never assert "should be done after X ms" — poll with generous timeouts.
- **Write dynamic assertions, not hardcoded ones.** Expected "high score = 5000" broke legitimately once kills started scoring (400). Read current state (`score.score`) first and compute the expected value from it.
- **When a test file gets mangled by patch attempts, rewrite the whole file** — patching patched code compounds errors; a fresh Write is cheaper than debugging doubled string replacements.
- **When a bug doesn't reproduce in isolation but fires in the full run**, add step markers (`PAGEERROR at [step]`) and re-run the full flow — the crash site and moment become obvious immediately.
