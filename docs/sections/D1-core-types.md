# Section D1 — Core Types & Constants

D1 is the **root** of the dependency graph. Every other section imports from it. This doc decomposes D1 into Quint-spec-sized leaves using the same workflow we used for the D9 pilot.

## Section contract

> **Core Types & Constants**: defines all data types shared across the domain (`Player`, `Race`, `Planet`, `Star`, `Fleet`, `Ship`, `Tech`, `Weapon`, `Treaty`, `Spy`, `Event`, `GameState`, …) and game-wide constants (max players, galaxy sizes, turn budget, etc.). Owns nothing but data; no behavior.

**Owns**: type definitions and game-wide constants. Pure value definitions.

**Does not own**:
- Any behavior (no functions beyond type aliases, enum definitions, and trivial `nextId` helpers).
- Catalog contents — those are written by D3 (races), D7 (hulls/weapons/specials), D6 (tech graph). D1 only defines the *shape* of those catalog entries.
- Any I/O, randomness, or side effects.

## Why a special approach for D1

D1 is **mostly mechanical** — it's a data-modeling exercise. The top-level decomposition in [`DECOMPOSITION.md`](../DECOMPOSITION.md#section-d1--core-types--constants) lists ten chunks (D1.1–D1.10). If we treat each as its own chunk, we get artificial boundaries: types cross-reference each other heavily (e.g., `Player` references `RaceId`, `Planet`, `Fleet`; `Fleet` references `PlayerId`, `StarId`, `ShipDesignId`).

Instead, we group D1's ten top-level chunks into **three natural Quint-file boundaries**:

| D1 doc chunk | Top-level chunks it covers | Quint file | Approx. lines |
|---|---|---|---|
| **D1.1 Primitives & constants** | D1.1 (IDs & enums), D1.9 (game-wide constants) | `specs/core/primitives.qnt` | ~120 |
| **D1.2 Entity types** | D1.2 (Race), D1.3 (Planet), D1.4 (Star), D1.5 (Fleet/Ship), D1.6 (Tech/Weapon), D1.7 (Treaty), D1.8 (Spy/Mission), plus event types and catalog types (hull/weapon/special/tech entries) | `specs/core/entities.qnt` | ~250 |
| **D1.3 Root GameState** | D1.10 (the big `GameState` record) | `specs/core/gameState.qnt` | ~100 |

Each file is a leaf-sized Quint spec. Each can be specified, tested, and implemented in one focused session.

## Recursive decomposition

### D1.1 → Primitives & constants

What lives here:
- **Identifier newtypes**: `PlayerId`, `StarId`, `PlanetId`, `FleetId`, `ShipDesignId`, `RaceId`, `HullId`, `WeaponKind`, `SpecialId`, `TechId`, `TreatyId`, `SpyId`, `EventId`, `BuildingId`. Newtypes over `int` so save-game IDs are stable.
- **Enumerated types** (numeric with named constants, for serialization size and Quint test readability):
  - `SpectralClass`: O, B, A, F, G, K, M
  - `PlanetType`: Terrestrial, Arid, Oceanic, Tundra, Barren, Volcanic, GasGiant, AsteroidBelt
  - `PlanetSpecial`: e.g., Fertile, MineralRich, UltraRich, Poor, Artifact, Native
  - `StarSpecial`: Ancient, PirateCache, GalacticCore
  - `Trait`: the v1 catalog is **16 traits** — Skilled, Versatile, Flight, Warrior, Militaristic, Subterranean, Stealthy, Cybernetic, Tolerant, Repulsive, Industrial, Creative, Honorbound, Telepathic, Erudite, Lithovore. (`HyperExpansion` is no longer in the ADT — it had no race assigned; v2 could re-add it as a fifth trait slot.)
  - `HullSize`: Small, Medium, Large, Huge
  - `WeaponKind`: Beam, Missile, Torpedo, **FighterBay**, **BomberBay** (renamed from `Fighter`/`Bomber` to avoid colliding with `Hull.Fighter`/`Hull.Bomber`; the bay launches fighters/bombers as spawned ships)
  - `SpecialKind`: Shield, Armor, ECM, PointDefense, FuelTank, ColonyModule, etc.
  - `TechTree`: Weapons, Propulsion, Construction, Computer, Shields, ForceFields
  - `TreatyKind`: Peace, Alliance, TradePact, NAP, NonAggression, Tribute
  - `MissionKind`: StealTech, SabotageBuilding, SabotageShip, CounterEspionage, ObserveFleet, ObserveTreasury, AssassinateLeader
  - `BuildingKind`: Factory, ResearchLab, Farm, Market, Defense, Spaceport, Capital
  - `EventKind`: many variants (combat, diplomacy, discovery, etc.)
  - `AiPersonalityTrait`: Aggressive, Expansionist, Technologist, Diplomat, Trader, Ruthless, Honorable (D3.4 race hints; mapped to D13.1 `StrategyKind` in `aiDefaultStrategy(race)`)
  - `StrategyKind`: Aggressive, Builder, Technologist, Diplomat, Balanced (D13.1; player-selected at game start; narrower than the trait-level hints in D3.4)
- **Mission duration table** (D12; used by D12.2's `duration(missionType)`):
  - `DURATION_STEAL_TECH = 3` turns
  - `DURATION_SABOTAGE_BUILDING = 1` turn
  - `DURATION_SABOTAGE_SHIP = 1` turn
  - `DURATION_OBSERVE_FLEET = 1` turn
  - `DURATION_OBSERVE_TREASURY = 1` turn
  - `DURATION_COUNTER_ESPIONAGE = 1` turn
  - `DURATION_ASSASSINATE_LEADER = 1` turn
- **Game-wide constants**: `MAX_PLAYERS = 10`, `MIN_PLAYERS = 2`, galaxy size presets (`SMALL = 24`, `MEDIUM = 36`, `LARGE = 56`, `HUGE = 80` → star counts and map dimensions), `STARTING_TECH_LEVEL`, `MAX_TURN = 200`, `INITIAL_POPULATION`, `COMBAT_MAX_ROUNDS = 50` (cap on tactical-combat length; exceeded = both sides disengage with no winner), `MAX_GROUND_ROUNDS = 30`, `COUNCIL_INTERVAL = 25`, `SPY_PRODUCTION_INTERVAL = 10`, `BC_PER_TAX_POINT = 1` (1 bc per tax point per turn — the v1 scale used by D5.5 and D11.3), `CONQUEST_VICTORY_THRESHOLD = 0.80` (D14.1; 80% of habitable planets, no rival above 10%), `MAX_EVENTS_IN_MEMORY = 200`, `SPY_UPKEEP_PER_TURN = 1` (bc per spy per turn; D5.5 reads from this), `PLANET_BASE_TAX_INCOME = 1` (bc per planet per turn at taxRate=0 baseline; D5.5 scales by taxRate), `RECENTLY_CONQUERED_TURNS = 5` (D5.6 morale `recentlyConquered` window; `Planet.conqueredOnTurn` carries the timestamp), **`TURN_WARNING_THRESHOLD = 10`** (D4.2 emits `ApproachingEndOfGameEvent` when `state.turn >= MAX_TURN - TURN_WARNING_THRESHOLD`), **`TAX_RATE_MIN = 0`**, **`TAX_RATE_MAX = 100`**, **`TAX_RATE_DEFAULT = 30`** (D5.5 slider range and v1 default; D5.6 reads `> 60` for the high-taxes penalty), **`BUILDING_LEVEL_MULTIPLIER(level) = level`** (D5.7: `~10 bc × level` cost formula is `state.buildings[kind].baseCost × BUILDING_LEVEL_MULTIPLIER(building.level)`), etc.

Modeling decisions (locked in for v1):
- IDs are nominal newtypes over `int`, not strings. Saves space; speeds map lookups; survives catalog reordering.
- Enums are `int` with named constants (`const O = 1; const B = 2; ...`). Trivial to serialize and pattern-match.
- No bitflags. `Set<Trait>` is a list/set of `Trait` values, not a bitfield. Cleaner Quint, no overflow concerns.

### D1.2 → Entity types

What lives here (record types):

- **Catalog entries** (immutable reference data; values supplied by D3/D6/D7):
  - `Race` — id, name, traits, homeworldPlanetType, aiPersonalityHints, portraitRef.
  - `Hull` — id, name, size, slotCount, baseHp, baseSpeed, baseSpace, baseWarpRange, baseCost, baseSupply, troopCapacity, prereqTech.
  - `Weapon` — id, kind (`Beam | Missile | Torpedo | FighterBay | BomberBay` — the `Fighter`/`Bomber` hull names are reserved for hulls; D7.2's fighter/bomber weapons are renamed to `FighterBay`/`BomberBay` to avoid the hull/weapon naming collision), baseDamage, range, shots, accuracy, spaceCost, prereqTech.
  - `Special` — kind, effect description (small enum: ShieldCapacity, ArmorReduction, ECMPenalty, PointDefenseRate, …).
  - `Tech` — id, name, tree, level, cost, prereqs, effects (ADT).
  - `Building` — kind, level (int, default 1; v1 cost = `baseCost × levelMultiplier(level)`, see D5.7), baseCost, buildIndustry, maxLevel, prereqTech, baseEffect (see `BuildingEffect` ADT below).

- **Mutable entities** (change during play):
  - `Player` — id, raceId, isAI, aiPersonality, aiMemory (`AIMemory`; persisted across turns; consumed by D13; default empty), homeworldPlanetId, knownStarIds, knownPlanetIds, techs (acquired), currentResearch (Option<TechId>), researchAccumulated (int; reset on tech acquisition and on switching research, per D6.3), treasury, relations (Map<PlayerId, Relation>), spies, treaties, intelLedger (`Map<PlayerId, PlayerIntel>`; written by D12.3), unlockedBuildings, unlockedHulls, unlockedWeapons, unlockedSpecials, shipStatBonuses (`Map<ShipStat, int>`), productionBonuses (`Map<TechTree, float>`), specialBonuses (`Map<SpecialKind, int>`), taxRate (int; clamped to `[TAX_RATE_MIN, TAX_RATE_MAX]`; default `TAX_RATE_DEFAULT`). *Note: `Player.score` is **not** a field — the score is stored on `GameState` as `state.score: PlayerId -> int` (see D1.3), recomputed once per turn by a D4 helper using D14.4's `computeScore` formula.*
  - `Planet` — id, starId, orbitIndex, type, size, richness, specials, population, maxPopulation, buildings, owner (Option<PlayerId>), garrison, conqueredOnTurn (Option<TurnId>; set on D10.5 conquest, consumed by D5.6 morale).
  - `Star` — id, x, y, spectralClass, planetIds, specials, owner (Option<PlayerId>; home star).
  - `Ship` (per-individual, not stack; D8.5's `Fleet.ships: List<{designId, count}>` is a *stack grouping* on the fleet, not on `Ship` — see "Ship semantics" below) — id, designId, hp (current/max), experience (v2-placeholder; see D9.6.3).
  - `Fleet` — id, ownerId, location (Either at StarId, or in-transit with from/eta/destination), ships (`List<{designId: ShipDesignId, count: int}>` — each entry is a *stack* of `count` ships of one design; HP and experience are tracked per-stack on each entry: `{designId, count, hpCurrent, hpMax, experience}`; partial-count splits across a single design are v2), orders (e.g., destination, stance), warpRange (int; cached from min over ship hulls; recomputed by D8.5 on merge/split).
  - `ShipDesign` — id, ownerId, name, hullId, slotAssignments (list of (slotIndex, componentId)), specialIds, buildQueue.
  - `Relation` — level (int, -100..+100), modifiers (list).
  - `Treaty` — id, parties (Set<PlayerId>), kind, terms, signedOnTurn.
  - `TradeRoute` — id, fromPlanetId, toPlanetId, ownerId, income (cached at end of D11.6 each turn; read by D5.5 the following turn), active (bool; false when either endpoint conquered or unowned, set by D10.5 / end-of-turn cleanup).
  - `Spy` — id, ownerId, skill (int, 1..10, increases with successful missions; default 3 for new spies), locationPlanetId, status (`Idle | OnMission(mission: MissionId) | Captured | Killed`; the `OnMission` payload references a `Mission` record by id), detectionRisk (float; passive chance modifier; rarely read directly because D12.5's detection formula uses `counterEspionageSkill(target)` — kept on the record for v2's per-spy detection modifiers).
  - `Mission` — id, spyId, kind (`StealTech | ObserveFleet | ObserveTreasury | SabotageBuilding | SabotageShip | AssassinateLeader | CounterEspionage`), targetPlayerId, assignedTurn, progressTurns (default 0; incremented by D12.2 each turn), difficulty (computed at assignment time by D12.1).
  - `Event` — id, turn, kind, payload (variant per kind).
  - `CombatEvent` (D9-only — per-shot playback stream; *not* stored in `state.events`; see "Event vs CombatEvent relationship" below) — id, turn, starId, sideAId (`FleetId`), sideBId (`FleetId`), round, kind (Hit, Miss, Crit, ShipDestroyed, Retreat, BattleEnded), payload. Used by D9.

**Ship semantics** (locked in for v1): `Ship` is per-individual. `Fleet.ships` is a `List<{designId, count, hpCurrent, hpMax, experience}>` stack grouping (one entry per design). HP and experience live on each stack entry, not on individual ships; D9.3.2's "remaining-hp-fraction" formula reads `hpCurrent / hpMax` *per stack entry*, and damage reduces `hpCurrent` for the targeted stack. v2 may split stacks to per-individual `Ship` records.

### `BuildingEffect` ADT (used by `Building.baseEffect`)

The D1.2 `Building.baseEffect` field is a sum type — each building kind produces one or more of these effects:

```
type BuildingEffect =
  | FoodOutput(int)              // per-turn food from a Farm
  | IndustryOutput(int)          // per-turn industry from a Factory
  | ResearchOutput(int)          // per-turn research from a Research Lab
  | TradeIncome(int)             // bonus trade income from a Market
  | EntertainmentBonus(int)      // +morale contribution
  | DefenseBonus(int)            // +ground combat morale/strength from a Defense building
  | IntelligenceCenter(int)      // +counter-espionage skill from an Intelligence Center
  | ShipRepairBonus(int)         // HP/turn repaired for ships docked here
```

A `Building` record carries a *list* of these (most buildings have one; some have multiple). Consumers pattern-match on the ADT to sum the contribution they care about. D5 reads `foodOutput/industryOutput/researchOutput/tradeIncome/entertainmentBonus`, D10 reads `defenseBonus`, D12 reads `intelligenceCenter`, D5.7 reads `shipRepairBonus` for the build fleet.

### Cross-entity helpers (computed view functions — no stored aggregates)

These are pure functions, declared in D1.2, consumed by the sections listed:

- **`totalFleetStrength(state, playerId) -> int`** — `Σ stats.attack × count` over every `Fleet.ships` entry owned by `playerId`. Declared here so D13's "weakest neighbor" target selection and D14.5's `PlayerStats.fleetStrength` have a single source of truth. (D9 reads the per-stack attack directly during combat; D13 and D14.5 read this aggregate.)
- **`homeworldBonus(planet, player) -> int`** — `+10 if planet.id == player.homeworldPlanetId else 0`. Consumed by D5.6's morale formula.
- **`playerTotalPlanets(state, playerId) -> int`** — `|{planet : state.planets | planet.owner == Some(playerId)}|`. Consumed by D14.1's conquest-share check.
- **`habitablePlanetIds(state) -> Set<PlanetId>`** — derived from `Planet.type` membership in `{Terrestrial, Oceanic, Arid, Tundra, Barren, Volcanic}` (matches D2.6's candidate-homeworld filter). Consumed by D14.1.

### `WeightVector` (D13.1)

```
type WeightVector = {
  military: float,        // 0..1
  economic: float,        // 0..1
  research: float,        // 0..1
  diplomatic: float,      // 0..1
  espionage: float,       // 0..1
}
```

The five fields sum to **1.0 by construction** (constructors enforce this; tests assert it). Used by D13.1's `Personality.weights`.

### `PlayerIntel` (D12)

```
type PlayerIntel = {
  observedTechs: Set<TechId>,
  observedFleets: Map<StarId, int>,      // star -> fleet size at last observation
  observedTreasury: Option<int>,         // last observed treasury value
  accuracyByField: Map<string, bool>,    // field -> "was this intel accurate?"
}
```

`Player.intelLedger: Map<PlayerId, PlayerIntel>` is what we know about each other player. Written by D12.3, read by D13 and P8.

Modeling decisions (locked in for v1):
- Mutable entities are stored as **maps keyed by id** in `GameState`, not lists. O(1) lookup; clearer diffing for save games.
- IDs are assigned by **monotonically increasing counters** per type (`nextPlayerId`, `nextFleetId`, etc.) so saves stay compact and stable.
- **No stored aggregates** on entities — see "Cross-entity helpers" above for the canonical computed views.
- Optional values use Quint's `Some(x) | None` pattern.
- Unions use Quint's variant types: `type FleetLocation = AtStar(StarId) | InTransit({ from: StarId, to: StarId, eta: int })`.

### D1.3 → Root GameState

The single big record holding the whole game. Composition only — every field is an entity collection or a scalar:

```ts
type GameState = {
  version: int,                  // save-game version (for migrations)
  turn: int,                      // current turn number
  rngSeed: int,                   // for deterministic replay
  options: GameOptions,           // galaxy size, race count, etc.

  // catalogs (immutable for the duration of a game)
  races: RaceId -> Race,
  hulls: HullId -> Hull,
  weapons: WeaponKind -> Weapon,
  specials: SpecialId -> Special,
  techs: TechId -> Tech,
  buildings: BuildingKind -> Building,

  // entities (mutable)
  players: PlayerId -> Player,
  stars: StarId -> Star,
  planets: PlanetId -> Planet,
  fleets: FleetId -> Fleet,
  designs: ShipDesignId -> ShipDesign,
  treaties: TreatyId -> Treaty,
  tradeRoutes: TradeRouteId -> TradeRoute,
  spies: SpyId -> Spy,

  // derived (rebuilt each turn or on demand)
  relations: (PlayerId, PlayerId) -> Relation,
  events: list<Event>,
  score: PlayerId -> int,            // derived; recomputed once per turn by D4 helper
}
```

`GameOptions` is a small record holding the player's setup choices (galaxy size, race count, difficulty, etc.).

Modeling decisions (locked in for v1):
- `relations` is derived (rebuilt from `Player` relations and treaty-derived modifiers). Stored as a map keyed by ordered `(PlayerId, PlayerId)` pair; rebuilt each turn by D11.1 (after drift, after treaty updates) so per-relation computed modifiers see the latest treaty state.
- `events` is an append-only list, **trimmed** (not rebuilt) at the end of each turn by D4.2 to the last `MAX_EVENTS_IN_MEMORY = 200` entries (D1.1). Older events are serialized to disk on save and re-merged on load.
- `score` is a derived per-player int recomputed by a D4 helper **once per turn** (after economy, before victory check). The D4 helper computes `score[player]` using the formula in D14.4 (`computeScore(player, state)` — pure function); both D11.5 (council vote count) and D14.4 (time-victory / end-of-game ranking) read from the cached `state.score` field. D14.4 owns the formula; D4 owns the recomputation side-effect. This avoids a D11↔D14 circular dependency.
- The map keys use Quint's `(p, q) -> Relation` syntax; ordered to canonicalize `(p, q)` with `p < q`.

### Event-kind index (cross-section)

Every `Event` flowing through `state.events` belongs to one of these kinds. This table is the source of truth for the universe of event kinds in v1 — each section adds new kinds in its own code, but consumers should not add events outside this list without an explicit decision.

| Event kind | Emitted by | Consumed by |
|---|---|---|
| `TurnStarted` | D4.2 | P12 (event log) |
| `TurnEnded` | D4.2 | P12 (event log) |
| `ApproachingEndOfGame` | D4.2 | P12 (turn summary) |
| `PopulationGrew` | D5.1 | P4 (planet view) |
| `Starvation` | D5.1 | P4 (planet view) |
| `PopulationStable` | D5.1 (Lithovore) | P4 |
| `RevoltRisk` | D5.1 (low morale) | P4 (v1: logged only; v2 flips ownership) |
| `FoodProduction` | D5.2 | (telemetry) |
| `IndustryProduction` | D5.3 | (telemetry) |
| `IncomeEvent` | D5.5 | P4, P9 |
| `QueueItemStarted` | D5.7 | P4 |
| `BuildingCompleted` | D5.7 | P4 |
| `ShipCompleted` | D5.7 | P4, P6 |
| `TechAcquired` | D6.4 | P5 |
| `NoResearch` | D6.3 | P5 |
| `TechReceived` | D6.5 | P5 |
| `Arrived` | D8.3 | P3 (star map) |
| `NoEncounter` | D8.4 | (telemetry) |
| `Encounter` | D8.4 | D4 → D9 |
| `PlanetColonized` | D8.6 (new colonization chunk; v1) | P3, P4, D14.5 (stats) — payload `{ planet, owner, colonistsLost }` |
| `SpyProduced` | D12.1 (every `SPY_PRODUCTION_INTERVAL = 10` turns per Intelligence Center) | P8 — payload `{ player, spy, locationPlanet }` |
| `CombatHit` / `CombatMiss` / `CombatCrit` / `ShipDestroyed` / `Retreat` | D9 | P10 (combat playback) |
| `BattleResolved` | D9.6 | D14.5 (stats), P10 — payload `{ star, winner: Option<FleetId>, loser: Option<FleetId>, outcome: Won | Drawn, turn }` (D9.6.5) |
| `GroundRound` | D10.2 | P11 (ground combat playback) |
| `Pillage` | D10.4 | P11 |
| `PlanetConquered` | D10.5 | P11, D14.5 (stats) |
| `InvasionRepelled` | D10.5 | P11 |
| `InvasionDraw` | D10.5 | P11 |
| `TreatySigned` | D11.4 | P7, P9 |
| `OfferRejected` | D11.4 | P7 |
| `WarDeclared` | D11.4 | P7, D8 (war state propagation) |
| `GalacticEmperorVictory` | D11.5 | D14.3 |
| `MissionResolved` | D12.2 | P8 |
| `IntelEvent` (TechObserved/FleetObserved/TreasuryObserved) | D12.3 | D13 (AI), P8, D11.3 |
| `Sabotage` (SabotageBuilding/SabotageShip) | D12.4 | P4, P8 |
| `LeaderAssassinated` | D12.4 | P8 (v1: logged only) |
| `SpyCaptured` | D12.5 | P8, P7 (relation penalty) |
| `AIPipeline` | D13.7 | (debug) |
| `GameEnded` | D14.5 | A4 (stops game loop) |

New event kinds added during the spec phase must be appended to this table.

### `Event` vs `CombatEvent` vs `FleetEvent` relationship

Three event-like types coexist in v1, with explicit ownership boundaries:

- **`Event`** (this table's kinds, in `state.events: list<Event>`) is the canonical "what happened this game" log — the one true persistence surface. Its `kind` field is one of the variants in the table above; the payload is `kind`-specific (e.g., `TechAcquired` carries `techId`, `BattleResolved` carries the payload documented above, `Starvation` carries `populationLost`).
- **`CombatEvent`** (D9.3–D9.5) is the per-shot/per-round playback stream consumed by P10 for tactical-combat animation. **`CombatEvent`s are *not* stored in `state.events`** — they're returned to the orchestrator (D4) as part of `combatResolution`'s event list, then handed directly to P10's combat stage. At the end of the battle, a single `BattleResolved` `Event` is appended to `state.events` to record the outcome.
- **`FleetEvent`** (D8.4) is the *internal* handoff type between D8 and D9 — it's the per-star arrival list that D8.4 returns from arrival detection, and D4 forwards each `Encounter` variant of `FleetEvent` to D9. **`FleetEvent`s are *not* stored in `state.events`** either — they're a transient pipeline type. The corresponding `Arrived`, `NoEncounter`, and `Encounter` `Event` kinds (in the table above) are what D8.4 *also* appends to `state.events` for the canonical log.

In short: `CombatEvent` and `FleetEvent` are pipeline types with narrow consumers; `Event` (the kind-indexed table) is the persistence surface. P10 reads `CombatEvent` from `combatResolution`'s return; everyone reads `Event` from `state.events`.

## Dependency graph (within D1)

```
D1.1 (primitives)
  ↓
D1.2 (entities)  ← imports IDs, enums, constants from D1.1
  ↓
D1.3 (GameState) ← imports everything from D1.1 and D1.2
```

Acyclic and linear. D1 has **no dependencies** on any other section.

## Cross-section dependencies (incoming)

Every other section imports from D1. The matrix:

| Section | What it imports from D1 |
|---|---|
| D2 Galaxy Generation | `StarId`, `PlanetId`, `Star`, `Planet`, `PlanetType`, `SpectralClass`, `PlanetSpecial`, `StarSpecial`, `GameState.options` |
| D3 Races & Traits | `RaceId`, `Trait`, `Race`, `Player`, `GameState.options` |
| D4 Turn Cycle | All entity types; the full `GameState` |
| D5 Economy | `Player`, `Planet`, `Building`, `Tech`, `TradeRoute`, `Event` |
| D6 Research & Tech Tree | `Tech`, `TechTree`, `TechEffect`, `Player`, `Event` |
| D7 Ship Design & Combat Stats | `Hull`, `Weapon`, `Special`, `ShipDesign`, `HullSize`, `WeaponKind` |
| D8 Fleet & Movement | `Fleet`, `Ship`, `Star`, `StarId`, `Player`, `FleetLocation` |
| D9 Space Combat | `Fleet`, `Ship`, `ShipDesign`, `CombatStats` (computed by D7.5), `CombatEvent` |
| D10 Ground Combat | `Planet`, `Fleet`, `Ship`, `Event` |
| D11 Diplomacy | `Player`, `Treaty`, `TreatyKind`, `Relation`, `TradeRoute`, `Event` |
| D12 Espionage | `Spy`, `Mission`, `MissionKind`, `Player`, `Planet`, `Event` |
| D13 AI Decision Logic | Reads from most entity types; outputs `Command[]` (defined in A3) |
| D14 Victory Conditions | All entities; `GameState` |

Because D1 is the root, no section can be specified or implemented until D1 is done.

## Quint-spec-sized leaves (the actual implementation units)

| Quint file | Implements | TS module |
|---|---|---|
| `specs/core/primitives.qnt` | D1.1 | `src/domain/core/primitives.ts` |
| `specs/core/entities.qnt` | D1.2 | `src/domain/core/entities.ts` |
| `specs/core/gameState.qnt` | D1.3 | `src/domain/core/gameState.ts` |

Each file:
- ~100–250 lines of Quint with `test` blocks.
- Imported by other Quint specs as needed.
- Mirrored by a Vitest test file under `src/domain/core/__tests__/`.

The TypeScript modules are pure type re-exports (`type PlayerId = number & { __brand: 'PlayerId' }`, etc.) — no runtime code beyond `nextId` helpers and constructors like `makePlayer(...)`.

## What "simple v1" looks like for D1

D1 is type definitions; there's not much to simplify. v1 just means we don't add fields "for the future":

- **No metadata blobs**. If we later want modding support, we add a `meta: map<string, string>` then.
- **No tagged-union proliferation**. A `FleetLocation` is `AtStar | InTransit`; we don't add `Docked`, `Boarded`, `Hyperspace` until we need them.
- **No polymorphic buildings**. `Building` has a fixed set of `BuildingKind`s. Custom building mods are v2.
- **No event payload polymorphism beyond what we use today**. We add event kinds as we encounter them.

The principle: model what's needed for v1; leave room (Quint's variant types are open, TS discriminated unions are easy to extend) to add more without breaking callers.

## Open questions for D1

- **Should IDs be `int` or `string`?** Decision: `int`. Stable across saves, smaller, faster. v2 can re-key if we ever need UUIDs for cloud sync (but multiplayer is explicitly out of scope).
- **Map vs. list for entity collections?** Decision: maps keyed by id. O(1) lookup; serialization-friendly.
- **Aggregates stored or computed?** Decision: computed. Stored aggregates drift from source-of-truth; Quint views make the computation cheap to express.
- **Event log cap?** Decision: cap last 200 events in memory; older events are flushed to disk on save. (Memory bound is a quality-of-implementation choice; could be configurable later.)
- **`COMBAT_MAX_ROUNDS` value?** Resolved: `50` rounds cap. If exceeded, both sides disengage (no winner). D9.6.5 emits `BattleResolved` with `outcome = Drawn` and `winner = loser = None`.
- **`MAX_TURN` value?** Resolved: `200` (cited from D14.4; propagated here for completeness).
- **Galaxy-size star counts?** Resolved: `SMALL=24, MEDIUM=36, LARGE=56, HUGE=80` (MoO-ish; tunable).
- **Per-spy upkeep?** Resolved: `SPY_UPKEEP_PER_TURN = 1` bc per spy per turn (D5.5 reads this; constant lives here).
- **Per-planet base tax income?** Resolved: `PLANET_BASE_TAX_INCOME = 1` bc per planet per turn at `taxRate = 0`; D5.5 scales by `taxRate / 100`.
- **`CONQUEST_VICTORY_THRESHOLD`?** Resolved: `0.80` (80% of habitable planets + no rival above 10%); owned here, consumed by D14.1.

No open questions remain that block starting the Quint spec for D1.1.

## Next step

Decomposition of D1 is complete. The next step (per [`ROADMAP.md`](../ROADMAP.md)) is **decomposing D2 (Galaxy Generation)** as a separate commit, then D3, then D7, then D8. After D8 is decomposed, write the D1 Quint spec files (`primitives.qnt`, `entities.qnt`, `gameState.qnt`) and proceed to D2's spec, etc.

D1's Quint spec can be written **now** without waiting for D2/D3/D7/D8 — D1 has no dependencies. Once D1's spec lands, the D2 spec can import from it.
