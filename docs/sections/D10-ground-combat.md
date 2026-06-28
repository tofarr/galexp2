# Section D10 — Ground Combat Resolution

D10 resolves planetary invasions — the moment after a fleet wins space combat at a star with a defended colony. It's smaller than D9 (no targeting AI, no retreat, simpler mechanics) but follows the same shape: pure function, deterministic, takes `(attacker, defender, planet) -> (state', events)`.

## Section contract

> **Ground Combat Resolution**: given an invading fleet and a defended colony, resolve the ground battle. Produces new population, buildings, and ground combat events. On conquest, ownership flips to the invader.

**Owns**:
- Troop strength calculation (invasion forces, garrison, native allies/enemies).
- Ground combat round resolution (attacker/defender rolls, casualties).
- Defender surrender conditions.
- Post-combat pillage and collateral damage (population loss, building damage).
- Final outcome (conquest, repelled, draw).

**Does not own**:
- Space combat (D9) — D10 reads the surviving invading fleet, but D9 produces it.
- Fleet movement (D8) — D10 doesn't move fleets; it only resolves the static "fleet at planet" scenario.
- Forced treaties from conquest (deferred to D11 Diplomacy or D14 Victory — see "Resolved decisions").
- Bombardment (deferred to v2 — see "Open questions").
- Post-conquest revolt mechanics (D5.1 emits `RevoltRiskEvent` when morale is low; actual revolt is v2).
- AI invasion decisions (D13) — D13 decides whether to invade; D10 only resolves the resulting battle.

## Top-level chunks

Five top-level chunks from `DECOMPOSITION.md`:

- **D10.1 Troop strength calculation** — invasions, garrison, native populations.
- **D10.2 Combat rounds** — attacker/defender rolls, casualties.
- **D10.3 Morale & surrender**.
- **D10.4 Pillage & collateral damage**.
- **D10.5 Outcome** — conquest, repelled, draw.

D10 is invoked **after** D9 produces a winner at a star with a defended colony. The orchestrator (D4) calls D9 first, then D10 if a ground invasion happens. The trigger: the winning fleet contains at least one troop transport AND the player issues an `Invade` command (or has the default "auto-invade after winning space combat" preference set).

**Multi-invasion per turn**: D4's `groundCombat` phase (phase 7b) loops over each planet with at least one invading fleet and runs D10 once per invasion order, sequentially in fleet-arrival order. The first invasion resolves against the planet's current state; the second sees the new (post-pillage or post-conquest) state, etc. This is the same "sequential resolution" pattern D9 uses for 3+ side encounters.

## Recursive decomposition

### D10.1 → Troop strength calculation

Pure function: `computeTroopStrengths(attackerFleet, defenderPlanet, attackerRace, defenderRace, ctx) -> TroopStrengths`.

`TroopStrengths` is a record with four numbers:

```
type TroopStrengths = {
  attackerTroops: int,          // from invasion transports in attackerFleet
  attackerNativeAllies: int,    // 0 unless planet was Native and now cooperates (v2; v1: 0)
  defenderGarrison: int,        // from defenderPlanet.garrison
  defenderNativeResistance: int,// 0 unless planet has Native/Artifact special
}
```

**Attacker troops** count comes from the invasion fleet:

```
attackerTroops = Σ ship.count × shipDesign(state, ship).hull.troopCapacity
                 // Transport hull from D7 has troopCapacity = 100 (default v1)
                 // Cruiser hull has troopCapacity = 0 (not an invasion ship)
                 × (1 + race.totalModifiers.groundAttack / 100)
```

`shipDesign(state, ship)` is the lookup into `state.designs[ship.designId]`; `troopCapacity` lives on the Hull (D7.1). The `groundAttack` modifier comes from D3.2's `Modifier` ADT (e.g., `Militaristic` grants `GroundAttack(1)`).

**Defender garrison** comes from the planet:

```
defenderGarrison = planet.garrison
                   × (1 + defenderRace.totalModifiers.groundDefense / 100)
                   + planet.population × 0.01   // civilian resistance is small
```

The `groundDefense` modifier likewise comes from D3.2 (e.g., `Subterranean` grants `GroundDefense(1)`).

**Native resistance** (if planet is `Native` or `Artifact` and not fully colonized):

```
defenderNativeResistance = planet.population × 0.05
                           // natives fight the invader even if they own the planet
```

In v1, natives always fight the invader regardless of who owns the planet. (v2: politics — colonized natives might cooperate with the new invader if conditions are right.)

**Attacker native allies**: v1 always 0. Natives don't cooperate with invaders in v1.

Emits: optional `GroundCombatSetupEvent { attackerTroops, defenderGarrison, defenderNativeResistance }` for UI / replay.

### D10.2 → Combat rounds

Pure function: `resolveGroundCombat(strengths, ctx) -> CombatResult`.

```
CombatResult = {
  rounds: int,
  attackerSurvivors: int,
  defenderSurvivors: int,
  surrendered: bool,
  log: List<GroundRoundEvent>,
}
```

Each round:
1. Compute casualties on each side:
   ```
   attackerCasualties = max(1, defenderStrength / attackerStrength)
   defenderCasualties = max(1, attackerStrength / defenderStrength)
   ```
   where each `*Strength` is `troops × (1 + morale/100)` (morale feeds in here, computed once before combat).

2. Apply casualties; emit `GroundRoundEvent { round, attackerLoss, defenderLoss }`.

3. Repeat until one side is at 0 (or surrenders per D10.3).

Maximum rounds: `MAX_GROUND_ROUNDS = 30` (from D1.1 constants; if neither side wins in 30 rounds, it's a draw and the invader must withdraw — no conquest).

`MAX_GROUND_ROUNDS` exists to prevent infinite loops and to give the AI a clear "this isn't working, retreat" signal.

### D10.3 → Morale & surrender

Per-round check (called from D10.2's loop):

```
defenderSurrenders = defenderSurvivors <= defenderInitial × 0.20
                     OR defenderMorale < 20
```

If `defenderSurrenders`, combat ends; outcome is conquest (D10.5). Attacker survivors get a small bonus (no final casualties) and the colony flips ownership.

Defender morale is computed once before combat:

```
defenderMorale = baseMorale(50)
                 + buildings.defenseBonus(planet)     // e.g., Defense +10 (D1.2 BuildingKind)
                 + race.totalModifiers.morale          // e.g., Honorable +1
                 - (defenderStrengthRatio < 1 ? 20 : 0) // hopeless defender penalty
```

If the defender has `Subterranean` trait, they don't surrender until their garrison is fully eliminated (defensive advantage). *The check is data-driven: D10.3 calls `hasTrait(state, defender, Subterranean)` rather than reading a special-case flag. The `GroundDefense(1)` modifier from D3.2's `traitModifiers(Subterranean)` is also applied to the defender's defense roll (separate effect).*

**Attacker surrender**: not modeled in v1. If the invader is losing, they can choose to retreat via a command, which D10.5 handles as a draw (no conquest, no fleet loss, invader can move away).

### D10.4 → Pillage & collateral damage

Applied only on conquest (called from D10.5):

```
populationLoss = planet.population × (0.10 + sumPillagerModifier(attackerRace))
                // 10% baseline + race modifier via D3.2's Modifier.Pillage(float)
                // (e.g., Bulrathi trait might grant Pillage(0.05))
buildingDamage = sample(planet.buildings, prob=0.15 per building)
                // 15% chance each building is damaged (needs repair)
```

`sumPillagerModifier(race)` reads `totalModifiers(race)` and sums the
`Pillage(float)` variants. See D3.2's `Modifier` ADT for the full
enumeration. The base 10% is the v1 default; trait-derived modifiers
add to it.

`buildingDamage` doesn't destroy buildings; it sets `building.damaged = true`. Repairs happen automatically next turn (D5.7's queue) or manually (P4).

If `attackerRace` has the `Ruthless` AI personality hint, `populationLoss` is doubled and `buildingDamage` probability is 30%.

Emits: `PillageEvent { planet, populationLost, buildingsDamaged }`.

### D10.5 → Outcome

Pure function: `applyGroundOutcome(planet, combatResult, attacker, ctx) -> (planet', events)`.

```
outcome =
  if combatResult.defenderSurvivors == 0 or combatResult.surrendered:
    Outcome.Conquest
  elif combatResult.attackerSurvivors == 0:
    Outcome.Repelled
  elif combatResult.rounds >= MAX_GROUND_ROUNDS:
    Outcome.Draw
  else:
    Outcome.Repelled   // default if anything else weird
```

**Conquest**:
1. Apply D10.4 (pillage).
2. `planet.owner = attacker.playerId`.
3. `planet.population = max(1, planet.population - populationLoss)`.
4. Mark damaged buildings.
5. Reset `planet.garrison` to invader's remaining troops (or 0 if invader takes only partial control).
6. `planet.conqueredOnTurn = Some(state.turn)` (consumed by D5.6's morale `recentlyConquered` modifier for the next 5 turns).
7. Emit `PlanetConqueredEvent { planet, newOwner, oldOwner, casualties }`.

**Repelled**:
- Planet unchanged.
- Invading fleet's troop transports are lost (their troops are casualties).
- Other ships in the invading fleet retreat to a star within warp range (handled by D8.5/D8.4's retreat path — D10 just emits `InvasionRepelledEvent` and D8 picks it up).
- Emit `InvasionRepelledEvent { planet, attacker, casualties }`.

**Draw**:
- Both sides withdraw.
- Invading fleet retreats (same as Repelled).
- Emit `InvasionDrawEvent { planet }`.

**Forced treaties from conquest**: out of scope for D10. v1 doesn't auto-force treaties from conquering N% of a player's planets. This is a v2 mechanic that would live in D11 (Diplomacy) or D14 (Victory). For now, conquering enough just denies the enemy productive planets; their treaties are independent.

## Dependency graph (within D10)

```
D10.1 (troops) ───────────────┐
  ↓                           │
D10.3 (morale) ← reads troops │
  ↓                           │
D10.2 (combat rounds) ← reads troops + morale
  ↓
D10.5 (outcome) ── reads combat result
  ↓
D10.4 (pillage) ← called only on conquest
```

Linear with D10.4 (pillage) called from D10.5 (outcome) only on conquest.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Planet`, `Fleet`, `Ship`, `Player`, `Race`, `Modifier`, `Building` | D1.2 |
| D3 Races & Traits | `totalModifiers(race).groundAttack/Defense`, `aiPersonalityHints` for Ruthless | D3.2 |
| D7 Ship Design | `Hull.troopCapacity` (Transport hull has N; others have 0) | D7.1 |
| D8 Fleet & Movement | `retreatFleet` for Repelled/Draw outcomes | D8.5 |
| D9 Space Combat | (no direct call) — D9 produces the winning fleet; D10 reads it | D9 |

D10 doesn't import from D9 directly — D9 produces `CombatOutcome` events; D4 orchestrator reads them and decides whether to invoke D10.

| Section | What it imports from D10 |
|---|---|
| D4 Turn Cycle | Calls D10 after D9 if an invasion was ordered |
| D5 Economy | Reads `planet.owner` (changed by D10.5 conquest) for production calculations |
| D11 Diplomacy | (v2) — would read conquest events for forced treaties |
| D14 Victory | Reads conquest events for score/victory calculation |
| D13 AI | Reads ground combat results to inform future invasion decisions |
| A1 Store | Calls D10 |
| P11 Ground Combat Screen | Renders ground combat events |

## Quint-spec-sized leaves (the actual implementation units)

Three Quint files (D10 is small):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/ground/troops.qnt` | D10.1 | `src/domain/ground/troops.ts` | ~120 |
| `specs/ground/groundRounds.qnt` | D10.2 + D10.3 (combat rounds + surrender) | `src/domain/ground/groundRounds.ts` | ~180 |
| `specs/ground/outcome.qnt` | D10.4 + D10.5 (pillage + final outcome) | `src/domain/ground/outcome.ts` | ~150 |

Total ~450 lines of Quint. D10.4 and D10.5 are tightly coupled (pillage runs only on conquest, which outcome decides), so they're in the same file.

No top-level orchestrator — D4 calls each chunk in sequence (`troops → groundRounds → outcome`).

## What "simple v1" looks like for D10

- **Simple casualty formula**: `max(1, attack/defense)` per round. v2 could add terrain, fortifications, etc.
- **No terrain modifiers** in v1. Every planet defends identically except for garrison size and native specials.
- **No forced treaties from conquest** in v1. v2 chunk.
- **No bombardment** in v1 (only invasion). v2 chunk.
- **Subterranean trait gives "no surrender until fully eliminated"** (defensive bonus).
- **`MAX_GROUND_ROUNDS = 30`**. After 30 rounds, draw. Prevents stalemates.
- **Invading fleet auto-retreats on Repelled/Draw** via D8 pathfinding. No ground-combat-level retreat logic in D10.
- **Natives always fight invaders** regardless of ownership. v2: politics.
- **Pillage damages but doesn't destroy** buildings. Auto-repair next turn.

## Resolved decisions for D10

- **No bombardment in v1** (only invasion). v2 chunk.
- **No forced treaties from conquest** in v1. v2 chunk.
- **Subterranean doesn't surrender** until fully eliminated (defensive trait via `GroundDefense(1)` from D3.2's Modifier ADT).
- **`MAX_GROUND_ROUNDS = 30`** prevents stalemates.
- **Invading fleet retreats via D8** on Repelled/Draw (D10 emits the retreat event; D8 handles the actual movement).
- **Natives always fight invaders** regardless of planet ownership.
- **Pillage damages but doesn't destroy** buildings; auto-repair next turn.
- **Ground combat is phase 7b** in D4's step (between `combatResolution` and `espionage`); runs once per invasion order, sequentially in fleet-arrival order.
- **`groundAttack` and `groundDefense` modifiers** come from D3.2's `Modifier` ADT; D10 reads them via `race.totalModifiers.X`.

## Open questions for D10

- **Pillage percentage baseline**: 10% population loss + race modifier. **Default v1: 10%.** Tune during playtesting.
- **Native resistance formula**: 5% of planet population fights as `defenderNativeResistance`. **Default v1: 5%.**
- **Invader retreat destination**: when invasion is repelled, where does the fleet go? Default v1: nearest star within warp range, picked randomly if multiple options. D8 picks the destination; D10 just emits the retreat intent.
- **Defensive buildings**: should "PlanetaryShield" building affect garrison strength? **Default v1: yes, +10 defenderMorale.** Adds a small strategic reason to build it.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D11 Diplomacy** (depends on D1, D3, D6 — all done).

D10's Quint spec can be written **now** — D10's only dependencies are D1 (done), D3 (done), and D7 (done).
