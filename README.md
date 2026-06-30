# 🛸 Dashlander

### ▶️ [**Play the Live Demo**](https://rencheng0525.github.io/Dashlander/)

A **spherical lunar lander** in a single, dependency-free HTML file — rendered in glowing monochrome **vector-tube** graphics. Fly in on an orbital approach, pitch back to brake, and set down softly on a landing pad. Or hit one button and watch the autopilot do it for you.

> Pure HTML/CSS/JS on a `<canvas>`. No frameworks, no build step, no dependencies, no network calls. One file. Open it and play.

---

## ✨ Highlights

- **Round world.** Gravity always pulls toward the centre of a circular moon. The camera rotates so the surface stays beneath you and zooms out as you climb.
- **Procedural & deterministic.** Every level is generated from a seed — the same seed always produces the same terrain and landing pads.
- **One-button autopilot (~99–100%).** Picks the safest pad, flies there, and lands gently. Toggle any time.
- **Auto Demo mode.** A hands-off showcase: dramatic orbital entry with the moon sweeping past, then a clean autoland, level after level.
- **Bilingual.** English and Traditional Chinese (繁體中文), switchable live. English by default.
- **Time control.** Slow to 0.25× to savour a touchdown, or fast-forward to 4×.
- **Vector / CRT aesthetic.** Phosphor-green line art, beam glow and persistence trails, scanlines and vignette.
- **Desktop & mobile.** Keyboard, mouse, and on-screen touch controls.

---

## 🚀 Getting started

No installation, no toolchain.

```bash
# Clone, then just open the file
git clone https://github.com/your-username/dashlander.git
cd dashlander
open index.html           # macOS  (use: xdg-open on Linux, start on Windows)
```

Or simply double-click `index.html`. It also works hosted on any static site (GitHub Pages, Netlify, etc.) — drop the single file in and you're done.

---

## 🎮 Controls

| Action | Keyboard | Touch |
| --- | --- | --- |
| Main thruster | `↑` / `W` / `Space` | thrust button |
| Rotate left / right | `←` `→` / `A` `D` | rotate buttons |
| Toggle autopilot | `P` | **Auto-Land** button |
| Cycle game speed | `,` slower · `.` faster | speed button (bottom-right) |
| Restart level | `R` | — |
| Back to menu | `Esc` | — |

The menu offers **Start Mission**, **Random Level**, and **Auto Demo**.

---

## 🌑 How to play

You begin each mission with an **approach**: the lander flies in horizontally and bleeds off speed on its own. When the approach ends, control is handed to you (or the autopilot).

Bring the lander down onto a glowing **landing pad**. A touchdown only counts as a landing if **all** of these are true on contact:

- **Tilt** ≤ 20° relative to the pad
- **Vertical speed** ≤ 3.0 m/s
- **Horizontal speed** ≤ 2.0 m/s
- You're actually **on a pad**

Miss any of those — or hit the terrain — and the lander is lost.

### Scoring

Your score rewards a clean, efficient landing:

- **Fuel** remaining
- **Velocity** — softer is better
- **Attitude** — the more upright, the better

The total is multiplied by a **pad difficulty multiplier** (0.5×–2.0×). Pads that are farther away or narrower are worth more.

---

## 🤖 Autopilot & Auto Demo

### Auto-Land
Tap the **Auto-Land** button (or press `P`) at any point during flight. The controller takes over from the current state, so you can fly manually and hand off whenever you like. It lands successfully on roughly **99–100%** of procedurally generated levels, typically with picture-perfect touchdowns.

How it works, briefly:

1. **Choose the safest pad** — flattest, widest, and reasonably near — rather than just the closest.
2. **Cruise** above the terrain and traverse to the pad's centre using a time-optimal "arrive" profile.
3. **Commit & descend** straight down onto the pad surface once stable overhead, going fully upright for the final metres.

Under the hood it turns the lander's binary (on/off) thruster into effective proportional thrust via **PWM duty-cycling**, and low-pass-filters the attitude command so the craft holds a steady cruise tilt instead of hunting.

### Auto Demo
Pick **Auto Demo** from the menu for a continuous, hands-off showreel. It opens with a cinematic high-speed **orbital approach** — the whole moon sweeps past beneath you and gradually slows — then the autopilot lands and automatically advances to the next level. On mobile, manual controls are hidden for a clean view; the speed control stays. Tap the **DEMO ✕ STOP** badge to exit.

---

## ⏱️ Time control

A single button (bottom-right) cycles through **0.25× · 0.5× · 1× · 2× · 4×**; on desktop, `,` and `.` step through speeds too. Because the simulation always advances in a fixed time step, changing speed only changes playback — it never changes the outcome of a landing.

---

## 🟢 Visual style

Dashlander is drawn like a classic **vector display**:

- Monochrome **phosphor green**, line art only — no filled sprites
- Glowing strokes with a bright core, plus **persistence trails** as things move
- **CRT overlay**: scanlines, corner vignette, and a subtle flicker
- Monospace, vector-style type

---

## 🌐 Internationalisation

All on-screen text lives in a small dictionary with **English** and **Traditional Chinese** entries. English is the default; a toggle in the corner switches languages live, including the results screen.

---

## 🧠 Technical notes

- **Single file.** Everything — markup, styles, and logic — is in `dashlander.html`.
- **Pure-vs-DOM split.** Level generation, ship state, the physics step, collision/landing checks, scoring, and the autopilot are written as **pure, DOM-free functions**, so they can be unit-tested headlessly (e.g. with `node`). Everything that touches the canvas, input, or the DOM lives in the "game shell" below.
- **Fixed-step integration.** Physics runs at a fixed **60 Hz** using semi-implicit Euler with an accumulator, independent of frame rate and of the time-scale setting.
- **Determinism.** Levels come from a seeded `mulberry32` PRNG; the same seed reproduces the same world exactly.
- **Coordinate conventions.** Up is `-Y`; angle `0` points radially outward; gravity points toward the origin. Touchdown is evaluated against each segment's surface normal.
- **No browser storage.** State is kept in memory only, so it runs anywhere — including sandboxed embeds.

---

## 📁 Project structure

```
dashlander/
├── index.html        # the entire game (open this)
├── CLAUDE.md         # architecture & contributor guidance for AI coding agents
└── README.md         # you are here
```

`CLAUDE.md` documents the physics-parameter contract, sign conventions, the pure/DOM boundary, determinism rules, and house rules — useful for both human and AI contributors.

---

## 🛠️ Development

There's no build to run — edit `index.html` and refresh the browser.

A couple of conventions worth keeping:

- Keep the **pure/DOM split** so the simulation stays testable.
- Treat the **physics parameters** and **touchdown limits** as a contract; changing them changes the game's feel, so do it deliberately.
- Add visible strings to **both** languages.

To sanity-check the logic without a browser, you can extract the pure functions and drive them from Node — for example, simulating the autopilot across a range of seeds and asserting the landing rate.

---

## 📜 License

Released under the **MIT License**. See `LICENSE` for details.

---

<p align="center">🌖 Happy landings.</p>
