# CLAUDE.md

Guidance for AI coding agents (and humans) working in this repository. Read this
before editing. It captures the architecture, the physics contract, and the
invariants that must not drift.

---

## 1. What this is

**Dashlander** is a single-file, dependency-free arcade game: a *lunar lander on
a spherical world*. The moon is a circle centred at the world origin `(0, 0)`.
Gravity always points toward that centre, so "down" depends on where the ship
is. The camera rotates so the surface always appears beneath the ship, and zooms
out as the ship climbs.

The entire game — markup, styling, and logic — lives in **`index.html`**. There
is no build step, no bundler, and no runtime dependency. Open the file in any
modern browser to run it. (The file is named `index.html` so it can be hosted as
the root page of a static site such as GitHub Pages.)

The visual style is a **vector display / CRT** look: monochrome phosphor green,
glowing line art (no filled sprites), beam persistence trails, scanlines and a
corner vignette.

---

## 2. How to run & test

- **Run:** open `index.html` in a browser. No server needed.
- **Syntax check:** extract the `<script>` block and run `node --check` on it.
- **Headless logic test:** the pure functions (`generateLevel`, `makeShip`,
  `physicsStep`, `checkCollision`, `validateLanding`, `computeScore`,
  `padCenter`, `autopilot`, `approachControl`) have no DOM dependencies. Extract
  everything above the `game shell` banner comment and `eval` it in Node to
  simulate flights and assert outcomes (e.g. free-fall must crash; the autopilot
  must land across a range of seeds). Note that `autopilot` and `approachControl`
  use a module-level `_pwm` accumulator — reset it (`_pwm = 0`) at the start of
  each simulated flight.

There is no test runner checked in. If you add one, keep the pure-vs-DOM split
below so logic stays unit-testable.

---

## 3. Code layout (within the single file)

The `<script>` is organised top-to-bottom into clearly bannered sections:

1. **i18n** — `I18N` dictionary, `LANG`, `T(key)`, `applyI18n()`.
2. **constants** — `C` (physics/world), `LIMIT` (touchdown), `SCORE_MAX`.
3. **helpers** — `clamp`, `easeIn`, `len`, `makeRNG`.
4. **level generation** — `generateLevel(seed)`.
5. **ship state** — `makeShip(level)`.
6. **physics step** — `physicsStep(s, dt)`, angle helpers.
7. **collision & landing** — `checkCollision`, `validateLanding`.
8. **scoring** — `computeScore`.
9. **particles** and **starfield**.
10. **auto-land controller** — `padCenter(level, phiS)`, `autopilot(s, level)`,
    and the module-level `_pwm` accumulator.
11. **game shell** (everything below the big banner) — canvas setup, input,
    flow control (`STATE`, `showMenu`/`startGame`/`endGame`), the approach
    controller (`approachControl`), the fixed-step loop (`fixedUpdate`, `loop`),
    rendering, and the HUD.

**Convention:** sections 1–10 are *pure / DOM-free*. The "game shell" is the only
place that touches `document`, `canvas`, `window`, or globals like `gameState`.
Keep this boundary. Do not call rendering or read the DOM from inside physics or
the controllers. (`approachControl` lives in the shell for proximity to the loop,
but is itself pure — it only reads/writes the ship object.)

---

## 4. The physics contract (do NOT silently change these)

These values define the game's feel. Treat them as a fixed contract; change them
only on explicit request, never as an incidental side effect of a refactor.

### World & environment
| Constant | Value | Meaning |
|---|---|---|
| `lunarGravity` | `3.2` | m/s² toward the moon centre |
| `standardGravity` | `9.80665` | reference g for fuel flow & G-force |
| `moonRadius` | `1000` | base world radius |
| `terrainSegments` | `80` | line segments around the circle |
| `maxTerrainHeight` | `300` | max hill/crater height over the radius |
| `noiseFrequency` | `4.5` | sine-wave hill frequency |
| `numLandingPads` | `3` | base pad count |
| `padWidthSegments` | `2` | pad width in segments |

### Ship
| Constant | Value |
|---|---|
| `dryMass` | `4280` kg |
| `engineMaxThrust` | `45040 * 1.8` N |
| `specificImpulse` | `311` s |
| `baseInertia` | `50000` |
| `rcsSteeringTorque` | `25000` |
| `rcsLeverArm` | `10` |
| `defaultMaxFuel` | `1000` kg |
| `shipRadius` | `14` |

### Boundaries, camera, spawn, scale
| Constant | Value |
|---|---|
| `deepSpaceBoundary` | `2000` |
| `crustBoundary` | `1000` |
| `cameraZoomSurfaceRatio` | `0.8` |
| `initialVelocityX` | `28.0` (see note) |
| `startAltitude` | `600` |
| `shipVisualHeight` / `shipRealHeightMeters` | `34` / `3` |

`pixelsPerMeter = shipVisualHeight / shipRealHeightMeters` (~`11.333`). All
real-world m/s and m/s² values are scaled by this factor before integration, so
the simulation and the rendered world stay in lock-step. **If you touch the
visual ship size, you change `pixelsPerMeter` and therefore the whole feel** — do
it deliberately.

> **Note on `initialVelocityX`:** the constant feeds `generateLevel`, but the
> actual horizontal entry speed is **overridden per-mode in `startGame`** (see
> §7): ~48 m/s for the demo's dramatic sweep, ~6 m/s for a gentle human opening.
> The ship spawns nose-up (`angle = 0`).

Each constant also has a matching `*Var` (variance) field consumed by
`generateLevel` via the `variance(base, v, randomFactor)` helper. Keep base and
variance together when editing.

### Touchdown limits (`LIMIT`)
A touchdown counts as a landing only if **all** of:
- tilt ≤ `20°` (relative to the pad's surface normal)
- vertical (radial) speed ≤ `3.0 m/s`
- horizontal (tangential) speed ≤ `2.0 m/s`
- contact segment is a flagged landing pad

Otherwise it is a crash, and the failing condition becomes the crash reason.

### Scoring (`SCORE_MAX`, `computeScore`)
- Fuel: `easeIn(fuel/maxFuel) * 2500`
- Velocity: curved score over a `2500..5000` band, full at 0 impact speed
- Tilt: curved score up to `2500`, full at 0 tilt
- Sum (floored at 0) × pad difficulty multiplier = final score
- Pad multiplier ranges `0.5..2.0` by angular distance from spawn, adjusted by
  pad width (narrower = harder = bonus; wider = easier = penalty)

`easeIn` is `t*t` — an intentional approximation. If you swap it, do so for both
the positive and negative branches in `computeScore`.

---

## 5. Coordinate & sign conventions (easy to get wrong)

- **Up is `-Y`.** Angle `0` means the ship points straight up (toward `-Y`).
  Positive angle rotates clockwise.
- Forward/thrust vector for a given `angle` is `(sin(angle), -cos(angle))`.
- Gravity direction is `-position.normalized()` (toward the origin).
- Surface **normal** of a segment `(a → b)` is `(-dy, dx)` normalised, then
  flipped if it points inward (`normal · midpoint < 0`) so it always points
  outward.
- Surface **tangent** is the normal rotated 90°: `(-ny, nx)` (positive =
  clockwise around the moon).
- Radial ("vertical") speed = `velocity · normal` (positive = moving away/up);
  tangential ("horizontal") speed = `velocity · tangent`.
- `relTiltDeg(px, py, absDeg)` returns the ship's tilt relative to the spherical
  normal beneath it, in `[-180, 180]`. The HUD shows its absolute value.

Integration is **semi-implicit Euler** at a **fixed 60 Hz** step (`FIXED = 1/60`)
with an accumulator. Keep the fixed-step loop; do not move physics into the
variable-rate `requestAnimationFrame` delta, or determinism and feel will drift.

---

## 6. Determinism & level generation

Levels are produced by `generateLevel(seed)` using a seeded `mulberry32` PRNG.
The same seed must always yield the same level. Rules:

- Pull random numbers from `rng()` in a **stable order**. Inserting a new
  `rng()` call mid-sequence reshuffles every downstream value and breaks all
  existing seeds — append new draws at the end of the relevant block instead.
- Pads are flattened into straight secants and tagged in `isPad[]`, with a
  per-segment `padMult[]`. **Single-segment pads (`padWidth === 1`) are NOT
  flattened** — they keep the underlying terrain slope and can therefore sit on
  steep ground. The autopilot accounts for this (see §8).
- Terrain is a closed polygon (`points`, last point == first) for seamless
  looping and collision.

---

## 7. Flow, approach intro, demo & time control (game shell)

### States
`STATE.MENU / PLAYING / OVER`, tracked by `gameState`. `startGame(seed)` builds a
level and ship; `endGame()` shows the result; `showMenu()` resets everything.

### Approach intro (`approaching`, `approachControl`)
Every mission opens with a scripted **approach**: for `apTime` seconds the ship
flies in horizontally and decelerates while **player input and the autopilot are
locked out** (a blinking "APPROACH" badge shows). `approachControl` handles
altitude-hold and a cosmetic pitch-back flare; the actual slowdown is a
**scripted tangential-velocity decay** applied in `fixedUpdate` (the approach is
a cutscene, so a small physics liberty is acceptable and far more reliable than
trying to brake a high entry speed with the slow RCS). When `approachT` reaches
`apTime`, control is released.

Profiles are set per-mode in `startGame`:

| | `apTime` | `apMaxTilt` | `apDecay` | entry speed |
|---|---|---|---|---|
| **Demo** | `3.2` | `1.05` | `0.16` | ~48 m/s (dramatic moon-sweep) |
| **Human** | `1.4` | `0.5` | `0.12` | ~6 m/s (gentle settle) |

`apDecay` is the fraction of tangential speed remaining at handoff (decayed
exponentially over the window).

### Demo mode (`demoMode`)
Started from the menu. `autopilotOn` is forced true, the dramatic approach runs,
and after each result the loop auto-advances to the next seed (`demoTimer`).
While in demo, `document.body` gets the `demo-on` class, which **hides the manual
touch controls and the Auto-Land toggle** (the speed control and a tappable
"DEMO ✕ STOP" badge remain). Tapping the badge calls `showMenu()`.

### Time control (`timeScale`)
A cycling button (and `,` / `.` keys) selects `0.25× / 0.5× / 1× / 2× / 4×`. The
loop scales the accumulator and the visual scene by `timeScale`, with a per-frame
step cap (240) and backlog reset to avoid a spiral of death. **Because physics
always advances in fixed `FIXED` steps, time-scale only changes playback speed —
it never changes the outcome of a landing.** Keep UI timers (`demoTimer`) on real
(unscaled) time.

---

## 8. Auto-land controller (`padCenter`, `autopilot`)

One-button autopilot, toggled by the **Auto-Land** button or `P`, usable at any
time (it re-plans from the current state). It lands on ~99–100% of generated
levels. Structure:

1. **`padCenter(level, phiS)` — pick the *safest* pad, not the nearest.** It
   scores every pad by surface flatness (heavily weighted), angular distance, and
   width, and returns the chosen pad's centre **angle and altitude**. This is the
   single most important robustness feature: it avoids the unfair, un-flattened
   single-segment slope pads (§6). Results are cached on `level._pads`.
2. **Cruise & traverse.** Climb above the terrain and move toward the pad centre
   using a time-optimal "arrive" velocity profile.
3. **Commit & descend.** Once stable over the pad, descend using **height above
   the pad surface** (not above the base radius — pads can sit on hills), going
   fully upright for the final few metres.

Two techniques make it work with this vehicle:

- **PWM throttle.** The engine is binary (full on/off) at ~4.8 g. The controller
  computes a desired average acceleration and turns it into a duty cycle via the
  `_pwm` accumulator, yielding effective proportional thrust.
- **Attitude low-pass.** The tilt command is smoothed (`s._tilt`) so the craft
  holds a steady cruise tilt instead of hunting against the slow RCS.

Per-ship controller scratch state: `s._tilt`, `s._stable`, `s._desc` (reset when
the autopilot is (re)engaged). Module-level `_pwm` is shared and reset in
`startGame`.

---

## 9. Rendering — vector / CRT look

- **Monochrome phosphor green.** Primary line `#46ff85`, bright cores
  `#cfffe0`/`#daffe9`, tube black `#000600`. The CSS palette variables
  (`--magenta` etc., kept for naming continuity) are all set to green tones; UI
  is monospace with a green text glow.
- **Glowing strokes.** Each shape is stroked with `shadowBlur` glow plus a thin
  bright core pass. Nothing is filled with colour — interiors are tube-black so
  they occlude the starfield.
- **Beam persistence.** `render()` does **not** hard-clear; it paints
  `rgba(0,5,0,0.55)` over the canvas each frame, leaving short phosphor trails on
  moving objects (and a nice smear during the demo sweep). Raise the alpha for
  shorter trails, lower it for longer ones.
- **CRT overlay.** A pointer-events-none `#crt` element adds scanlines, a corner
  vignette, and a subtle flicker over everything.
- **Landing pads** are drawn brighter/thicker than the surface, with short legs
  pointing into the ground so they read as platforms.
- World→screen transform (`worldTransform`): translate to screen centre → shake →
  `scale(zoom)` → `rotate(-camAngle)` → `translate(-ship)`. `camAngle =
  atan2(ship.x, -ship.y)`; zoom is `1.0` until altitude exceeds
  `cameraZoomSurfaceRatio * (H/2)`, then scales to keep the surface on screen.
  Stroke widths that should look constant on screen are divided by `camZoom`.

---

## 10. Internationalisation

- All user-facing strings live in `I18N.en` and `I18N.zh`. **Default is `en`.**
- Static markup uses `data-i18n="key"`; `applyI18n()` fills it via `innerHTML`
  (some strings contain `<kbd>`/`<b>`). Dynamic text (crash reasons, HUD target
  arrow, result screen, badges) is read at render time through `T(key)`.
- When adding any visible text, add the key to **both** languages. Never hardcode
  a display string outside `I18N`. The language toggle re-runs `applyI18n()` live.

---

## 11. House rules for agents

- **Self-contained.** No external libraries, fonts, CDNs, or network calls. No
  build tooling. Everything ships in `index.html`.
- **No browser storage.** Do not use `localStorage`/`sessionStorage`; keep state
  in JS variables. (This also keeps it runnable inside sandboxed embeds.)
- **Preserve the physics contract** (§4) and **sign conventions** (§5) unless the
  task explicitly asks to change them. Call out any feel change in your summary.
- **Keep the pure/DOM split** (§3) so logic stays testable, and keep the
  **fixed-step** loop so time-scale and determinism hold.
- **Two opening profiles.** Human play must stay gentle and controllable; the
  dramatic high-speed sweep is for demo mode only (it's autopilot-flown).
- **Comments and identifiers stay neutral and self-descriptive** — describe what
  the code does, not where it came from.
- Prefer small, surgical edits. When adding a feature, add a new bannered section
  rather than threading it through existing functions.
