# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

Open `Solar app/solar_iteration_3_.html` via a local HTTP server — **not** directly from the filesystem. Three.js loads textures via XHR, which fails under `file://` due to browser CORS restrictions.

```bash
# From the project root
python3 -m http.server 8000
# Then open http://localhost:8000/Solar%20app/solar_iteration_3_.html
```

No build step, no package manager, no dependencies to install. Three.js r128 and OrbitControls are loaded from CDN.

## File Layout

All code lives in `Solar app/`. Each iteration is a self-contained single HTML file with inline CSS and JS:

| File | What's in it |
|------|-------------|
| `solar_iteration_1_.html` | Basic version: procedural planet colors, no textures on most bodies, simple speed slider |
| `solar_iteration_2_.html` | Adds planet textures, orbit toggle, bodies list panel, speed label |
| `solar_iteration_3_.html` | **Current / full version** — see below |

Texture images (`.jpg`/`.png`) sit alongside the HTML and are loaded by relative path.

## Architecture of `solar_iteration_3_.html`

Everything runs inside a single `<script>` block. There is no module system.

**Scene construction (top of script)**
- `THREE.WebGLRenderer` → `THREE.PerspectiveCamera` → `OrbitControls`
- Camera `up` vector is tilted 23.4° to match Earth's equatorial frame
- Milky Way skybox: `SphereGeometry(500)` with `BackSide` rendering
- Sun: `MeshBasicMaterial` + separate transparent glow mesh
- Planet instances are built from the `data` array (each entry has `name`, `dist`, `size`, `color`, `speed`, `info` HTML string, `texture`)
- Moons created via `createMoon(size, distance, speed, color, infoText, texture, startAngle)` — returns a mesh parented to its planet's orbit pivot

**Three mutually exclusive view modes**
1. **Solar System** (default) — planets orbit the Sun; camera uses OrbitControls
2. **Galactic View** (`galacticViewActive = true`) — `galacticGroup` THREE.Group becomes visible; sun and earth markers orbit a galactic centre at 60.2° ecliptic tilt; independent log-scale speed slider
3. **Spaceship Earth** (`spaceshipViewActive = true`) — fixed camera locked to Earth; five velocity arrows shown; years counter

Switching views calls `enterGalacticView()` / `exitSpaceshipView()` etc. The two non-default views are independent and fully exited before entering the other.

**Animation loop**
`animate()` called via `requestAnimationFrame`. Each frame:
- Advances each planet's `orbitAngle` by `speed × elapsed × simulationMultiplier`
- Updates moon angles relative to their parent planet
- If galactic view: advances `galacticAngle`, updates sun/earth trail buffers
- If spaceship view: updates `seYearDisplay` from elapsed ms

**Interaction**
- `THREE.Raycaster` on canvas `click` — hits checked against `allClickable` (planets + sun) and `jupiterMoonMeshes`
- On hit: `flyToObject(obj)` animates camera to the target over ~60 frames using `requestAnimationFrame` recursion
- Info panel populated from `mesh.userData.info` (raw HTML string)

**Speed control**
Slider value is on a log₁₀ scale: `simulationMultiplier = Math.pow(10, sliderValue)`. Negative slider values produce sub-realtime; positive values produce super-realtime. `getSpeedLabel()` formats the display.

**Simulation time**
`simElapsedMs` accumulates each frame. `updateSimTimeDisplay()` converts to a calendar date starting from a configurable epoch.

## Git Workflow

Repository: `https://github.com/psittcode/ClaudeCodeTest` (branch `main`)

**Commit and push after every meaningful unit of work.** Do not batch changes across multiple features or fixes into one commit. The goal is that the GitHub history always reflects the current working state so any version can be recovered.

```bash
git add "Solar app/<changed-file>"
git commit -m "<concise description of what changed and why>"
git push origin main
```

Good commit message examples:
- `Add Saturn ring opacity control to UI panel`
- `Fix Jupiter moon orbit speed to match real ratios`
- `Increase skybox radius to eliminate clipping at galactic view zoom`

A Stop hook auto-saves any uncommitted changes with a timestamped message as a safety net, but every intentional change must get a proper descriptive commit — not just the auto-save fallback.
