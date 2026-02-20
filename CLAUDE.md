# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm install         # Install dependencies (requires Node ≤12.18.3, pnpm ≥8)
pnpm run start       # Dev server at localhost:3000 with HMR
pnpm run build       # Production webpack build → build/
pnpm run lint        # Lint with semistandard
pnpm run lint:fix    # Auto-fix lint issues
```

No test suite exists — this project relies on manual QA.

## Architecture

Moon Rider is a **WebXR music game** (Beat Saber-style) built on A-Frame (Three.js WebXR wrapper). Players hit notes synchronized to music loaded from the BeatSaver API.

### Entry Points

- `index.html` — HTML shell, loads A-Frame from CDN + bundles
- `src/index.js` — Registers all A-Frame components, imports scene template
- `src/scene.html` — A-Frame scene definition (entities, component bindings)
- `webpack.config.js` — Two entry points: `src/index.js` → `build/build.js`, `src/workers/zip.js` → `build/zip.js`

### State Management

`src/state/index.js` — Single global state via `aframe-state-component` (`AFRAME.registerState`). Components bind to state via `bind__` attributes in scene.html. Mutations happen via named handlers.

### Components (`src/components/`)

97 A-Frame components, each registered via `AFRAME.registerComponent`. Key ones:

- `beat.js` / `beat-generator.js` — Note objects and spawning from beatmap data
- `blade.js` / `controller.js` — Player weapon and VR controller input
- `song.js` — Audio playback and synchronization
- `search.js` — BeatSaver API integration for song lookup
- `beat-cut-fx.js` — Slash/hit particle effects

### Templates (`src/templates/`)

Nunjucks HTML templates for UI screens (menu, search results, leaderboard, victory screen, etc.), loaded via `super-nunjucks-loader`.

### Beatmap Loading

Songs are fetched as zip files from BeatSaver, decompressed in a Web Worker (`src/workers/zip.js`), then parsed by `src/lib/convert-beatmap.js` which normalizes v2/v3 Beat Saber map formats.

### Key Libraries

- **A-Frame** (vendored in `/vendor/` + CDN) — 3D/XR framework
- **aframe-state-component** — Reactive state management
- **Firebase** — Backend (leaderboards)
- **semistandard** — Linting (Standard JS + semicolons required)

### Coding Conventions

- Semicolons required (semistandard)
- A-Frame component pattern: `AFRAME.registerComponent('name', { schema: {}, init() {}, ... })`
- Assets declared in `src/assets.html`, referenced by A-Frame asset system
- GLSL shaders inline or as `.glsl` files loaded via webpack-glsl-loader
