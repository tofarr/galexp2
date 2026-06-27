# Section D2 — Galaxy Generation

D2 produces the world the game is played in. It is the largest pure-function subsystem in the domain: given `(seed, options)`, it deterministically builds a full galaxy (stars, planets, specials, candidate homeworlds) without placing any empire on it.

## Section contract

> **Galaxy Generation**: produces a deterministic `Galaxy` from `(seed, options)`. The generator is pure, deterministic, and side-effect-free. D2 picks **candidate** homeworlds but does not assign empires — final assignment to races happens in D3 (race homeworld preferences) and D4 (game setup).

**Owns**:
- The `Galaxy` value (a `Star[]` plus tagged candidates and specials).
- All randomness used during galaxy generation — injected via `rng: () => number`, never `Math.random()`.
- Validation of `GalaxyOptions` (size, race count, etc.).

**Does not own**:
- Empire placement on stars (D3.5 homeworld assignment + D4 game init).
- Anything that changes after the game starts — D2 runs once at game start.
- Save/load — D2 only generates; A2 (persistence) handles saving the resulting `Galaxy` as part of `GameState`.

## A note on the original chunk list

The DECOMPOSITION.md entry for D2 lists D2.5 as "Start position selection — pick one good homeworld per race." That sentence is inconsistent with D2's own contract ("Does not place empires"). We resolve this in two lines:

- **D2.5 (this section)** picks **candidate** homeworlds during galaxy generation — a tagged subset of planets that are "good enough" to be someone's homeworld.
- **D3.5 (added when D3 is decomposed)** assigns each race to one candidate, using the race's homeworld preferences.

D2 stays pure ("here are the planets"). D3 brings preference logic ("this race likes Terrestrial, size 6+, rich"). D4 ties them together at game init.

## Top-level chunks

Reordered for dependency cleanliness — D2.0 (options & validation) goes first because everything else reads from it:

- **D2.0 Options & validation** — parse `GalaxyOptions`, derive star count, validate race count fits galaxy size.
- **D2.1 Cluster centers** — pick K cluster center points; K depends on galaxy size.
- **D2.2 Star field placement** — for each star, pick a cluster and place near it; enforce minimum distance between stars.
- **D2.3 Spectral classification** — assign each star a class (O/B/A/F/G/K/M) using a game-tuned distribution.
- **D2.4 Planet generation per star** — number of planets, types, sizes, richness; depends on spectral class.
- **D2.5 Star & planet specials** — tag a fraction of stars with `StarSpecial`s and a fraction of planets with `PlanetSpecial`s.
- **D2.6 Candidate homeworld selection** — tag a subset of planets as "good enough to be someone's homeworld"; emit the candidate list as part of the result.

## Recursive decomposition

Each top-level chunk is already close to leaf-sized. The sub-decomposition is mostly about clarifying inputs/outputs and the dependency direction.

### D2.0 → Options & validation

Inputs: `GalaxyOptions { size: GalaxySize, raceCount: int, specialDensity: float, ... }`.
Outputs: validated options + derived constants (`starCount: int`, `clusterCount: int`, map bounds).

Validation rules:
- `MIN_PLAYERS ≤ raceCount ≤ MAX_PLAYERS` (from D1.1 constants).
- `starCount` derived from `size`: SMALL=24, MEDIUM=36, LARGE=56, HUGE=80. (Numbers MoO-ish; tunable later.)
- `clusterCount ≈ starCount / 8`, clamped to `[3, 10]`.
- Map bounds derived: roughly `sqrt(starCount) × 12` parsecs in each axis.

This chunk is small (~30 lines) and trivial, but it deserves its own chunk because:
- Validation is testable in isolation (no randomness needed).
- It establishes the "what does `GalaxySize` mean?" table once, in one place, instead of scattered through D2.1–D2.6.

### D2.1 → Cluster centers

Pure function: pick K points within map bounds, with a minimum distance between clusters (so clusters are spread out). No dependence on star count or spectral class.

Output: `Cluster { id: ClusterId, x: int, y: int }[]`.

### D2.2 → Star field placement

Pure function: for each of N stars, pick a cluster (weighted — usually a 70/20/10 distribution favoring central clusters), then a position near that cluster (Gaussian offset, clipped to map bounds). Reject placements closer than `MIN_STAR_DISTANCE` to any existing star; re-roll up to a bounded number of attempts; on exhaustion, fall back to a uniform-random retry.

Output: `Star { id: StarId, x: int, y: int, clusterId: ClusterId }[]` (spectral class not yet assigned).

### D2.3 → Spectral classification

For each star, draw a spectral class from a game-tuned distribution. Real astrophysics skews heavily toward K and M, but MoO tuned the distribution to give more variety:
- O: 2%
- B: 4%
- A: 8%
- F: 14%
- G: 22%
- K: 25%
- M: 25%

(Sum = 100%. Tunable in v1; configurable in v2.)

Output: `Star { ..., spectral: SpectralClass }[]`.

### D2.4 → Planet generation per star

Number of planets depends on spectral class. Original MoO had roughly:
- O, B: 1–2 (huge stars, few stable orbits)
- A, F: 2–4
- G: 3–6 (Sol-like)
- K: 4–8
- M: 5–10

For each planet, roll type (weighted distribution favoring Terrestrial/Oceanic), size (1–10), and richness (1–10). Position within the star system is the `orbitIndex`; the tactical combat map (D9) uses it as a starting x-coordinate hint.

Output: `Planet { id, starId, orbitIndex, type, size, richness }[]`.

### D2.5 → Star & planet specials

A small fraction of stars get one `StarSpecial` (Ancient, PirateCache, GalacticCore) and a small fraction of planets get one `PlanetSpecial` (Fertile, MineralRich, UltraRich, Poor, Artifact, Native). Densities come from `GalaxyOptions.specialDensity`. Each kind has its own probability table.

Specials are assigned deterministically from the seed. No randomness outside what D2 already controls.

Output: `Star.specials` and `Planet.specials` filled in.

### D2.6 → Candidate homeworld selection

Scan generated planets; mark as a candidate homeworld any planet matching:
- `type ∈ {Terrestrial, Oceanic, Arid}` (hospitable for most races; some races in D3 expand this list).
- `size ≥ 5` (large enough to be productive).
- `richness ≥ 4` (productive enough).
- Not already tagged with `Native` or `Artifact` (those are gameplay-special and shouldn't be someone's starting world).

Output: `Planet.candidateHomeworld: bool` flag on each planet, plus a flat `candidates: PlanetId[]` list on the `Galaxy` value for convenient lookup.

The candidate count is not strictly controlled — it depends on the galaxy. In a typical medium galaxy we expect ~6–12 candidates, more than enough for `MAX_PLAYERS = 10`. If we ever need exactly N candidates (e.g. for scenario setup), that's a v2 knob.

## Dependency graph (within D2)

```
D2.0 (options) ──────────────────────────────────────┐
  ↓                                                  │
D2.1 (clusters) ── depends on D2.0 only             │
  ↓                                                  │
D2.2 (placement) ── depends on D2.1                 │
  ↓                                                  │
D2.3 (spectral) ── depends on D2.2                  │
  ↓                                                  │
D2.4 (planets) ── depends on D2.3 (spectral affects planet count) │
  ↓                                                  │
D2.5 (specials) ── depends on D2.4                  │
  ↓                                                  │
D2.6 (candidates) ── depends on D2.4                │
```

Strictly linear; easy to test each step in isolation. Note that D2.5 and D2.6 both depend on D2.4 but **not on each other**, so they can be tested in either order.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Star`, `Planet`, `PlanetType`, `SpectralClass`, `PlanetSpecial`, `StarSpecial`, `GalaxySize`, `GalaxyOptions`, `Galaxy`, `StarId`, `PlanetId`, `ClusterId` | D1.1 (primitives), D1.2 (entities) |
| D1.9 constants | `MIN_PLAYERS`, `MAX_PLAYERS`, galaxy size presets | D1.1 |

D2 has no other dependencies. D3 will import `Galaxy` and the candidates list. D4 will import `Galaxy` to assemble the initial `GameState`. D8 (fleet movement) reads the `Star` positions to build warp-lane topology.

| Section | What it imports from D2 |
|---|---|
| D3 Races & Traits | `Galaxy`, `Planet.candidateHomeworld` (used by D3.5 homeworld assignment) |
| D4 Turn Cycle | `Galaxy` (initial `GameState` assembly at game start) |
| D8 Fleet & Movement | `Star.x`, `Star.y`, `Cluster.id` (for pathfinding topology) |
| P3 Star Map Screen | `Star`, `Planet` (rendered; pure read) |

## Quint-spec-sized leaves (the actual implementation units)

The pipeline is small enough to live in one main spec with helpers, mirroring how D9 has one main spec plus a few siblings. Proposed layout:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/galaxy/galaxyOptions.qnt` | D2.0 | `src/domain/galaxy/galaxyOptions.ts` | ~50 |
| `specs/galaxy/starField.qnt` | D2.1 + D2.2 + D2.3 | `src/domain/galaxy/starField.ts` | ~120 |
| `specs/galaxy/planetGen.qnt` | D2.4 | `src/domain/galaxy/planetGen.ts` | ~100 |
| `specs/galaxy/specials.qnt` | D2.5 | `src/domain/galaxy/specials.ts` | ~80 |
| `specs/galaxy/candidates.qnt` | D2.6 | `src/domain/galaxy/candidates.ts` | ~60 |
| `specs/galaxy/galaxyGen.qnt` | top-level `generateGalaxy(seed, options, rng)` orchestrator | `src/domain/galaxy/galaxyGen.ts` | ~80 |

Total ~490 lines of Quint — comfortably within Phase 1's leaf-sized budget when split across five spec files.

Each file maps 1:1 to a TypeScript module. The orchestrator (`galaxyGen.qnt`) imports the helpers; the helpers are independently testable.

## What "simple v1" looks like for D2

- **Cluster-based galaxy**, not pure random. Matches the original MoO's recognizable feel and gives the star map visible structure.
- **Game-tuned spectral distribution**, not astrophysically realistic. We want variety, not realism.
- **No special "advanced" star types** (black holes, neutron stars, etc.). MoO had only "Ancient", "Pirate Cache", and "Galactic Core". v1 stops there.
- **No balance sliders** for race placement — all candidates are equally viable; race preference (D3) picks among them.
- **No procedural names for stars and planets** — they're `Star #42` and `Planet #42.3` until P3 assigns names from a word list. (Names are a presentation concern, not a domain concern.)
- **No cluster-aware pathfinding bonuses** — all stars are equally reachable. Distance is Euclidean (we'll add warp lanes in D8).

## Resolved decisions for D2

- **D2.5 is "candidate homeworld selection", not empire placement.** Final assignment to races is D3.5's job (added when D3 is decomposed). D2 stays focused on world generation.
- **Cluster-based galaxy structure.** Original MoO feel; tests can validate "clusters exist" as an invariant.
- **Game-tuned spectral distribution.** Variety > realism. Values listed in D2.3 above; tunable.
- **Planet count varies with spectral class.** Listed in D2.4 above.
- **Candidates are tagged, not enumerated.** No magic candidate count; the count is whatever the galaxy produces.

## Open questions for D2

- **Should `GalaxyOptions` include a "player-controlled race count" separate from "total race count"?** v1: total race count = human + AI players. If we later allow observers, this becomes a separate field. Defer until needed.
- **Density of specials**: default MoO had roughly one special per 8–10 stars. v1 picks a default but makes `specialDensity` configurable. No open question.
- **Should we guarantee a minimum number of candidates?** If a small galaxy with bad luck produces only 2 candidates, we have room for only 2 players. Suggested mitigation: re-roll the seed if candidates < MIN_PLAYERS. **Defer to v2** — a 24-star galaxy in practice produces 6+ candidates every time we ran the original MoO; the edge case is rare and recoverable by a UI "re-roll seed" button.

No open questions block starting the D2 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D3 Races & Traits** (which will introduce D3.5 to receive D2's candidate list).

D2's Quint spec can be written **now** without waiting for D3/D4/D7/D8 — D2 has only D1 dependencies, which are already decomposed. Once D2's spec lands, D3's spec can import `Galaxy` from it.
