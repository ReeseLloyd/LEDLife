# LEDLife Project

## Project Location
`~/Dropbox/05-Projects/Programming/LEDLife/`

## What This Is
LEDLife is a single-file, self-contained HTML application — a browser-based artificial life simulator with an LED dot-matrix aesthetic. Organisms are mobile agents on a fixed grid; each living cell is a glowing LED dot whose color encodes its genome-derived phenotype. The simulation runs an approximately energy-conserved economy with sexual and asexual reproduction, predation, defense, evolvable mutation rates, weather cycles, pathogens, and environmental catastrophes.

No server, no build step, no dependencies, no installation required. Designed to run on desktop browsers and mobile (iPad/iPhone).

---

## Repo Structure
```
ledlife.html     ← the application (always the current version)
README.md        ← documentation (always current, displayed on GitHub)
CLAUDE.md        ← this file
.gitignore
_archive/        ← old versioned files for reference (gitignored)
```

**Always read both `ledlife.html` and `README.md` before making any changes.**

There are no versioned files (e.g. `ledlife-v0_16.html`). Git history is the version history.

---

## Versioning Rules
- **Every change — including minor bug fixes — gets a new version number**
- Increment the minor version for all changes (e.g., `0.15` → `0.16`)
- Increment the major version only for significant structural rewrites
- Update the `<title>` tag in `ledlife.html` to reflect the new version
- Add an entry to the version history table in `README.md` describing what changed
- Version history entries should be concise (one line), e.g.:
  `| **v0.16** | Fixed organism inspector not updating when paused |`
- For milestone versions, create a git tag: `git tag v0.16 && git push --tags`

---

## Tech Stack Constraints
- **Vanilla HTML, CSS, and JavaScript only** — no frameworks, no libraries, no CDNs
- Everything must live in a single `.html` file — do not split into multiple files
- No npm, no build process, no transpilation
- No external dependencies of any kind (not even Google Fonts or icon libraries)
- Must work when opened directly as a local file (`file://`) with no server

---

## Architecture

### Core Classes
- `RNG` — Mulberry32 seeded PRNG. All randomness uses this so runs are deterministic and reproducible.
- `Organism` — Individual agent: position, genome (12-gene Float32Array), energy, age, derived hue.
- `World` — Grid state, organism map, tick logic, history. Owns all three environmental systems.
- `Renderer` — Canvas LED dot rendering: scorch overlay, glow pass, infected pulse, incubation tint.
- `WeatherSystem` — Sinusoidal seasonal cycles + random perturbations; two channels: light and temperature.
- `PathogenSystem` — Parameterized outbreak events with incubation phase; contact transmission; reproductive suppression.
- `CatastropheSystem` — Meteor (instant kill zone) and Wildfire (fuel-dependent spreading cellular automaton).

### Seeded Randomness
- All randomness uses a seeded PRNG so the same seed + settings always produce the same run.
- **Never use `Math.random()` directly** — always use the seeded RNG.
- Breaking determinism is a critical bug.

### Genome System
12 genes, all floats in `[0, 1]`, stored as a `Float32Array`. Indexed by the `GENE` constant object:
`HUE`, `HUE_TOL`, `PHOTO`, `PRED`, `DEF`, `META`, `MOB`, `AGG`, `KIN`, `REPRO`, `MUT`, `LIFE`.

The `MUT` gene is evolvable — it scales per-gene mutation probability for all offspring.

### URL Parameter System (if applicable)
- If any settings are encoded as URL parameters for sharing, every new user-facing setting must also be added as a URL parameter so shared links remain complete.
- Do not remove or rename existing URL parameters — this breaks shared links.

### Pin / Save System
- If a pin/save system is added, store state to `localStorage` under a key prefixed with `ledlife_`.

---

## Design Principles
1. **Determinism** — same seed + settings must always produce the identical simulation.
2. **Self-contained** — the file must work with no internet connection and no server.
3. **No regressions** — changes must not break existing seeds, saved states, or share URLs.
4. **Cross-platform** — test mentally against desktop Chrome/Firefox and iPad Safari.
5. **Performance awareness** — large grids with glow and energy trails are the stress case.
6. **Emergent behavior over scripted outcomes** — mechanics should create interesting dynamics, not dictate them.

---

## How to Add Common Things

### New Environmental System
1. Write a new class (e.g. `TerrainSystem`) owned by `World`
2. Call its tick logic from `World`'s main tick function
3. Add renderer support if it affects display
4. Add sidebar controls (collapsible section)
5. Document it in `README.md` under the appropriate section

### New Genome Gene
1. Add a new index to the `GENE` constant
2. Update `randomGenome()` and `biasedGenome()` to include it
3. Update `mutateGenome()` — it mutates all genes by index so usually automatic
4. Update `describeGenome()` to display it in the inspector
5. Document the gene in `README.md` under The Genome

### New Ecology Preset
1. Add a new entry to the presets definition
2. Implement its `biasedGenome()` weighting
3. Add it to the Starting Ecology `<select>` in the HTML
4. Document it in `README.md` under Starting Ecology

### New User Setting
1. Add the HTML control in the appropriate collapsible sidebar section
2. Wire up the JS handler
3. If the setting should persist/share, add a URL parameter
4. Document it in `README.md` under Settings Reference

---

## What NOT to Do
- Do not use `Math.random()` — use the seeded RNG
- Do not introduce any external dependency, CDN link, or import
- Do not split the code into multiple files
- Do not create versioned files (e.g. `ledlife-v0_16.html`) — edit `ledlife.html` directly
- Do not change the behavior of existing seeds without a clear reason — it breaks reproducibility
- Do not add settings without documenting them in `README.md`

---

## Testing
Open `ledlife.html` directly in a browser (`File > Open` or drag onto browser). No build step needed.

Key things to verify after any change:
- Simulation starts and runs without errors
- The browser console is clean (no JS errors)
- Organisms render correctly (colors, glow, inspector)
- Existing seeds produce the same behavior
- Performance is acceptable on a 100×100 grid with glow enabled

---

## Git & GitHub
- Remote: https://github.com/ReeseLloyd/LEDLife.git
- Branch: `main`
- After completing changes, commit and push:
  ```
  git add ledlife.html README.md
  git commit -m "v0.XX: short description of what changed"
  git push
  ```
- Commit messages should describe the functional change, not the mechanism
  (e.g., "v0.16: Fix inspector not updating when paused" not "v0.16: Call updateInspector() in pause handler")
- For milestone versions, also tag and push the tag:
  ```
  git tag v0.XX
  git push --tags
  ```
