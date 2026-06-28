# Section D6 — Research & Tech Tree

D6 defines the tech tree, accumulates research points, decides when techs complete, and handles tech trade between players. It's the bridge between the economy (which produces research points) and the rest of the game (which consumes tech unlocks).

## Section contract

> **Research & Tech Tree**: defines the tech graph (6 trees), computes tech costs, applies research, and emits tech-acquired events. Also handles tech trade between players (used by diplomacy).

**Owns**:
- The tech catalog (graph of techs with prereqs and effects).
- Tech cost calculation (base cost × level × race modifiers).
- Per-player research selection and accumulation.
- Tech acquisition event emission and effect application.
- Tech trade validation (a tech can only be given to a player who has the prereqs).

**Does not own**:
- Research point generation (D5.4 produces `grossResearch`; D6 consumes it).
- Ship component unlocks (D7.1/D7.2/D7.3 catalogs reference prereqs from D6.1, but D7 owns the catalogs themselves).
- Building unlocks in production queues (D5.7 reads unlocked-buildings state set by D6.4).
- Trade offers (D11 owns the offer flow; D6.5 just provides the trade primitive).

## Top-level chunks

Five top-level chunks from `DECOMPOSITION.md`:

- **D6.1 Tech graph definition** — nodes (techs) and edges (prereqs).
- **D6.2 Tech cost calculation** (level × base cost, with race modifiers).
- **D6.3 Research selection & accumulation** — what each empire is researching.
- **D6.4 Tech acquisition events** — when a tech completes, who learns it, what effects fire.
- **D6.5 Tech trade & gifts** — used by diplomacy.

## Recursive decomposition

### D6.1 → Tech graph definition

Six tech trees, each roughly linear with some cross-tree prereqs:

| Tree | Levels (v1) | Theme |
|---|---|---|
| Weapons | 6 | Beam/missile/torpedo damage, accuracy, range |
| Propulsion | 6 | Ship speed, warp range |
| Construction | 6 | Factory output, hull space, ship cost reduction |
| Computer | 6 | Targeting accuracy, ECM resistance, scanner range |
| Shields | 6 | Shield capacity, regen |
| Force Fields | 6 | Planet shield generators |

~36 techs total in v1 (6 trees × 6 levels). Each `Tech` record:

```
type Tech = {
  id: TechId,
  name: string,
  tree: TechTree,
  level: int,                    // 1..6 within the tree
  baseCost: int,                 // industry-equivalent research cost
  prereqs: Set<TechId>,          // cross-tree prereqs allowed
  effects: List<TechEffect>,
}
```

`TechEffect` is an ADT — each effect type triggers a different side effect when the tech is acquired:

```
type TechEffect =
  | UnlockBuilding(BuildingKind)
  | UnlockHull(HullId)
  | UnlockWeapon(WeaponId)
  | UnlockSpecial(SpecialId)
  | StatBonus(stat: ShipStat, amount: int)             // e.g., +1 ship attack
  | ProductionBonus(tree: TechTree, multiplier: float) // e.g., +10% weapons output
  | SpecialEffectBonus(kind: SpecialKind, amount: int) // e.g., shields +20%
```

The catalog itself is a `Map<TechId, Tech>` constant in the Quint spec. v1 uses 36 hardcoded techs.

Cross-tree prereqs example: "Computer IV" requires "Construction II" (advanced computers need better ships to mount). The `prereqs` field allows arbitrary edges; v1's graph is mostly linear but a few cross-tree prereqs add strategic depth.

### D6.2 → Tech cost calculation

Pure function: `techCost(techId, race) -> int`.

```
techCost(techId, race) =
    tech.baseCost
  × tech.level                                       // higher level = more expensive
  × (1 - race.techCostReduction)                     // e.g., -10% if race has Technologist
  × treeCostModifier(tech.tree, race)                // D3.3 affinity: small bonus on favored trees
```

`treeCostModifier(tree, race) = 1.0` by default; some races have `0.8` on their favored tree (Psilons get Weapons faster).

Cost is paid in **accumulated research points** (one tech at a time). The player can't research a tech they don't have the prereqs for.

### D6.3 → Research selection & accumulation

Per-player function: `applyResearch(player, grossResearch) -> (player', events)`.

Logic:
1. If `player.currentResearch = None`: emit `NoResearchEvent { player }` (player forgot to pick). Return unchanged.
2. Else: `accumulated = player.researchAccumulated + grossResearch × researchEfficiencyModifier(player.race)`.
3. If `accumulated >= techCost(player.currentResearch, player.race)`:
   - Subtract cost from accumulated.
   - Acquire the tech (call D6.4).
   - Reset `currentResearch` to `None`.
4. Else: update `player.researchAccumulated = accumulated`.

```
researchEfficiencyModifier(race) =
    product of all `ResearchEfficiency(float)` variants in `totalModifiers(race)`
    // e.g., Erudite grants ResearchEfficiency(1.20) → 20% bonus
```

This is the v1 path: the trait-derived `ResearchEfficiency` modifier (D3.2
`Modifier` ADT) is the single source of truth for the per-turn research
multiplier. Race tech affinities (D3.3) are applied separately as
`treeCostModifier` inside `techCost`.

If the player changes `currentResearch` mid-research, accumulated points are **lost** (MoO behavior; v1 matches). v2 could allow partial carry-over.

Commands accepted in this phase:
- `SetResearch(player, techId)` — change current research.
- `ClearResearch(player)` — abandon current research.

### D6.4 → Tech acquisition events

Pure function: `acquireTech(player, techId) -> (player', events)`.

Steps:
1. Add `techId` to `player.techs`.
2. Apply each `TechEffect` to `player` (or to the global catalogs).
3. Emit `TechAcquiredEvent { player, tech, effects }`.

Effect application is the meatiest part:
- `UnlockBuilding(BuildingKind)` — adds to `player.unlockedBuildings`.
- `UnlockHull(HullId)` — adds to `player.unlockedHulls`.
- `UnlockWeapon(WeaponId)` — adds to `player.unlockedWeapons`.
- `UnlockSpecial(SpecialId)` — adds to `player.unlockedSpecials`.
- `StatBonus(stat, amount)` — adds to `player.shipStatBonuses[stat]`. Read by D9 (combat) when applying per-instance modifiers.
- `ProductionBonus(tree, mult)` — adds to `player.productionBonuses[tree]`. Read by D5 (economy) when computing gross industry.
- `SpecialEffectBonus(kind, amount)` — adds to `player.specialBonuses[kind]`. Read by D9 when resolving specials.

The `unlockedX` sets are checked by D5.7 (production queues can only build things the player has unlocked) and by D7.4 (well, actually D7.4 is structural-only — but D5.7 / P6 check unlocked sets before allowing a design to be built).

### D6.5 → Tech trade & gifts

Used by D11 (Diplomacy) when processing trade offers. Two operations:

**`canReceiveTech(recipient, techId) -> bool`**:
- Returns `true` iff recipient has all `tech.prereqs` (computed against `recipient.techs`).
- v1: prereq-strict. A player can't accept a tech they haven't earned the prereqs for.

**`receiveTech(recipient, techId, fromPlayer) -> (recipient', events)`**:
- Adds `techId` to `recipient.techs`.
- Applies effects (same as D6.4 acquisition).
- Emits `TechReceivedEvent { recipient, tech, fromPlayer }`.

Trade itself (the diplomatic offer: "I give you X tech for Y tech or Z bc") is D11's concern. D6.5 just provides the primitives.

## Dependency graph (within D6)

```
D6.1 (catalog) ──────────────────────────────────┐
  ↓                                               │
D6.2 (cost) ← reads D6.1, D3.3 (race affinities)  │
  ↓                                               │
D6.4 (acquireTech) ← reads D6.1, D1.2 (effects)  │
  ↓                                               │
D6.3 (research) ← reads D6.2, D6.4               ┘

D6.5 (trade) ← reads D6.1, D6.4 (acquisition logic)
```

Linear with D6.5 as a side branch that imports D6.4's acquisition helper.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Tech`, `TechEffect`, `Player`, `TechTree`, `BuildingKind`, `HullId`, `WeaponId`, `SpecialId`, `ShipStat` | D1.2 |
| D3 Races & Traits | `techAffinity(race, tree)`, `totalModifiers(race).researchProduction` | D3.2, D3.3 |
| D5 Economy | (no import) — D6 reads `Player.researchThisTurn` set by D5.4 | D5 |

D6 has minimal imports from other sections.

| Section | What it imports from D6 |
|---|---|
| D4 Turn Cycle | Calls D6 (research phase) once per turn |
| D5.7 Production | Reads `player.unlockedBuildings/Hulls/Weapons/Specials` for queue validation |
| D5 Economy | Reads `player.productionBonuses` for industry calculation |
| D7 Ship Design | (no direct call — D7 catalogs reference D6 tech ids but D7 owns the data) |
| D9 Space Combat | Reads `player.shipStatBonuses` for per-instance modifier application |
| D11 Diplomacy | Calls `canReceiveTech` / `receiveTech` for trade offers |
| D13 AI | Reads `Player.techs` and `unlockedX` sets to decide what to build/research |
| P5 Research Screen | Reads tech catalog and player progress (display only) |
| P6 Ship Design | Reads `player.unlockedX` to gate the picker |
| P7 Diplomacy | Reads tech catalog for trade offers |

## Quint-spec-sized leaves (the actual implementation units)

Three Quint files (D6.1+D6.2 share `techCatalog.qnt`, D6.3+D6.4 share `research.qnt`, D6.5 is `techTrade.qnt`):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/research/techCatalog.qnt` | D6.1 + D6.2 (catalog + cost calc) | `src/domain/research/techCatalog.ts` | ~250 (mostly catalog data) |
| `specs/research/research.qnt` | D6.3 + D6.4 (selection + accumulation + acquisition) | `src/domain/research/research.ts` | ~180 |
| `specs/research/techTrade.qnt` | D6.5 (trade primitives) | `src/domain/research/techTrade.ts` | ~120 |

Total ~550 lines of Quint. The catalog is data-heavy (~36 techs × ~6 lines each = ~220 lines). Logic is in the other ~330 lines.

No top-level orchestrator needed — D4 calls `applyResearch` directly, and D11 calls `canReceiveTech` / `receiveTech` directly.

## What "simple v1" looks like for D6

- **6 tech trees**, ~36 techs total. Roughly linear per tree with a few cross-tree prereqs.
- **One tech researched at a time** per player. (MoO matched this; some 4X games allow parallel research. v1: single.)
- **No partial carry-over** when switching research — accumulated points are lost. v2 could allow carry-over.
- **Prereq-strict trade** — recipients need their own prereqs. No "gift without prereq" mechanic in v1.
- **No random tech discovery** (libraries, ruins, native gifts). v1: all tech comes from research.
- **No "tech trading companies"** (some 4X games let you buy tech from neutral traders). v1: only direct player-to-player trade via D11.
- **Tech effects are limited to**: unlocks, stat bonuses, production bonuses, special-effect bonuses. No exotic effects (e.g., "discoverable ancient tech") in v1.

## Resolved decisions for D6

- **6 trees × 6 levels = 36 techs in v1.** Roughly MoO's shape.
- **Single active research** per player; switching loses accumulated points.
- **Prereq-strict trade** in v1.
- **No random tech discovery** in v1.
- **`TechEffect` ADT is open** — new effect types addable without changing consumers. (Same principle as D3.2's `Modifier` ADT.)
- **`unlockedX` sets on Player** are how unlocks propagate to D5.7 / P6.

## Open questions for D6

- **Tech cost scaling**: linear with level (cost = baseCost × level), or quadratic (cost = baseCost × level²)? MoO was approximately linear with a level multiplier. **Default v1: linear with level.**
- **Tech tree prereqs**: how many cross-tree prereqs? **Default v1: ~5 cross-tree prereqs** for variety. Final list in spec.
- **Tech discovery from exploration**: should exploring certain stars grant tech (e.g., Ancient stars)? MoO did this. v1: out of scope; deferred to v2 (would belong in D8 fleet movement upon arrival).
- **Negative effects**: do any techs have downsides (e.g., "Pollution tech boosts industry but reduces morale")? **Default v1: no**, all effects are pure positives. v2 could add tradeoffs.
- **Tech refund**: can a player "unlearn" a tech? **Default v1: no.** v2: maybe, for captured colonies or espionage.

No open questions block starting the D6 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D10 Ground Combat** (small focused section, depends on D1/D8/D9 which are all done).

D6's Quint spec can be written **now** — D6's only dependencies are D1 (done) and D3 (done).
