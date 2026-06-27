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
  - `Trait`: HyperExpansion, Militaristic, Toltec, Cybernetic, etc. (full list lives in D3)
  - `HullSize`: Small, Medium, Large, Huge
  - `WeaponKind`: Beam, Missile, Torpedo, Fighter, Bomber
  - `SpecialKind`: Shield, Armor, ECM, PointDefense, FuelTank, ColonyModule, etc.
  - `TechTree`: Weapons, Propulsion, Construction, Computer, Shields, ForceFields
  - `TreatyKind`: Peace, Alliance, TradePact, NAP, NonAggression, Tribute
  - `MissionKind`: StealTech, SabotageBuilding, SabotageShip, CounterEspionage, Observe
  - `BuildingKind`: Factory, ResearchLab, Farm, Market, Defense, Spaceport, Capital
  - `EventKind`: many variants (combat, diplomacy, discovery, etc.)
- **Game-wide constants**: `MAX_PLAYERS = 10`, `MIN_PLAYERS = 2`, galaxy size presets (`SMALL`, `MEDIUM`, `LARGE`, `HUGE` → star counts and map dimensions), `STARTING_TECH_LEVEL`, `MAX_TURN`, `INITIAL_POPULATION`, `COMBAT_MAX_ROUNDS`, etc.

Modeling decisions (locked in for v1):
- IDs are nominal newtypes over `int`, not strings. Saves space; speeds map lookups; survives catalog reordering.
- Enums are `int` with named constants (`const O = 1; const B = 2; ...`). Trivial to serialize and pattern-match.
- No bitflags. `Set<Trait>` is a list/set of `Trait` values, not a bitfield. Cleaner Quint, no overflow concerns.

### D1.2 → Entity types

What lives here (record types):

- **Catalog entries** (immutable reference data; values supplied by D3/D6/D7):
  - `Race` — id, name, traits, homeworldPlanetType, aiPersonalityHints, portraitRef.
  - `Hull` — id, name, size, slotCount, baseHp, baseSpeed, baseSpace.
  - `Weapon` — kind, baseDamage, range, shots, accuracy, spaceCost, prereqTech.
  - `Special` — kind, effect description (small enum: ShieldCapacity, ArmorReduction, ECMPenalty, PointDefenseRate, …).
  - `Tech` — id, name, tree, level, cost, prereqs, effects (ADT).
  - `Building` — kind, max level, prereqTech, baseEffect.

- **Mutable entities** (change during play):
  - `Player` — id, raceId, isAI, aiPersonality, homeworldPlanetId, knownStarIds, knownPlanetIds, techs (acquired), currentResearch (Option<TechId>), treasury, relations (Map<PlayerId, Relation>), spies, treaties.
  - `Planet` — id, starId, orbitIndex, type, size, richness, specials, population, maxPopulation, buildings, owner (Option<PlayerId>), garrison.
  - `Star` — id, x, y, spectralClass, planetIds, specials, owner (Option<PlayerId>; home star).
  - `Ship` — designId, count, hp (current/max), experience.
  - `Fleet` — id, ownerId, location (Either at StarId, or in-transit with from/eta/destination), ships, orders (e.g., destination, stance).
  - `ShipDesign` — id, ownerId, name, hullId, slotAssignments (list of (slotIndex, componentId)), specialIds, buildQueue.
  - `Relation` — level (int, -100..+100), modifiers (list).
  - `Treaty` — id, parties (Set<PlayerId>), kind, terms, signedOnTurn.
  - `TradeRoute` — id, fromPlanetId, toPlanetId, ownerId, income.
  - `Spy` — id, ownerId, locationPlanetId, mission (Option<Mission>), detectionRisk.
  - `Mission` — kind, targetPlayerId, startedOnTurn.
  - `Event` — id, turn, kind, payload (variant per kind).
  - `CombatEvent` — id, turn, starId, sideAId, sideBId, round, kind (Hit, Miss, Crit, ShipDestroyed, Retreat, BattleEnded), payload. Used by D9.

Modeling decisions (locked in for v1):
- Mutable entities are stored as **maps keyed by id** in `GameState`, not lists. O(1) lookup; clearer diffing for save games.
- IDs are assigned by **monotonically increasing counters** per type (`nextPlayerId`, `nextFleetId`, etc.) so saves stay compact and stable.
- **No stored aggregates**. `Player.totalFleetStrength` is a computed view function in the consuming section, never a field. Avoids drift; tests don't have to remember to keep aggregates in sync.
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
}
```

`GameOptions` is a small record holding the player's setup choices (galaxy size, race count, difficulty, etc.).

Modeling decisions (locked in for v1):
- `relations` is derived (rebuilt from `Player` relations and treaty-derived modifiers). Stored as a map keyed by ordered `(PlayerId, PlayerId)` pair; rebuilt each turn by the economy/diplomacy phase.
- `events` is an append-only list (cap at e.g. last 200 for memory; older events serialized to disk only on save).
- The map keys use Quint's `(p, q) -> Relation` syntax; ordered to canonicalize `(p, q)` with `p < q`.

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

No open questions remain that block starting the Quint spec for D1.1.

## Next step

Decomposition of D1 is complete. The next step (per [`ROADMAP.md`](../ROADMAP.md)) is **decomposing D2 (Galaxy Generation)** as a separate commit, then D3, then D7, then D8. After D8 is decomposed, write the D1 Quint spec files (`primitives.qnt`, `entities.qnt`, `gameState.qnt`) and proceed to D2's spec, etc.

D1's Quint spec can be written **now** without waiting for D2/D3/D7/D8 — D1 has no dependencies. Once D1's spec lands, the D2 spec can import from it.
