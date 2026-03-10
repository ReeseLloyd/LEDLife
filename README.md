# LEDLife v0.18

A browser-based artificial life simulator with an LED dot-matrix aesthetic. Organisms are mobile agents on a fixed grid — each living cell is a glowing LED dot whose color encodes its genome-derived phenotype. The simulation runs an approximately energy-conserved economy with sexual and asexual reproduction, predation, defense, evolvable mutation rates, weather cycles, pathogens, and environmental catastrophes.

Inspired by **Primordial Life** (Tim Hutton, 1996).

---

## Quick Start

Open `ledlife.html` in any modern browser. No build step, no dependencies, no server required. The simulation field auto-fits to the available display area on load. Click **▶ Start** to begin. Click any organism on the canvas to inspect its genome in a floating panel. Scroll/pinch to zoom; two-finger pan to navigate.

---

## Architecture

Single self-contained HTML file (~3140 lines). No external dependencies.

### Classes

| Class | Responsibility |
|---|---|
| `RNG` | Mulberry32 seeded PRNG — deterministic, reproducible runs |
| `Organism` | Individual agent: position, genome, energy, age, derived base hue |
| `World` | Grid state, organism map, tick logic, history; owns all three environmental systems |
| `Renderer` | Canvas LED dot rendering: scorch overlay, glow pass, infected pulse, incubation tint |
| `WeatherSystem` | Sinusoidal seasonal cycles + random perturbations; two channels: light and temperature |
| `PathogenSystem` | Parameterized outbreak events with incubation phase; contact transmission; reproductive suppression |
| `CatastropheSystem` | Meteor (instant kill zone) and Wildfire (fuel-dependent spreading cellular automaton) |

### Key standalone functions

| Function | Purpose |
|---|---|
| `hslToRgb(h, s, l)` | HSL → RGB conversion |
| `randomGenome(rng)` | Uniformly random 12-gene Float32Array |
| `biasedGenome(rng, ecology)` | Soft-biased genome for ecology presets |
| `mutateGenome(parent, rng, baseMutRate)` | Per-gene mutation with evolvable rate |
| `crossGenomes(a, b, rng)` | 50/50 gene-by-gene crossover |
| `genomicDistance(a, b)` | Circular hue distance — used for mate compatibility and kin recognition |
| `genomeToBaseColor(g)` | Derives display hue from genome (predator→red, defender→blue, autotroph→green shift) |
| `hueNicheBonus(h, g)` | Per-tick energy bonus from hue–niche alignment |
| `classifyStrategy(g)` | Returns: Autotroph / Predator / Defender / Opportunist / Generalist |
| `getShape(g)` | Returns LED shape for a genome: hexagon / diamond / square / triangle / cross / circle |
| `drawLEDShape(ctx, cx, cy, r, shape)` | Fills the given LED shape centered at (cx, cy) with radius r |
| `describeGenome(org)` | Returns HTML for the floating genome inspector |
| `drawStrategyChart(canvas, history)` | Renders 3-line strategy balance chart |
| `drawHueChart(canvas, world)` | Renders live hue diversity histogram |
| `fitToWindow()` | Calculates and applies zoom so the simulation field fills the display area on load |
| `logEvent(type, msg)` | Prepends a timestamped entry to the event log |

---

## The Genome

12 genes, all floats in `[0, 1]`, stored as a `Float32Array`. Indexed by the `GENE` constant object.

| Index | Key | Description |
|---|---|---|
| 0 | `HUE` | Species/lineage identity — maps to the full color wheel. The primary speciation axis. |
| 1 | `HUE_TOL` | Reproductive isolation dial. Low = strict species boundaries; high = interbreeds across hue gaps. |
| 2 | `PHOTO` | Photosynthesis — passive solar energy gain per tick (quadratic scaling). |
| 3 | `PRED` | Predation strength — energy stolen per successful attack. |
| 4 | `DEF` | Defense — resistance to predation attempts. |
| 5 | `META` | Metabolism rate — base energy burned per tick. Also determines thermal regulation (see Weather). |
| 6 | `MOB` | Mobility — probability of attempting to move each tick (subject to Movement Mode scaling). |
| 7 | `AGG` | Aggression — probability of attempting to attack an adjacent organism each tick. |
| 8 | `KIN` | Kin tolerance — suppresses attacks against similar-hued organisms when high. |
| 9 | `REPRO` | Reproduction threshold — energy fraction needed before spawning. |
| 10 | `MUT` | Mutation rate — **evolvable**. Scales per-gene mutation probability for all offspring. |
| 11 | `LIFE` | Lifespan fraction — scales max age against the global Max Lifespan parameter. |

### Phenotype derivation

`genomeToBaseColor(g)` applies three hue shifts to `HUE`:
- **Predator-dominant** (`PRED − DEF > 0.1`): hue pulled toward red (0°)
- **Defender-dominant**: hue pulled toward blue (234°)
- **Photosynthesizer** (`PHOTO > 0.5`): hue pulled toward green (119°)

Saturation = `0.6 + PRED×0.3 + PHOTO×0.2`. Lightness = `0.25 + energy×0.45` at render time — organisms visibly dim when starving and glow when well-fed.

**Infection rendering:**
- **Incubating** (first 35% of infection): subtle warm yellow-green tinge, slow dim pulse — looks almost healthy; the silent spread wave
- **Symptomatic** (remaining 65%): desaturated toward grey-purple, fast sickly pulse

### Strategy classification

`classifyStrategy(g)` labels organisms for the inspector and strategy chart:

| Label | Condition |
|---|---|
| **Autotroph** | `PHOTO > 0.6` and `PRED < 0.4` |
| **Predator** | `PRED > 0.6` and `DEF < 0.4` |
| **Defender** | `DEF > 0.6` |
| **Opportunist** | `PRED > 0.5` and `PHOTO > 0.5` |
| **Generalist** | Everything else |

---

## Energy Economy

Energy is approximately conserved; `MAX_ENERGY = 1.0`.

### Gains per tick
- **Photosynthesis:** `(PHOTO² × solarIntensity × 0.8 + solarPerCell × 0.5) × weatherLightMult`
- **Hue niche bonus** (see below): up to ~0.012/tick at a niche peak
- **Rare-hue bonus**: up to ~0.0027/tick when hue bin represents < 15% of population
- **Predation:** stolen energy × 0.7 transferred on successful attack

### Costs per tick
- **Metabolism:** `0.002 + META×0.006 + MOB×0.001 + PRED×0.001`
- **Weather temperature stress:** `|tempOffset| × (1 − META) × 0.003` — cold-blooded organisms penalized during heat waves and cold snaps
- **Pathogen drain (incubating):** `virulence × 0.03` — light, but enough to suppress energy buildup and reduce reproduction
- **Pathogen drain (symptomatic):** `virulence × 0.12` — competing with photosynthesis; virulence 0.4 costs 0.048/tick (stressful); virulence 1.5 costs 0.18/tick (lethal even for well-fed autotrophs)
- **Reproduction:** `0.25 + REPRO×0.1` from parent; offspring receives 70% of that
- **Mating cost:** 0.08 energy deducted from the non-initiating mate

Death occurs when `energy ≤ 0` or `age > 100 + LIFE × maxLifespan`. Half of a dead organism's remaining energy returns to the global `energyPool`.

---

## Reproduction

Triggered when `energy > 0.45 + REPRO×0.4`. Offspring placed in a random empty non-scorched neighbor cell.

**Infection suppresses reproduction:**
- Symptomatic organisms: reproduction fully blocked
- Incubating organisms: 50% chance to skip reproduction each tick

This is the mechanism by which outbreaks crash populations rather than simply taxing them — it cuts off birth rate at the same time death rate rises.

**Sexual reproduction** is attempted first. A random occupied neighbor is evaluated as a candidate mate. Mating succeeds if `genomicDistance(org, candidate) < avgHueTolerance × 0.5`:
- Genomes crossed 50/50 gene by gene
- `HUE` gene inherited from **one parent by coin flip** — not averaged — preserving species identity
- Crossed genome then mutated
- Pathogen can transmit sexually at 40% probability if either mate is infected

**Asexual reproduction** (clone + mutate) occurs when no compatible mate is nearby.

### Speciation

`HUE_TOL` enforces reproductive isolation. Low-tolerance populations that diverge in hue lose the ability to interbreed. Hue inheritance (rather than blending) means adjacent populations maintain distinct hues rather than converging to an intermediate.

---

## Hue Diversity Mechanics

Three mechanisms maintain long-term color spectrum activity:

### A — Hue–niche coupling
Three energy bonus peaks reward organisms whose hue aligns with their dominant strategy:

| Peak | Hue | Benefits gene | Max bonus/tick |
|---|---|---|---|
| Green | ~120° (0.33) | `PHOTO` | 0.012 |
| Red | ~0°/360° (0.00) | `PRED` | 0.010 |
| Blue | ~234° (0.65) | `DEF` | 0.008 |

Effective radius ≈ ±65° on the color wheel. Bonus = `max(0, 1 − dist/0.18) × geneValue × constant`.

### B — Reproductive isolation
`HUE_TOL = 0` makes a lineage reproductively isolated from organisms with any different hue. Combined with `HUE` inheritance (one parent, not blended), distinct species can coexist spatially without genetically merging. The mating floor (a previous 0.05 minimum compatibility) was removed to enforce strict speciation.

### C — Rare-hue frequency-dependent selection
Organisms in sparse hue bins (< 15% of population) receive a small energy bonus per tick — up to ~0.0027/tick. This opposes drift-driven fixation: when a hue becomes dominant it loses its bonus; rare hues gain advantage. This sustains diversity even after one lineage becomes numerically dominant.

---

## Environmental Systems

### Weather

A `WeatherSystem` runs two sinusoidal channels updated each tick:

**Light channel:** A primary seasonal sine wave (period = Season Length) modulated by random perturbations (storms). Multiplies all photosynthetic energy gain. Range approximately 0.4×–1.4× baseline solar.

**Temperature channel:** Out-of-phase seasonal sine + perturbations. Organisms with low `META` (cold-blooded) are penalized during extreme temperatures in either direction.

Weather transitions are logged to the event log. Overlay shows current icon, light %, and temperature bar in the top-left of the display area.

| Icon | Condition |
|---|---|
| ☀ | Clear — solar at peak |
| 🌤 | Partly cloudy — mild |
| 🌧 | Storm — sharply reduced light |
| 🌡 | Heat wave — metabolic stress for low-META organisms |
| ❄ | Cold snap — same penalty, opposite temperature direction |

### Pathogen

Parameterized outbreaks; not co-evolving agents. Each outbreak triggers with current slider values.

**Lifecycle:**
1. 2–3% of eligible organisms seeded as initial infected
2. **Incubation phase** (first 35% of duration): organism spreads normally, receives light energy drain (`virulence × 0.03`), shows warm yellow tinge. Reproduction 50% suppressed.
3. **Symptomatic phase** (remaining 65%): full energy drain (`virulence × 0.12`), grey-purple coloration with fast pulse. Reproduction fully blocked.
4. After `200 + 300/(virulence + 0.1)` ticks, organism recovers (removed from infected map). Shorter duration at higher virulence — fast killers don't linger.

**Transmission:** Contact-based — each tick, each infected organism checks occupied neighbors. Transmission probability = `spreadRate × 0.35` per neighbor. Sexual transmission also occurs at 40% probability during mating if either partner is infected.

**Target modes:** Any · Predators only · Autotrophs only · Hue band (random hue sampled from eligible population at outbreak trigger)

**Observable epidemic arc:** Silent spread wave (yellow tinge radiating through population) → symptomatic crash (grey-purple die-off, birth rate collapsed) → recovery as survivors repopulate.

### Catastrophe

**Meteor:** Instant circular kill zone. All organisms within `meteorRadius` cells of the impact point are removed. Cells scorched for `120 + rand(40)` ticks. Wraps around grid edges.

**Wildfire:** Spreading cellular automaton.
- Ignites 2–3 seed cells near the click point (TTL 15–30 ticks each)
- Each tick, each live fire cell attempts to spread to its 4 cardinal neighbors
- **Spread requires fuel** — fire only spreads to cells occupied by a living organism (bare ground does not carry fire)
- **Spread requires unscorched ground** — target cell must have zero scorch remaining; recently burned cells cannot re-ignite until fully recovered (120 ticks)
- If spread succeeds: organism killed, cell scorched, new fire cell created (TTL 15–30)
- Fire self-extinguishes naturally when it exhausts available fuel in an area
- Hard cap: fire cannot exceed 8% of grid cells simultaneously

**Scorch:** Scorched cells block organism movement and reproduction. Scorch timer counts down 1/tick. `SCORCH_RECOVERY = 120` ticks. Intensity visualized as a dark overlay in the renderer, fading as recovery progresses.

---

## Settings Reference

All settings are in the collapsible sidebar. Click any section header to collapse/expand it.

### Simulation
| Control | Default | Notes |
|---|---|---|
| ▶ Start / ⏸ Pause | — | Toggles simulation running state |
| ↺ Reset | — | Stops and re-initializes with current settings and seed |
| Speed | 30 | Target ticks per second (1–120) |
| Seed | random | Integer seed for the RNG. Same seed + settings = identical run. Click 🎲 to randomize. |
| 🎰 Randomize Parameters | — | Randomizes population, solar intensity, mutation rate, and lifespan |
| 🔗 Copy Share Link | — | Copies a URL encoding all current settings; opening it restores the exact configuration |

### World
| Control | Default | Range | Notes |
|---|---|---|---|
| Grid Width | 100 | 20–200 | Columns; step 4. Applied on Reset. |
| Grid Height | 100 | 20–200 | Rows; step 2. Applied on Reset. |
| Initial Population | 300 | 10–1000 | Organisms seeded at start; step 10 |
| Solar Intensity | 0.15 | 0.02–0.50 | Scales all photosynthetic gain |

### Starting Ecology
Four preset genome bias profiles applied at seed time. Evolution can diverge from any preset.

| Preset | Character |
|---|---|
| 🧫 Primordial Soup | Fully random genomes — pure evolutionary chaos |
| 🌱 Garden | Photosynthesizer-heavy, low predation — lush and stable until a predator mutates |
| ⚔️ Battleground | Predators and prey seeded in balance — boom-bust arms race dynamics |
| 🤝 Commune | High kin tolerance, low aggression — clustering and mutualism vs. invasion pressure |

### Movement Mode
| Mode | Effect |
|---|---|
| Fixed | No movement; spread by reproduction only (CA-style colony growth) |
| Hybrid | Limited movement (mobility scale 0.4×) — some drift, colonies can form |
| Mobile | Full movement (mobility scale 1.0×) — organisms roam freely |

### Evolution
| Control | Default | Range | Notes |
|---|---|---|---|
| Base Mutation Rate | 0.03 | 0.001–0.15 | Added to each organism's evolvable `MUT` gene rate |
| Max Lifespan | 500 | 50–2000 | Upper bound of age range; scaled by `LIFE` gene |

### Population
Live charts and stat boxes. See **Population Charts** section below.

### Display
| Control | Default | Notes |
|---|---|---|
| LED glow effect | on | Radial gradient behind each organism; disable for performance on large grids |
| Show unlit panel dots | on | Dim dots at empty cells; creates the LED matrix look |
| Energy trails (slow) | off | Fading energy traces; significant performance cost |
| LED shapes by strategy | on | Each organism's dot shape reflects its dominant gene: hexagon=autotroph, diamond=predator, square=defender, triangle=mobile, cross=kin-social, circle=generalist |

### Weather
| Control | Default | Range | Notes |
|---|---|---|---|
| Enable weather cycles | on | — | Disabling gives constant solar at base intensity |
| Season length | 400 | 50–2000 | Ticks per full light cycle; step 50 |
| Intensity | 0.50 | 0.1–1.0 | Amplitude of weather perturbations; step 0.05 |

### Pathogens
| Control | Default | Range | Notes |
|---|---|---|---|
| Virulence | 0.40 | 0.05–1.50 | Energy drain coefficient. 0.4 = stressful; 1.5 = lethal |
| Spread rate | 0.05 | 0.01–0.50 | Per-contact transmission probability multiplier |
| Target | Any | Any / Pred / Auto / Hue | Restricts which organisms can be infected |
| 🦠 Trigger Outbreak | — | — | Seeds 2–3% of eligible organisms as initial infected |

### Catastrophes
| Control | Default | Range | Notes |
|---|---|---|---|
| ☄️ Meteor | — | — | Strikes a random grid location |
| 🔥 Wildfire | — | — | Ignites at a random grid location |
| Meteor radius | 8 | 2–30 | Kill zone radius in cells |
| Fire spread prob. | 0.12 | 0.00–0.20 | Per-tick chance to spread to an occupied neighbor |

### Event Log
Scrollable log of notable simulation events, newest first (max 40 entries). Color-coded by type: blue=weather, pink=pathogen, orange=fire, purple=meteor.

---

## Population Charts

Both charts update every 10 ticks, displaying the last 200 samples (~2,000 ticks of history).

### Strategy balance
Three-line time-series, Y-axis normalized to total population:
- **Green** — Autotroph count
- **Red** — Predator count
- **Gray** — All others (Defender, Opportunist, Generalist)

Classic Lotka-Volterra oscillations appear as the red line lagging and crashing after green population peaks. Pathogen outbreaks targeting autotrophs appear as a green line collapse with subsequent predator crash.

### Hue diversity histogram
36 bins × 10° each, covering the full color wheel. Each bar is colored by the hue it represents. Persistent peaks at ~120° (green/autotrophs), ~0°/360° (red/predators), and ~234° (blue/defenders) indicate the niche attractors at work. New peaks emerging indicate speciation events; disappearing peaks indicate lineage extinction.

---

## Genome Inspector

Click any organism on the canvas to open the floating genome inspector. Updates live every tick. Draggable by its header; ✕ to close. Available in both normal and focus mode.

**Header:** Color swatch (organism's actual rendered color), strategy label, age, ID.

| Section | Fields | Bar color |
|---|---|---|
| Vitals | Energy (traffic-light: green/orange/red), Lifespan | — |
| Energy Budget | Photosynthesis, Predation, Defense, Metabolism | Green, Red, Blue, Gray |
| Behavior | Mobility, Aggression, Kin Tolerance | Purple, Orange, Teal |
| Reproduction | Repro Threshold, Hue Tolerance, Mutation Rate | Yellow, Light blue, Pink |

---

## Focus Mode

Click **⛶** (top-right corner) to enter focus mode:
- Sidebar and header hidden; canvas fills the full viewport
- Overlay controls: **⏸/▶** and **⊠** (or Esc to exit)
- Space bar toggles play/pause
- Clicking an organism opens the floating genome inspector (same as normal mode)
- View transform (zoom/pan) persists across focus mode transitions

---

## Zoom / Pan

| Gesture | Action |
|---|---|
| Two-finger scroll (trackpad) | Pan the view |
| Pinch in/out (trackpad or touch) | Zoom toward pinch center |
| Scroll wheel (mouse) | Pan vertically |
| Ctrl + scroll wheel | Zoom toward cursor |
| Left-drag (when zoomed in) | Pan |
| Middle-mouse drag | Pan (any zoom level) |
| Alt + left-drag | Pan (any zoom level) |
| Double-click | Reset zoom and pan |
| Touch single-finger drag (zoomed) | Pan |
| Touch two-finger pinch | Zoom |

Zoom range: 25%–800%. On load and resize, the view auto-fits so the full simulation field is visible. A hint label in the bottom-left shows the current zoom level and fades after 1.8 seconds. Click coordinates are remapped through the transform so organism selection works correctly at all zoom levels.

---

## Emergent Dynamics to Watch For

- **Lotka-Volterra oscillations** — predator peaks lagging autotroph blooms on the strategy chart, then crashing as prey runs thin
- **Seasonal population rhythm** — total population visibly rises and falls with the light cycle; troughs during storms and cold snaps
- **Epidemic curves** — the incubation wave spreads as a warm yellow tinge across the field; then the symptomatic die-off wave follows; birth rate collapse amplifies the crash; recovery as survivors repopulate
- **Hue attractor peaks** — long-running simulations develop stable histogram bars at green/red/blue niche positions
- **Speciation events** — a new hue mutant in sparse histogram territory blooms rapidly via the rare-hue frequency bonus
- **Colony formation** — Fixed or Hybrid movement + high `KIN` produces sharp-bordered monochromatic patches competing for territory
- **Arms races** — Battleground mode drives `PRED` and `DEF` genes upward over hundreds of generations; the hue histogram polarizes toward red and blue
- **Fire corridors** — wildfire burns through dense populations and self-extinguishes at low-density margins; scorched zones recover visibly as color returns
- **Cascade events** — drought (low solar season) + pathogen outbreak + meteor impact in rapid succession can destabilize even a resilient population

---

## Known Design Decisions and Tradeoffs

**Energy conservation is approximate.** `solarGain` is applied per-organism independently of a global budget, meaning a large population receives proportionally more total solar energy. `energyPool` collects energy from deaths but is not redistributed. This is intentional — strict conservation caused population crashes at high density.

**`genomicDistance` uses only hue.** The distance function for mate compatibility and kin recognition ignores all non-hue genes. Hue is the sole speciation axis. A richer multi-gene distance would be more biologically accurate but harder to visualize and tune.

**Pathogens are parameterized, not evolving.** Each outbreak is a fixed parameter set triggered manually. This keeps the system predictable and tunable. A fully co-evolutionary pathogen would require its own genome representation and fitness landscape.

**Wildfire requires living fuel.** Fire does not spread across bare ground or recently scorched cells. This means fire self-extinguishes naturally as it exhausts fuel, and low-population areas act as natural firebreaks. A fire in a very sparse population will die quickly regardless of the spread probability slider.

**Fire cap is hard-coded at 8%.** `MAX_FIRE_CELLS` is a safety limit that prevents total extinction regardless of spread setting. At maximum spread probability (0.20), fire is dramatic but bounded.

**Weather perturbations are seeded per-world.** Random weather events use the world's RNG, so a given seed + ecology combination always produces the same weather history. Weather is deterministic and reproducible but not independently controllable.

**Sexual reproduction requires spatial proximity.** Only the 8-cell Moore neighborhood is checked for mates. In Fixed movement mode, organisms rarely change neighbors, so most reproduction is asexual.

**Auto-repopulation on extinction.** If the entire population dies, the world re-seeds at 5% of initial population every 20 ticks using the current ecology preset.

---

## Version History

| Version | Summary |
|---|---|
| **v0.18** | Share link: 🔗 Copy Share Link button encodes all settings as URL parameters; opening the link restores the exact configuration and seed |
| **v0.17** | Genome inspector swatch now renders the organism's LED shape (50% larger; uses CSS clip-path matching the canvas geometry); drop-shadow follows shape outline |
| **v0.16** | LED shapes by strategy: organism dots now render as hexagon (autotroph), diamond (predator), square (defender), triangle (high-MOB), cross (high-KIN), or circle (generalist) based on dominant genome gene above 0.55; toggleable in Display settings |
| **v0.15** | Pathogen: symptomatic organisms blocked from reproducing; incubating organisms 50% suppressed; light incubation energy drain added — outbreaks now crash populations rather than just taxing them. Wildfire: fire requires living fuel to spread; only spreads to fully unscorched cells — eliminates eternal re-ignition feedback loop; fires self-extinguish naturally |
| **v0.14** | Pathogen overhaul: energy drain raised 0.035→0.12; spread multiplier raised 0.08→0.35; incubation phase added (silent spread + yellow tinge before symptomatic die-off); renderer distinguishes incubating vs symptomatic states |
| **v0.13** | Layout fixes: `html/body` locked to viewport height (window no longer scrolls); canvas `position:absolute` removes double-offset centering conflict; `fitToWindow()` deferred one rAF for accurate dimensions |
| **v0.12** | Bug fixes carried forward from v0.11 |
| **v0.11** | Floating genome inspector active in both normal and focus modes (sidebar panel removed); all settings sections collapsible; Randomize Parameters moved below Seed; settings sidebar scrolls independently; default zoom auto-fits simulation field to display area |
| **v0.10** | Trackpad zoom/pan (pinch=zoom, two-finger scroll=pan via ctrlKey detection); fire spread range narrowed to 0–0.20; pathogen mortality ×0.035; virulence slider extended to 1.5 |
| **v0.9** | Wildfire rebalanced (shorter TTLs 15–30, bare-ground resistance, 8% grid cap, no edge wrap); pathogen default spread 0.05, redundant proximity pass removed, mortality ×0.012; zoom/pan system added (scroll-wheel zoom toward cursor, drag-to-pan, pinch-zoom, double-click reset) |
| **v0.8** | Weather system (seasonal cycles, perturbations, cold-bloodedness, canvas overlay); Pathogen system (contact + sexual transmission, strategy/hue targeting, infected pulse rendering); Catastrophe system (Meteor, Wildfire, scorch grid, event log) |
| **v0.7** | Hue diversity: (A) niche coupling — green/red/blue peaks confer strategy bonuses; (B) reproductive isolation — strict `HUE_TOL` enforced, hue inherited from one parent not blended; (C) rare-hue frequency-dependent selection bonus |
| **v0.6** | Population charts: strategy balance 3-line chart + live hue diversity histogram; shared `classifyStrategy()` |
| **v0.5** | Starting Ecology presets (Soup / Garden / Battleground / Commune); Movement Mode toggle (Fixed / Hybrid / Mobile) |
| **v0.4** | Floating genome inspector in focus mode — draggable, live-updating, ✕ close button |
| **v0.3** | Focus mode (⛶/⊠/Esc); Space bar play/pause; genome inspector redesigned with color-coded bars |
| **v0.2** | Start/pause combined; Reset; 🎲 seed randomize; 🎰 randomize all; 100×100 default grid |
| **v0.1** | Initial release — grid world, 12-gene genome, LED renderer, energy economy, sexual/asexual reproduction |

---

## Future Exploration

- **Resource gradients / terrain** — non-uniform solar intensity creating stable ecological zones with distinct evolutionary pressures
- **Nutrient cycling** — organism deaths fertilize local cells, creating spatial bloom-after-die-off dynamics
- **Dormancy / spore states** — low-energy organisms enter suspended animation during hostile periods
- **Horizontal gene transfer** — genes diffuse sideways between proximate organisms without reproduction
- **Symbiosis** — persistent adjacent pairs develop mutualism from `KIN` + `AGG` gene interaction
- **Gene locks** — per-gene UI controls to constrain evolution along specific axes during a run
- **Export / save state** — serialize world to JSON for resuming sessions
- **Species tracking** — label and follow named lineages across generations
- **Visible epidemic stats** — infected count and phase breakdown added to the stats grid
- **Evolving pathogens** — pathogen strain with its own mutation rate and virulence gene that co-evolves with host resistance
