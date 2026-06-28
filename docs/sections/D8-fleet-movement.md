# Section D8 — Fleet & Movement

D8 moves fleets around the galaxy and resolves what happens when they arrive. This section closes the loop with D9: by the time D9 sees a star, every player has been collapsed into a single fleet (auto-merge), and sides have been paired for combat.

## Section contract

> **Fleet & Movement**: given movement orders and the galaxy state, advance fleets, merge same-owner co-located fleets, and emit encounter events for D9 to resolve.

**Owns**:
- Fleet order types (data) and their validation.
- Pathfinding / distance computation (Euclidean in v1; warp-lane topology in v2).
- Movement: advancing in-transit fleets to their destinations.
- Arrival: detecting arrivals, auto-merging same-owner fleets at each star, pairing sides for encounter detection.
- Pure fleet operations: merge and split.

**Does not own**:
- Combat resolution (D9 reads the encounter events D8 emits).
- Ship design (D7) — D8 reads `Hull.baseWarpRange` per ship but does no design work.
- Production queues (D5.7) — D8 doesn't build ships.
- AI movement decisions (D13) — D13 issues the orders D8 executes.
- Anything UI-side (P3 reads fleet state; P6 issues move/split commands).

## A note on the chunk list

`DECOMPOSITION.md` lists D8.2 as "Pathfinding (warp lanes)". Original MoO had **no warp lanes** — warp was Euclidean distance with a per-fleet range limit. We keep that for v1 and reframe D8.2 as **distance and range**. Warp-lane topology is a v2 chunk.

## Top-level chunks

Five top-level chunks lifted from `DECOMPOSITION.md` (D8.2 reframed):

- **D8.1 Fleet orders** — destination, stance (defend, attack, explore); command validation.
- **D8.2 Distance & range** — Euclidean distance between stars; per-fleet warp range from min `Hull.baseWarpRange`.
- **D8.3 Movement** — advance in-transit fleets one turn; compute new arrivals.
- **D8.4 Arrival & encounter detection** — auto-merge same-owner fleets at each star; pair sides for D9.
- **D8.5 Fleet merge & split** — pure operations on fleet records (used by D8.4 and player commands).
- **D8.6 Colonization** (added in v1 to address REVIEW-NOTES 10.12) — when a fleet with a Colony Ship (`Hull.Colony Ship` or any ship with `Special.ColonyModule`) and ≥1 colonist reaches an *uninhabited* planet at a star with no other fleet, the planet's `owner` flips to the player's id, `population` is seeded to `INITIAL_POPULATION`, and `PlanetColonizedEvent` is emitted. If multiple colonizers arrive at the same planet in the same turn, the first-arriving fleet claims it.
- **D8.7 Retreat destination helper** — `pickRetreatDestination(fleet, state, rng) -> StarId`. Pure function shared by D9.5.4 (space-combat retreat) and D10.5 (invasion retreat). Selects the nearest star in `state.stars` whose Euclidean distance from the fleet's current star is `≤ fleet.warpRange`; ties broken randomly via the deterministic `ctx.rng`. Returns the chosen `StarId`; the caller (D9 or D10) emits a `RetreatOrder { fleet, destination }` event. If no star is in range, returns `None` (the fleet is destroyed).

## Recursive decomposition

### D8.1 → Fleet orders

Order types are pure data, validated before being applied:

```
type FleetOrder =
  | MoveTo(fleet: FleetId, destination: StarId, stance: Stance)
  | Split(fleet: FleetId, newFleet: FleetId, shipIndices: Set<int>)
  | SetStance(fleet: FleetId, stance: Stance)
  | CancelOrder(fleet: FleetId)

type Stance = Defend | Attack | Explore
```

Validation (D8.1's pure function): `validateOrder(order, state) -> Result<Unit, OrderError>`.

Rules:
- `MoveTo`: destination is known; fleet is currently `AtStar` (not already in transit).
- `Split`: fleet has more *design slots* than the requested split (must leave ≥1 slot in each result); new fleet id is unused. v1 splits only by whole slots — partial-count splits from a single design are not supported.
- `SetStance`: always valid.
- `CancelOrder`: only valid for fleets in transit (no-op for fleets at a star).

`OrderError` is a small ADT — `UnknownFleet`, `FleetInTransit`, `EmptySplit`, etc.

Stance semantics:
- `Defend` — fleet engages if attacked; doesn't auto-trigger combat on arrival.
- `Attack` — fleet engages any enemy at destination on arrival.
- `Explore` — fleet engages only if explicitly attacked.

**v1 status**: stance is *reserved data* only. No code in v1 reads the stance field. D9.2 (targeting AI) will consume stance in v2 to grant initiative bonuses or first-strike preference to `Attack` fleets. In v1, the field exists so the data model is forward-compatible and so D8.1's validation can enforce "stance is a valid enum value" without further changes when v2 turns it on.

### D8.2 → Distance & range

Pure functions:

```
distance(from: Star, to: Star): int          // Euclidean, in parsecs (integer)
inWarpRange(fleet: Fleet, to: Star): bool    // distance ≤ fleet.warpRange
```

`fleet.warpRange` is computed as the **minimum** `Hull.baseWarpRange` across all ships in the fleet (a fleet moves as slow as its slowest ship). Read once per movement order; cached on the fleet to avoid recomputation per turn.

v1: no warp-lane topology. v2 adds a `Map<StarId, Set<StarId>>` reachable-from-each-star and changes the distance function to "follows lanes."

(Note: `CombatStats.range` was previously the source of warp range. That field has been removed — `Hull.baseWarpRange` is now the canonical per-ship warp range. See D7.5 and the resolved decisions log.)

### D8.3 → Movement

Pure function: `advanceFleets(state, currentTurn) -> (newState, arrivedEvents)`.

For each fleet with `location = InTransit({ from, to, eta })`:
- If `currentTurn >= eta`, set `location = AtStar(to)` and emit `ArrivedEvent { fleet, at: to, onTurn: currentTurn }`.

ETA is **locked at order issue**: range upgrades during transit do not change an in-flight fleet's arrival turn. ETA recompute is deferred to v2.

That's it for v1. v2 adds:
- Mid-transit rerouting (player changes destination mid-flight).
- Fleet-to-fleet intercept (combat during transit, not just at stars).
- Range-upgrade ETA recompute (with a clear "upgrade applies on next turn" rule).

### D8.4 → Arrival & encounter detection

This is the most-touched D8 chunk because it's the bridge to D9. Three phases:

**Phase A — Collect arrivals and current occupants.**
For each star S, compute `fleetsAtS` = `(arrivedThisTurn at S) ∪ (fleets already AtStar(S))`.

**Phase B — Auto-merge same-owner fleets.**
For each owner present at S, collapse their fleets into one. Use D8.5's `mergeFleets(fleets)` helper (sum ships, take min HP, take slowest speed, keep first fleet's id). The result is **one fleet per player-side at S**.

**Phase C — Pair sides for encounters.**
Let `sides = distinctOwners(fleetsAtS)`.
- If `|sides| < 2`: no encounter; emit `NoEncounterEvent`.
- If `|sides| == 2`: emit one `EncounterEvent { star: S, sideA: side[0], sideB: side[1] }`.
- If `|sides| >= 3`: shuffle `sides` randomly, then pair adjacent (0,1), (2,3), (4,5), …. For each pair, emit one `EncounterEvent`. Any unpaired (last) side emits `NoEncounterEvent` for this turn.

The shuffle is part of the pure function — it takes an `rng: () => number` parameter (per Architecture Principle 1) so tests can supply a fixed seed. **No `Math.random()` in the domain.**

The output of D8.4 is a list of events that D4 (Turn Cycle) hands to D9:

```
type FleetEvent =
  | Arrived(fleet: FleetId, at: StarId, onTurn: int)
  | NoEncounter(star: StarId)
  | Encounter(star: StarId, sideAId: FleetId, sideBId: FleetId)   // naming matches D1.2's CombatEvent (sideAId/sideBId)
```

The "one fleet per player-side at the star" guarantee is what D9's contract requires. D8.4 is the only place this guarantee is produced.

### D8.5 → Fleet merge & split

Pure operations on `Fleet` records:

**Merge**: `mergeFleets(fleets: List<Fleet>) -> Fleet`.
- `id` = first fleet's id (canonical owner).
- `owner` = same for all (precondition).
- `location` = same for all (precondition).
- `ships` = sum of all ships per `designId`, then per-ship `count = sum`, `hp = min` across merged ships (worst-case HP is conservative), `experience = max`.
- `warpRange` = min across all ships.
- `orders` = empty (no in-flight orders; player reissues).
Returns a single Fleet; the input fleets are removed from `GameState.fleets`.

**Split**: `splitFleet(fleet: Fleet, newId: FleetId, shipIndices: Set<int>) -> (Fleet, Fleet)`.
- Splits the fleet into two fleets at the same location.
- `shipIndices` references **ship-design slots** in the original fleet's `ships: List<{designId: ShipDesignId, count: int}>` list (one index per slot, not per individual ship). For example, a fleet `[{designId: Fighter, count: 10}, {designId: Cruiser, count: 3}]` has two slots (indices 0 and 1); to send the Cruisers to the new fleet you'd split with `{1}`. To split a partial count from one slot you'd need a separate `SplitFromSlot(designId, count)` operation — v1 doesn't ship that; v1 splits only by whole slots.
- Ships in the named slots go to the new fleet; the rest stay.
- Preconditions: `shipIndices` is non-empty and a strict subset of `{0..|ships|-1}`; newId is unused; each split-off slot has at least one ship (always true for whole-slot splits).
- Both result fleets share `location`, `owner`, `orders = empty`.

These operations are pure functions; they don't update `GameState` themselves. The caller (D8.4 for merge; A4 Turn Manager for player-initiated split) applies the result.

## Dependency graph (within D8)

```
D8.1 (orders) ──────────────────────────────────────┐
  ↓                                                  │
D8.2 (distance & range) ── reads D7.1 hulls          │
  ↓                                                  │
D8.3 (movement) ── reads D8.1, D8.2                 │
  ↓                                                  │
D8.5 (merge & split ops) ───────────────────────────┤
  ↓                                                  │
D8.4 (arrival + encounter) ── reads D8.3, D8.5 ─────┘
                                                 │
                                                 ▼
                                       emits Encounter events
                                       to D4 → D9
```

Strictly linear except D8.5 which is shared with the orchestrator.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Fleet`, `Ship`, `ShipDesign`, `Star`, `FleetLocation`, `FleetId`, `StarId`, `FleetOrder`, `Stance`, `FleetEvent` | D1.1, D1.2 |
| D2 Galaxy Generation | `Star` positions for distance computation | D2.2 |
| D7 Ship Design | `Hull.baseWarpRange` (per ship), `Hull.baseSpeed` (for fleet speed display; not used in v1 movement) | D7.1 |

D8 has no other dependencies.

| Section | What it imports from D8 |
|---|---|
| D4 Turn Cycle | `advanceFleets`, `arrivalEvents`; runs D8 once per turn |
| D9 Space Combat | Reads `EncounterEvent`s to know which sides fight at which star |
| D10 Ground Combat | (indirectly) — invasion requires arriving at a defended planet |
| D13 AI | Issues `MoveTo` orders |
| A4 Turn Manager | Calls D8.3/D8.4 between phases |
| P3 Star Map | Reads fleet locations, animates in-transit fleets |
| P6 Fleet Screen | Issues player `MoveTo`/`Split` commands |

D8 is the most-imported section among the "movement" subsystems.

## Quint-spec-sized leaves (the actual implementation units)

Four Quint files, no top-level orchestrator (D4 calls D8.3 then D8.4 in sequence):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/fleet/orders.qnt` | D8.1 | `src/domain/fleet/orders.ts` | ~100 |
| `specs/fleet/movement.qnt` | D8.2 + D8.3 | `src/domain/fleet/movement.ts` | ~120 |
| `specs/fleet/fleetOps.qnt` | D8.5 | `src/domain/fleet/fleetOps.ts` | ~100 |
| `specs/fleet/arrival.qnt` | D8.4 | `src/domain/fleet/arrival.ts` | ~150 |
| `specs/fleet/retreat.qnt` | D8.7 | `src/domain/fleet/retreat.ts` | ~50 |

Total ~520 lines of Quint.

`arrival.qnt` is the biggest because it contains the auto-merge + side-pairing logic. It's the file D9 depends on most heavily.

## What "simple v1" looks like for D8

- **Euclidean warp**, no lanes. Any star within range is reachable in one move.
- **Single-fleet-per-owner at each star.** Auto-merge is the rule; split is the exception (player-initiated).
- **No mid-transit rerouting.** Order is set at issue time; ETA doesn't change once issued.
- **No in-transit interception.** Combat only happens at stars.
- **Stance has minor effect.** Mostly captured in D9. The field exists so v2 can add behavior without changing the data model.
- **Shuffle-and-pair for 3+ sides.** Per the D9 decision; clean v1 semantics. Unpaired side sits out the turn.

## Resolved decisions for D8

- **D8.2 reframed as distance & range** (no warp lanes in v1; lanes deferred to v2).
- **Same-owner auto-merge happens in D8.4**, not via a separate manual command. Players can't merge manually; it just happens.
- **Split is player-initiated only.** No auto-split.
- **Fleet warpRange is the min** of ship `Hull.baseWarpRange` values in the fleet.
- **D8.4 emits encounter events** in a form D9 reads directly — no shared state between D8 and D9.
- **Shuffle-and-pair for 3+ sides** (per D9 decision). Unpaired side sits out.
- **Random shuffling takes `rng: () => number` as a parameter** (per Architecture Principle 1 — no `Math.random()`).
- **ETA is locked at order issue.** Range upgrades during transit do not change arrival turn. v2 may add range-upgrade ETA recompute.
- **Stance is reserved data in v1.** D9 does not read stance in v1; D9.2 consumes it in v2.
- **D8.7 retreat helper** — shared by D9.5.4 (space-combat retreat) and D10.5 (invasion retreat). Single source of truth for retreat-destination algorithm.

## Open questions for D8

- **Fuel concept**: MoO didn't have fuel. Some 4X games do. **v1: no fuel.** Add as v2 chunk if requested.
- **Do fleets have a minimum size for combat?** E.g., can a fleet of 1 Fighter fight? v1: yes, no minimum. **Default: no minimum.**

## Cross-link with D9

By the time D9 reads `EncounterEvent` from D8.4:
- Each side in the event is **a single fleet id** (post-merge).
- The star has **exactly two sides** (or fewer; D9 only sees events for pairs).
- The owning player of each side is **fetched via `state.players[fleet.ownerId]`** by D9 (D8 doesn't pass player ids; the fleet carries its owner).

D9's `resolveSpaceCombat` reads `(state, EncounterEvent) -> newState + combatEvents`. The `EncounterEvent` is the input; the `state` provides the fleets, designs, race traits, etc.

This contract is **stable** — D9 doesn't need to know how the encounter was produced, only that "two sides at one star, here's the data."

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), all D9 dependencies are now decomposed (D1, D2, D3, D7, D8 — and D8 finishes the chain). The next commit per the roadmap is **D4 Turn Cycle** (small central orchestrator), then D5, D6, D10, D11, D12, D13, D14.

D8's Quint spec can be written **now** without waiting for D4/D5/etc. — D8's only dependencies are D1, D2, and D7, all done.
