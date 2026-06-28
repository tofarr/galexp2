# Section D12 — Espionage

D12 owns the spy system: assigning missions, resolving them each turn, producing intel and sabotage, and counter-espionage defense. It's small and focused — fewer moving parts than D9 or D10 — but it touches several other sections because espionage can affect techs, ships, buildings, and diplomacy.

## Section contract

> **Espionage**: given active spy missions, resolve them. Produces intel events (tech observed, fleet spotted, treasury observed), sabotage effects (building damaged, ship damaged, leader killed), and counter-espionage outcomes (spy captured, diplomatic consequences).

**Owns**:
- Spy mission data model (mission types, target, status).
- Mission assignment commands.
- Per-turn mission resolution (success check, detection check).
- Intel event production.
- Sabotage effect application.
- Counter-espionage skill computation and detection outcomes.

**Does not own**:
- Tech acquisition (D6) — D12 can OBSERVE a tech the target has; the player doesn't actually learn it (no effect applies to the player's `techs`). StealTech is just intel; actual tech transfer happens through D6.5's `receiveTech`.
- Ship damage mechanics (D9) — D12 just deals damage to a ship; D9 doesn't run.
- Building damage mechanics (D10.4) — D12 damages a building; D5.7's repair handles recovery.
- Leader mechanics — v1 has no separate leader system; "AssassinateLeader" just emits an event with no further effect (D3 doesn't yet have leaders).
- AI espionage decisions (D13) — D13 decides what missions to assign; D12 executes.

## Top-level chunks

Five top-level chunks from `DECOMPOSITION.md`:

- **D12.1 Spy mission assignment**.
- **D12.2 Mission resolution** — success check, detection check.
- **D12.3 Intel types** — tech stolen, fleet spotted, treasury observed.
- **D12.4 Sabotage effects** — building destroyed, ship damaged, etc.
- **D12.5 Counter-espionage** — defending against enemy spies.

## Recursive decomposition

### D12.1 → Spy mission assignment

A spy is a `Player` resource. Each player has zero or more spies; each spy has exactly one current mission (or is `Idle`). *Spy and Mission records live in D1.2 (`Spy`, `Mission`) — D12.1 only defines the mission-kind ADT and the assignment logic.*

```
// Mission.kind ADT — payload is target-specific (defined in D12.1, *not* in D1.2):
type MissionKind =
  | StealTech(target: PlayerId)
  | ObserveFleet(target: PlayerId, atStar: StarId)
  | ObserveTreasury(target: PlayerId)
  | SabotageBuilding(target: PlayerId, planet: PlanetId, building: BuildingKind)
  | SabotageShip(target: PlayerId, star: StarId, shipDesignId: ShipDesignId, count: int)
  | AssassinateLeader(target: PlayerId)
  | CounterEspionage(target: PlayerId, locationPlanet: PlanetId)
```

Records (declared in D1.2):

```
Spy    = { id, ownerId, skill, locationPlanetId, status, detectionRisk }
Mission = { id, spyId, kind, targetPlayerId, assignedTurn, progressTurns, difficulty }
```

**Assignment command**: `AssignMission(playerId, spyId, kind)` validates that the spy is `Idle`, then sets its status to `OnMission(missionId)` (referencing a new `Mission` record by id). Difficulty is computed from the target's tech level and counter-espionage skill:

```
difficulty = baseDifficulty(missionKind)
             + target.counterEspionageSkill
             - spy.skill * 2
```

**Spy production**: each `IntelligenceCenter` building on any planet produces 1 spy per `SPY_PRODUCTION_INTERVAL = 10` turns (D1.1 constant). Spies start at `skill = 3`. The production tick emits `SpyProducedEvent { player, spy, locationPlanet }`.

### D12.2 → Mission resolution

Each turn, resolve all active missions:

```
resolveMission(mission, ctx) -> MissionOutcome =
    advance progressTurns by 1
    if progressTurns < duration(mission.type): return InProgress
    roll success (vs difficulty)
    roll detection (vs counter-espionage)
    return MissionOutcome { success: bool, detected: bool, ... }
```

**Success roll**:
```
successChance = max(5%, min(95%, 50% + (spy.skill - difficulty) * 5%))
```
(spy skill vs difficulty, ±5% per skill point difference, clamped.)

**Detection roll** (independent of success):
```
detectionChance = max(5%, min(95%, counterEspionageSkill(target) - spy.skill * 3%))
```

**Mission duration**: defined by the `duration(missionKind)` table in D1.1 (constants `DURATION_STEAL_TECH = 3`, all other mission kinds = 1 turn). StealTech is longer because it's riskier (you have to root through the target's archives).

Outcomes:
- **Success + undetected**: produce intel or sabotage; spy returns to `Idle`, gains `+1 skill` (up to 10).
- **Success + detected**: produce intel/sabotage, BUT spy is captured (target gets diplomatic penalty, spy is lost).
- **Failure + undetected**: mission produces nothing; spy returns to `Idle`.
- **Failure + detected**: spy is captured; mission produces nothing.

`Captured` spies are removed from the player's spy pool. `Killed` is reserved for very high-detection missions (v1: same as Captured).

Emits: `MissionResolvedEvent { spy, mission, outcome }`.

**Intel accuracy flag** (D12.3): for *intel-gathering* missions (StealTech, ObserveFleet, ObserveTreasury), the resulting `IntelEvent` carries an `accurate: bool` field. The flag is set by D12.2 during resolution:
- `accurate = true` if the mission succeeded AND the spy was not detected.
- `accurate = false` if the mission succeeded AND the spy was detected (counter-espionage fed the spy bad info).
- `accurate = false` for failed missions (no intel produced).

For sabotage missions (SabotageBuilding, SabotageShip, AssassinateLeader), the `accurate` flag is not applicable — the event is the effect itself, not intel.

### D12.3 → Intel types

`IntelEvent` is an ADT describing what was learned:

```
type IntelEvent =
  | TechObserved(player: PlayerId, tech: TechId, accurate: bool)
  | FleetObserved(player: PlayerId, star: StarId, fleetSize: int, shipKinds: Map<HullId, int>)
  | TreasuryObserved(player: PlayerId, treasury: int, accurate: bool)
```

`accurate = true` means the spy got reliable intel. If detected during intel missions, `accurate = false` (counter-espionage fed them bad info).

Intel accumulates in `Player.intelLedger: Map<PlayerId, PlayerIntel>` — a record of what each player knows about each other. Used by:
- D13 (AI) — to make informed decisions.
- P8 (Espionage screen) — display what you know.
- D11 (Diplomacy) — to inform trade offers ("I know they have Tech X, so I'll offer Tech Y in trade").

**StealTech does NOT actually transfer the tech.** It's intel only — you know the target has it, but you don't get it for free. To actually get the tech, you'd use D6.5's `receiveTech` via diplomacy (e.g., trade for it).

This is a deliberate v1 design: espionage gives information, not direct power. Keeps the game from being decided by who has the most spies.

### D12.4 → Sabotage effects

For successful sabotage missions:

**SabotageBuilding**:
```
building.damaged = true
emit SabotageEvent { planet, building }
```
Same as D10.4 pillage effect. Auto-repair next turn via D5.7.

**SabotageShip**:
```
ship.hp -= 50% of ship.maxHp
emit ShipSabotagedEvent { fleet, ship, newHp }
```
Permanent until repaired (which can only happen at a planet with a Shipyard, D5.7).

**AssassinateLeader**:
```
emit LeaderAssassinatedEvent { player }
```
v1: emits the event but doesn't actually do anything else (no leader system yet). v2 will define leaders and have this affect governance/AI personality.

### D12.5 → Counter-espionage

Per-player skill, computed each turn:

```
counterEspionageSkill(player) =
    base 1
  + Σ(planet.buildings.baseEffect).filter(isIntelligenceCenter).value   // each Intelligence Center building (D1.2)
  + race.totalModifiers.counterEspionage                                  // race trait modifier (D3.2 — CounterEspionage(int))
  + techLevel(player, Computer) * 1                                       // better computers = better detection
  + min(5, sumOfEnemySpyAttempts)                                        // recent attempts sharpen awareness
```

The detection chance (D12.2) applies the *defending* player's `SpyDetection` trait modifier as a multiplier on top of the skill-derived base. Per D3.2's `SpyDetection` semantic, this multiplier is applied to the **defending** player's detection: a `Stealthy` defending player (1.0/0.85 = 1.18× defensive boost makes them *worse* per the inverted MoO semantic; a non-Stealthy defending player at 1.0 is neutral).

```
detectionChance = clamp(0.05, 0.95,
    (counterEspionageSkill(target) - spy.skill * 3%)
    * totalModifiers(targetPlayer.race).spyDetection
)
```

The skill is used in:
- Detection chance (D12.2's detection roll, as above).
- Difficulty of incoming missions (D12.1's difficulty calculation).

When a spy is detected and captured:
- `player.relations[attacker].score -= 5` (mild diplomatic penalty).
- Emit `SpyCapturedEvent { target, attacker, spy }`.

v1: detection is the only counter-espionage mechanic. v2: "counter-sabotage" where detected sabotage triggers a retaliation mission.

## Dependency graph (within D12)

```
D12.1 (assignment) ─────────────────────────────────┐
  ↓                                                  │
D12.2 (resolution) ← reads D12.1, D12.5              │
  ↓                                                  │
D12.3 (intel) ← called from D12.2 on success         │
D12.4 (sabotage) ← called from D12.2 on success      ┘

D12.5 (counter-espionage) ── modifies D12.1 and D12.2
```

Linear with D12.5 as a side-channel that affects both assignment difficulty and resolution detection.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Spy`, `SpyMission`, `MissionType`, `IntelEvent`, `Player`, `Planet`, `BuildingKind`, `Fleet`, `BuildingEffect.IntelligenceCenter` | D1.2 |
| D3 Races & Traits | `totalModifiers(race).counterEspionage` (from D3.2's `CounterEspionage(int)` variant) | D3.2 |
| D6 Research | (read-only) — `TechId` for StealTech intel | D6.1 |
| D7 Ship Design | `HullId` for ObserveFleet intel; `Hull.baseHp` for sabotage | D7.1 |
| D11 Diplomacy | (reads) — `relation.score` modified on detection | D11.1 |

D12 has small, well-defined imports.

| Section | What it imports from D12 |
|---|---|
| D4 Turn Cycle | Calls D12 once per turn (espionage phase) |
| D5 Economy | Reads `Player.intelLedger` indirectly via AI; D12 doesn't affect treasury |
| D11 Diplomacy | Reads `IntelEvent`s to inform offers (via D13) |
| D13 AI | Assigns spy missions, reads intel ledger |
| A1 Store | Calls D12 |
| P8 Espionage | Renders missions, intel ledger |

## Quint-spec-sized leaves (the actual implementation units)

Two Quint files (D12 is small):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/espionage/spyMission.qnt` | D12.1 + D12.2 + D12.3 + D12.4 (mission lifecycle: assign → resolve → intel/sabotage) | `src/domain/espionage/spyMission.ts` | ~280 |
| `specs/espionage/counterEspionage.qnt` | D12.5 (counter-espionage skill + detection effects) | `src/domain/espionage/counterEspionage.ts` | ~120 |

Total ~400 lines of Quint. D12.1-D12.4 are tightly coupled (mission data flows through resolution into intel/sabotage), so they share a file. D12.5 is the defending-side logic — separate concern.

## What "simple v1" looks like for D12

- **Most missions resolve in 1 turn.** StealTech takes 3 turns (more risky).
- **StealTech produces intel only** — not actual tech transfer. v2 could add actual steal.
- **IntelligenceCenter building produces spies** (1 per 10 turns per building).
- **Spy skill caps at 10** (max effectiveness).
- **Detected spy is captured** (removed from player). No "spy escapes" mechanic in v1.
- **Diplomatic penalty for detection** is mild (-5 relation).
- **AssassinateLeader is a stub event** in v1 (no leader system).
- **Sabotage damages but doesn't destroy** buildings/ships.

## Resolved decisions for D12

- **StealTech produces intel, not tech transfer** in v1. Espionage gives information; diplomacy gives tech.
- **Most missions resolve in 1 turn** (StealTech takes 3 turns).
- **Spy skill caps at 10**.
- **Detected spy is captured** (no escape).
- **AssassinateLeader is a stub** in v1 (emits event with no further effect).
- **Detection has mild diplomatic penalty** (-5 relation).
- **SabotageBuilding reuses the pillage damaged-flag** pattern from D10.4.
- **`IntelEvent.accurate` flag** is set by D12.2: `true` for success+undetected, `false` for success+detected or failure.
- **`CounterEspionage` race trait** is part of D3.2's `Modifier` ADT (`CounterEspionage(int)`).
- **`IntelligenceCenter` building effect** is part of D1.2's `BuildingEffect` ADT (`IntelligenceCenter(int)`).

## Open questions for D12

- **Spy skill growth rate**: +1 per successful mission, cap at 10. **Default v1.**
- **SPY_PRODUCTION_INTERVAL**: 10 turns per Intelligence Center. **Default v1.** Tune during playtesting.
- **Initial spy count**: each player starts with 1 spy. **Default v1.**
- **StealTech duration**: 3 turns. **Default v1.** Trade-off between risk (longer = more detection) and reward (intel).

No open questions block starting the D12 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D13 AI Decision Logic** (depends on D1, D3, D5, D6, D8, D9, D10, D11, D12 — all done or about to be).

D12's Quint spec can be written **now** — D12's dependencies (D1, D3, D6, D7, D11) are all done.