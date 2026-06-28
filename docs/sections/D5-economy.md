# Section D5 — Economy

D5 is the biggest single subsystem in the domain. Every turn, it computes food, industry, research, income, morale, population growth, and consumption for every colony. It's also the section where race traits, buildings, and the tax slider all interact. Because of its size, D5 is the integration test that proves the rest of the architecture hangs together.

## Section contract

> **Economy**: resolves the economy phase. Computes food, production, research, income, morale, population growth, and consumption for each colony. Owns the production queues that turn accumulated industry into completed ships and buildings.

**Owns**:
- Per-planet population growth and starvation.
- Per-planet food, industry, and research production from buildings.
- Per-player income, maintenance costs, and treasury changes.
- Per-planet morale calculation.
- Production queues: item selection, partial progress, completion events.

**Does not own**:
- Tech acquisition (D6 reads the gross research D5 produces; D6 decides what tech is researched).
- Ship design validation (D7) — D5.7 reads `Hull.baseCost` but does no design work.
- Fleet creation (D8) — D5.7 produces ships and adds them to a planet's garrison fleet (or a designated build fleet).
- AI economic decisions (D13) — D13 chooses what to build; D5.7 only executes the queue.

## Top-level chunks

Seven top-level chunks from `DECOMPOSITION.md`:

- **D5.1 Population growth & starvation**.
- **D5.2 Food production & surplus**.
- **D5.3 Industry production**.
- **D5.4 Research generation (before tax slider)**.
- **D5.5 Income & maintenance costs**.
- **D5.6 Morale calculation**.
- **D5.7 Building & ship production queues**.

The chunks are called in roughly this order within the economy phase (D4 calls D5 once per turn, and D5 internally orders its sub-phases):

```
economy(s, cmds, ctx) =
    foodIndustryResearch(s, ctx)               // D5.2 + D5.3 + D5.4 (gross production)
  .then(morale(s, ctx))                       // D5.6
  .then(population(s, ctx))                   // D5.1 (uses food + morale)
  .then(incomeAndMaintenance(s, cmds, ctx))   // D5.5 (uses tax slider)
  .then(productionQueues(s, cmds, ctx))       // D5.7 (consumes industry)
```

The "tax slider" is a per-player command (`SetTaxRate(int)`) applied in D5.5. D5.4 produces *gross* research; D5.5 splits it into net research + income via the slider.

## Recursive decomposition

### D5.1 → Population growth & starvation

Per-planet function: `applyPopulation(planet, race, food, morale, ctx) -> (planet', events)`.

Inputs:
- `planet` — current population, maxPopulation, type, size.
- `race` — for trait modifiers (e.g., Lithovore ignores food, growth modifier).
- `food` — from D5.2.
- `morale` — from D5.6.

Logic:
- If `food >= 0`: population grows. Growth rate = `baseGrowth * (1 + morale/100) * race.totalModifiers.growthMod`, capped at `planet.maxPopulation`.
- If `food < 0` and race is Lithovore: no starvation (Lithovore eats minerals, not food). Emit `PopulationStableEvent`.
- If `food < 0` (non-Lithovore): population starves. Loss rate = `|food| / 2`. Emit `StarvationEvent { planet, lostPopulation }`.
- If `morale < 20` for several consecutive turns: emit `RevoltRiskEvent { planet, morale }`. v1: the event is logged only — ownership does not flip. (D5.6 just computes the morale value; D5.1 owns the threshold check and event emission.) v2 adds the actual ownership flip.

Emits: `PopulationGrewEvent`, `StarvationEvent`, `PopulationStableEvent`, `RevoltRiskEvent`.

Pure per-planet; trivially testable.

### D5.2 → Food production

Per-planet function: `foodProduction(planet, race, buildings, ctx) -> int`.

```
foodProduction = Σ(building.baseEffect)            // sum across all buildings on the planet
                   .filter(isFoodOutput)            // pattern-match the BuildingEffect ADT
                   .value                           // extract the int
                 × planet.size (modest scaling)
                 × planet.type.agricultureBonus
                 × race.totalModifiers.foodProduction
```

Where `isFoodOutput` and `value` are pattern-match helpers over the `BuildingEffect` ADT (D1.2). Race multipliers come from D3.2's `Modifier` ADT — these are distinct mechanisms and never overlap.

If the planet has `Native` special (from D2), no farming is possible (natives reject it). Food production is 0 for `Native` planets.

Emits: optional `FoodProductionEvent { planet, amount }` (debug/telemetry; UI can derive this).

### D5.3 → Industry production

Per-planet function: `industryProduction(planet, race, buildings, morale, ctx) -> int`.

```
industryProduction = Σ(building.baseEffect)
                       .filter(isIndustryOutput)
                       .value
                     × planet.size
                     × planet.type.industryBonus
                     × race.totalModifiers.industryProduction
                     × (morale / 100)    // low morale hurts production
```

If the planet is `Native` or `Artifact`, the industry is capped (natives/artifacts resist colonization-industry). v1: capped at 50% until colonized (no native resistance after colonization).

Emits: optional `IndustryProductionEvent`.

### D5.4 → Research generation (before tax slider)

Per-player function: `grossResearch(player, planets, race, ctx) -> int`.

```
grossResearch = Σ planet.researchLabCount × lab.baseOutput
                × planet.size
                × race.techAffinity(tree, currentResearch)   // D3.3
                × race.totalModifiers.researchProduction
```

This is **gross** — the tax slider (applied in D5.5) will subtract some of this and turn it into income.

The `currentResearch` field on `Player` is set by D6 (when the player picks what to research). D5.4 reads it but doesn't change it.

### D5.5 → Income & maintenance costs

Per-player function: `netIncome(player, planets, grossResearch, ctx) -> (net, events)`.

```
grossIncome = Σ planet.baseTaxIncome                       // per-planet flat income: PLANET_BASE_TAX_INCOME × (taxRate / 100)
            + Σ tradeRoutes(player).income                 // cached from D11.6 last turn
            + floor(player.grossResearch × (taxRate / 100))  // slider

// per-ship supply cost = Σ ship.count × shipDesign(state, ship).hull.baseSupply (D7.1)
fleetSupplyCost = sum over all fleets owned by player of Σ ship.count × shipDesign(state, ship).hull.baseSupply

// per-spy upkeep = |player.spies| × SPY_UPKEEP_PER_TURN  (D1.1 constant; default 1)
spyUpkeep = |player.spies| × SPY_UPKEEP_PER_TURN

maintenance = Σ(planet.buildings.baseEffect).filter(isMaintenance).value  // per-building upkeep (v1: no BuildingEffect.Maintenance variant — sum is 0)
            + fleetSupplyCost
            + spyUpkeep
            + subjugationTribute(player)                   // 10% of gross income to suzerain (D11)

netIncome = grossIncome - maintenance
```

**Trade income** is computed by D11.6 at end-of-turn and cached on `TradeRoute.income`. D5.5 reads the cached value. This means the *current* turn's economy sees *last* turn's trade income — acceptable because trade routes are static across turns in v1 (no auto-trade; routes are player-created).

**Subjugation tribute** (D11): if player A is subjugated to player B, 10% of A's gross income is transferred to B each turn as a maintenance line. Mechanism: `subjugationTribute(player) = 0.10 × grossIncome(player)` if `player` has a suzerain.

**Research agreement transfer** (D11): if A and B have a research agreement, the gross research each generates in D5.4 is split 50/50 between them after D6.3 has deducted the cost of any tech-acquired this turn. The transfer is applied **inside D5.5** as a maintenance line for the paying side and a positive line for the receiving side (so the total is conserved). This is the implementation chunk for the research-agreement mechanism — D11's chunk list does not add a new section for it; D5.5 reads the treaties map and applies the transfer inline. See the PLANNING.md decision log (2026-06-05 entry on D11.6→D5.5 caching and the 2026-06-05 entry on the 50/50 split) for the rationale.

Tax slider semantics:
- `taxRate ∈ [0, 100]`. Default 30.
- `taxRate = 0`: 0 income from research; full research points.
- `taxRate = 100`: all research → income; no research.
- High `taxRate` reduces `morale` (D5.6 reads `taxRate` as a penalty).

The function updates `player.treasury += netIncome` and emits `IncomeEvent { player, gross, maintenance, net, treasury }`.

If `treasury < 0` after the update, the player is in debt. Debt handling in v1: each turn of debt reduces `morale` globally (extra penalty in D5.6).

### D5.6 → Morale calculation

Per-planet function: `morale(planet, race, buildings, player, ctx) -> int`.

```
baseMorale = 50   // neutral

modifiers = sum([
    race.totalModifiers.morale,                                       // flat bonus from race traits (D3.2)
    Σ(planet.buildings.baseEffect).filter(isEntertainmentBonus).value, // building EntertainmentBonus effects (D1.2)
    -warWeariness(player, planet),                                    // 0..-30 if at war
    -highTaxes(player.taxRate),                                       // 0..-20 if taxRate > 60
    -debtPenalty(player.treasury),                                    // 0..-20 if treasury < 0
    +homeworldBonus(planet, player),                                  // +10 if this is the homeworld
    +recentlyConquered(planet),                                       // -20 if conquered last 5 turns (revolt risk)
])

morale = clamp(baseMorale + modifiers, 0, 100)
```

D5.6 *computes* the morale value. D5.1 *consumes* it: when `morale < 20` for several consecutive turns, D5.1 emits `RevoltRiskEvent`. (D5.6 itself emits no events.)

Morale feeds back into D5.3 (industry production scaled by morale/100) and D5.1 (population growth scaled by morale).

### D5.7 → Building & ship production queues

Per-planet function: `applyProductionQueue(planet, availableIndustry, race, ctx) -> (planet', events)`.

The planet's queue holds **at most one item** in v1 (a `QueueItem` is
`BuildBuilding(BuildingKind)` or `BuildShip(ShipDesign, count)`):

```
// Single-item queue. Empty queue = nothing to produce this turn.
if planet.queueItem == None: return (planet, [])

progress = planet.queueProgress + availableIndustry
cost = queueItemCost(planet.queueItem, state, planet)   // see below

if progress >= cost:
    emit QueueItemCompletedEvent
    applyItem(planet, queueItem)   // adds Building or Ship
    queueProgress = 0
    queueItem = None               // v1: queue empties on completion; player must issue a new SetQueue command to produce more
else:
    queueProgress = progress
```

`queueItemCost`:
- `BuildBuilding(kind)` → `BuildingKind.cost(kind)` (lookup table in D7 spec; ~10 bc × level by default).
- `BuildShip(designId, count)` → `count × design(state, designId).cost` (`CombatStats.cost` from D7.5).

The completed ship is added to a designated build fleet at the planet
(creating it if it doesn't exist; ownership is the player).

If `availableIndustry < cost`, the item stays in the queue with
accumulated progress (progress is preserved across turns until either
the item completes or the player issues a `ClearQueue` /
`SetQueue(newItem)` command — which resets `queueProgress = 0`).

Emits: `BuildingCompletedEvent`, `ShipCompletedEvent`, `QueueItemStartedEvent`.

## Dependency graph (within D5)

```
D5.2 (food)         ┐
D5.3 (industry)     │ pure functions, no internal deps
D5.4 (research)     ┘

D5.6 (morale) ← reads taxRate from Player, warWeariness state, treasury

D5.5 (income) ← reads D5.4's gross research, applies tax slider

D5.1 (population) ← reads D5.2's food, D5.6's morale

D5.7 (queues) ← reads D5.3's industry, queue state, D7.5 ship cost
```

Linear, with D5.6 (morale) feeding into D5.3 (industry) and D5.1 (population) — those two need D5.6 computed first.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Planet`, `Player`, `Race`, `Building`, `BuildingKind`, `TradeRoute`, `Spy`, `Fleet`, `Modifier` | D1.2 |
| D2 Galaxy Generation | `Planet.specials` (Native, Artifact affect production) | D2.5 |
| D3 Races & Traits | `race.totalModifiers`, `race.techAffinity` | D3.2, D3.3 |
| D6 Research | (no import — D5.4 reads `Player.currentResearch` set by D6) | D6 |
| D7 Ship Design | `Hull.baseCost`, `ShipDesign.cost` (= `CombatStats.cost`) for queue cost | D7.5 |
| D11 Diplomacy | `TradeRoute` income (read-only) | D11 |
| D13 AI | (no import — D13 sets the queue contents, D5.7 executes them) | D13 |

D5 has the most dependencies of any single domain section (after D4).

| Section | What it imports from D5 |
|---|---|
| D4 Turn Cycle | Calls D5 once per turn |
| D6 Research | Reads `grossResearch` from D5.4 (already in `Player.researchThisTurn`) |
| D9 Space Combat | Reads `Fleet.supplyCost` from D5.5 (no direct call) |
| D14 Victory | Reads `Player.treasury`, `Player.totalProduction` for score |
| A1 Store | Calls D5 chunks |
| P4 Planet Screen | Displays D5 outputs (food/industry/research/morale) |
| P5 Research Screen | Reads research progress (D6 ultimately) |
| P9 Production Summary | Reads D5.7 queue state |
| P12 Council | Reads `Player.totalProduction` for vote weight |

## Quint-spec-sized leaves (the actual implementation units)

Six Quint files, mirroring the chunk boundaries:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/economy/population.qnt` | D5.1 | `src/domain/economy/population.ts` | ~120 |
| `specs/economy/foodAndIndustry.qnt` | D5.2 + D5.3 (shared per-planet production pattern) | `src/domain/economy/foodAndIndustry.ts` | ~180 |
| `specs/economy/research.qnt` | D5.4 | `src/domain/economy/research.ts` | ~100 |
| `specs/economy/income.qnt` | D5.5 | `src/domain/economy/income.ts` | ~150 |
| `specs/economy/morale.qnt` | D5.6 | `src/domain/economy/morale.ts` | ~120 |
| `specs/economy/queues.qnt` | D5.7 | `src/domain/economy/queues.ts` | ~150 |
| `specs/economy/economy.qnt` | top-level `economy(s, cmds, ctx)` orchestrator calling the chunks in order | `src/domain/economy/index.ts` | ~80 |

Total ~900 lines of Quint. D5 is the largest single section. The orchestrator at the bottom is small (just threads state through the chunks).

We split D5.2 and D5.3 into one file because they share the per-planet production pattern. Each is a small function within that file.

## What "simple v1" looks like for D5

- **Linear population growth** (no exponential, no logistic curve).
- **Single-item queue per planet** (MoO matched this). v2 supports multi-item queues.
- **One tax slider** (income vs research split). v2 adds sliders for "spy spending", "trade", etc.
- **No revolt mechanics** in v1 — D5.1 emits `RevoltRiskEvent` when morale is low, but ownership doesn't actually flip. v2 adds full revolt.
- **No black-market / piracy income**. Trade routes only.
- **Debt is a morale penalty**, not a treasury floor. Players can go negative.
- **Native/Artifact planets resist industry** (50% cap) until colonized.
- **No planet-buster economics** — D5 just feeds the rest of the game.

## Resolved decisions for D5

- **Population grows linearly** (capped at maxPopulation). No logistic curve in v1.
- **Single-item queue per planet.** Multi-item queues are v2.
- **One tax slider** splits research vs income. Default 30.
- **Debt reduces morale** rather than capping treasury at zero.
- **Native/Artifact planets have capped industry** until colonized.
- **Morale feeds back into industry and population growth** (low morale = low output).
- **D5.7 produces ships into a designated build fleet at the planet** (creates one if missing).
- **Revolt events are emitted by D5.1** when morale is low; D5.6 just computes the morale value. Events are logged but don't flip ownership in v1.
- **Lithovore ignores food** (no starvation, but no growth bonus either).
- **Trade income is cached on `TradeRoute.income` by D11.6** at end of turn; D5.5 reads the previous turn's cached value.
- **Subjugation tribute (10% of gross income to suzerain)** is applied as a maintenance line in D5.5.
- **Research agreement transfers (50/50 split)** happen between D6.3 and D5.5 in the same turn.

## Open questions for D5

- **Max population formula**: how is `planet.maxPopulation` computed? **Resolved v1: `maxPop = planet.size × sizeFactor(planet.type)`**, where the `sizeFactor` table is `Terrestrial=2, Oceanic=2, Arid=1.5, Tundra=1.5, Barren=1, Volcanic=1, GasGiant=0.5, AsteroidBelt=0.25` (planets with `size=10` max out at 20 population on Terrestrial/Oceanic). Tune during playtesting.
- **Building catalog**: what buildings exist? MoO had ~10. **Resolved v1: 7 buildings** — `Factory, ResearchLab, Farm, Market, Defense, Spaceport, Capital` — each with one `BuildingEffect` from the ADT in D1.2. Per-building costs are a small lookup table (`~10 bc × level`); the exact list/effects live in the D7 spec.
- **Supply cost per fleet**: how much does each ship cost in maintenance? **Resolved v1: `cost = Σ ship.count × shipDesign(state, ship).hull.baseSupply`**. `baseSupply` is a `Hull` field (D7.1) with v1 values: Fighter=1, Bomber=1, Destroyer=1, Cruiser=3, Battleship=8, Dreadnought=12, Titan=20, Transport=2, Colony Ship=2, Doomship=50. Per-fleet total supply feeds D5.5's `fleetSupplyCost`.
- **War weariness formula**: how much does each war reduce morale? **Resolved v1: `-10` per active war**, capped at `-30`. Easy to tune later.
- **Recently-conquered penalty duration**: how many turns does the -20 penalty last? **Resolved v1: 5 turns.** Driven by `Planet.conqueredOnTurn` (D1.2) and a `RECENTLY_CONQUERED_TURNS = 5` constant in D1.1.
- **Per-spy upkeep?** Resolved: `SPY_UPKEEP_PER_TURN = 1` bc per spy per turn (D1.1). `spyUpkeep = |player.spies| × SPY_UPKEEP_PER_TURN` (D5.5).
- **Per-planet base tax income?** Resolved: `PLANET_BASE_TAX_INCOME = 1` bc per planet per turn at `taxRate = 0` baseline (D1.1). D5.5 scales by `taxRate / 100`.

No open questions block starting the D5 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D6 Research & Tech Tree**.

D5's Quint spec can be written **now** — D5's dependencies (D1, D2, D3, D7) are all decomposed. D5 doesn't need D6/D11/D13 to be specified first.
