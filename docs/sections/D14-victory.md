# Section D14 — Victory Conditions

D14 is the smallest and final D-section. It runs once per turn (after diplomacy, before end-of-turn cleanup per D4) to check whether anyone has won. If so, it produces the final stats and ends the game.

## Section contract

> **Victory Conditions**: after each turn, check whether any player has won. If so, gather end-game stats, emit `GameEndedEvent`, and stop the game loop.

**Owns**:
- Conquest victory check (owns X% of galaxy / last empire standing).
- Technological victory check (researched X key techs).
- Diplomatic victory check (read from D11.5's Galactic Emperor result).
- Score / time victory check (at MAX_TURN, highest score wins).
- End-game state computation (rankings, final stats).

**Does not own**:
- The game loop itself (A4 calls D14 once per turn).
- The post-game screen (P8 reads D14's output; P8 is presentation only).
- Scoring internals — D14 reads economy/tech/fleet state to compute scores; D14 doesn't define the formula.

## Top-level chunks

Five top-level chunks from `DECOMPOSITION.md`:

- **D14.1 Conquest victory** — owns X% of galaxy, last empire standing.
- **D14.2 Technological victory** — researched X key techs.
- **D14.3 Diplomatic victory** — elected by council.
- **D14.4 Score / time victory** — at turn N, highest score wins.
- **D14.5 End-game state** — rankings, final stats, post-game screen.

## Recursive decomposition

### D14.1 → Conquest victory

Two conditions, either of which triggers conquest victory:

**Condition A — Domination**: a player owns at least `CONQUEST_VICTORY_THRESHOLD = 80%` of habitable planets AND no other player has more than 10%.

```
checkConquest(player, state) =
    let owned = count(planet.owner == player for planet in state.planets if planet.habitable)
    let totalHabitable = count(planet for planet in state.planets if planet.habitable)
    let playerShare = owned / totalHabitable
    let maxOtherShare = max(
        count(planet.owner == otherPlayer for planet in state.planets if planet.habitable) / totalHabitable
        for otherPlayer in state.players if otherPlayer != player
    )
    return playerShare >= 0.80 and maxOtherShare <= 0.10
```

**Condition B — Last empire standing**: only one player owns any planets (i.e., everyone else has been eliminated entirely).

```
checkLastEmpireStanding(player, state) =
    let playersWithPlanets = { other.id for other in state.players if count(otherPlanets > 0) > 0 }
    return playersWithPlanets == {player}
```

Either condition triggers `ConquestVictory { player }`.

### D14.2 → Technological victory

A player wins by researching all `TECH_VICTORY_KEY_TECHS` — a fixed set of high-tier techs.

```
KEY_TECHS = {
    WeaponsVI, ConstructionVI, ComputerVI, ShieldsVI, ForceFieldsVI, PropulsionVI
    // 6 techs (one per tree, top level)
}

checkTechVictory(player, state) =
    return all(tech in KEY_TECHS, tech in player.techs)
```

(The exact list lives in `techCatalog.qnt` as a constant; could grow in v2.)

### D14.3 → Diplomatic victory

Reads from D11.5's `GalacticEmperorVictoryEvent`. D11 already runs the election; D14 just checks whether the event was emitted this turn.

```
checkDiplomaticVictory(events) =
    for event in events:
        if event.kind == GalacticEmperorVictory:
            return Some(event.winner)
    return None
```

(Note: D11.5 emits the event; D14.3 picks it up. No duplicate logic.)

### D14.4 → Score / time victory

At `state.turn >= MAX_TURN`, the player with the highest score wins.

Score formula:

```
score(player) =
    count(player.planets) × 1                         // 1 point per planet
  + count(player.techs) × 5                         // 5 points per tech
  + totalPopulation(player) × 0.001                // 0.001 per population unit
  + totalFleetStrength(player) × 0.1              // 0.1 per fleet combat stat sum
  + player.treasury × 0.001                        // 0.001 per bc
```

`MAX_TURN = 200` (D1.1 constant).

At MAX_TURN, the player with the highest score wins. Ties broken by who reached the score first (turn of last score update).

If MAX_TURN is reached and multiple players have the same score, it's a tie (no winner; show as "draw" in post-game).

### D14.5 → End-game state

Once any victory is detected:

```
buildEndGameState(state, winner, victoryKind) -> EndGameState =
    EndGameState {
        winner: winner,
        victoryKind: victoryKind,        // Conquest | Tech | Diplomatic | Score
        rankings: sortByScore(state.players),
        stats: Map<PlayerId, PlayerStats>,
        finalTurn: state.turn,
    }

PlayerStats {
    playerId: PlayerId,
    score: int,
    planetsOwned: int,
    techsResearched: int,
    population: int,
    fleetStrength: int,
    battlesWon: int,
    battlesLost: int,
    spiesDeployed: int,
    treatiesSigned: int,
}
```

Emits `GameEndedEvent { winner, victoryKind, stats, finalTurn }`. The orchestrator (D4) checks for this event at the end of `step` and stops the game loop.

## Dependency graph (within D14)

```
D14.1 (conquest) ────────────────┐
D14.2 (tech) ────────────────────┤
D14.3 (diplomatic, reads events) ├──> checkAllVictories(state, events)
D14.4 (score) ───────────────────┘            │
                                                ↓
                                          D14.5 (end-game state)
```

D14.5 is called only when one of D14.1-D14.4 detects a winner.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Player`, `GameState`, `Planet`, `Event`, `EndGameState`, `PlayerStats` | D1.2, D1.3 |
| D6 Research | (read-only) — `Player.techs` for tech victory | D6 |
| D11 Diplomacy | (read-only) — D11.5's `GalacticEmperorVictoryEvent` | D11.5 |

D14 has minimal imports — it's a leaf consumer of game state.

| Section | What it imports from D14 |
|---|---|
| D4 Turn Cycle | Calls D14 once per turn (after diplomacy, before end-of-turn cleanup) |
| A4 Turn Manager | Stops game loop on `GameEndedEvent` |
| P8 End-Game Screen | Reads `EndGameState` |
| P15 Stats / Hall of Fame | Persists historical `PlayerStats` across games |

## Quint-spec-sized leaves (the actual implementation units)

Two Quint files (D14 is small):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/victory/victoryConditions.qnt` | D14.1 + D14.2 + D14.3 + D14.4 (the four victory checks + `checkAllVictories`) | `src/domain/victory/victoryConditions.ts` | ~150 |
| `specs/victory/endGame.qnt` | D14.5 (final stats, rankings, event emission) | `src/domain/victory/endGame.ts` | ~100 |

Total ~250 lines of Quint. The four checks are small (each ~30-50 lines); the orchestrator function `checkAllVictories` is a switch over the four kinds.

## What "simple v1" looks like for D14

- **Four victory kinds**: Conquest, Technological, Diplomatic, Score.
- **Conquest**: 80% planet domination OR last empire standing.
- **Tech**: 6 key techs (one per tree, top level).
- **Diplomatic**: read from D11.5's `GalacticEmperorVictoryEvent`.
- **Score**: at MAX_TURN (200), highest score wins.
- **Stats recorded**: planets, techs, population, fleet strength, treasury, battles, treaties.
- **No "alliance victory"** in v1 (would require multiple allied players to win together). v2.
- **No "transcendence" victory** (some games have a "build a special project" win). v2.
- **No "diplomatic points over time"** — only the council election triggers diplomatic victory.

## Resolved decisions for D14

- **Four victory kinds**: Conquest, Tech, Diplomatic, Score.
- **Conquest threshold: 80% planets + no rival above 10%**, OR last empire standing.
- **Tech victory: 6 key techs** (top level of each tree).
- **`MAX_TURN = 200`** for the score/time victory.
- **Score formula**: planets + techs + population + fleet + treasury.
- **No alliance or transcendence victory** in v1.
- **Ties in score**: declared a draw (no winner).

## Open questions for D14

- **Score formula balance**: weights for planets, techs, etc. are placeholders. **Default v1: 1 × planets + 5 × techs + 0.001 × pop + 0.1 × fleet + 0.001 × treasury.** Tune during playtesting.
- **Tech victory key techs**: top of each tree (6 techs), or specific named techs (e.g., "Trans-galactic Constructor")? **Default v1: top of each tree (6 techs).**
- **Draw condition**: if MAX_TURN reached and tied, show "draw" or pick by tiebreaker? **Default v1: draw.** v2: tiebreaker (e.g., highest fleet strength).
- **End-game stats tracking**: do we record `battlesWon` / `battlesLost`? If so, D9 needs to emit counters. **Default v1: yes; D9 emits `BattleResolvedEvent` with winner/loser sides.**

No open questions block starting the D14 Quint spec.

## Final D-section summary

This completes the **domain layer (D-layer)** of Phase 1.

| Section | Status | Topic |
|---|---|---|
| D1 Core Types | ✅ | Primitives, entity types, GameState |
| D2 Galaxy Generation | ✅ | Procedural galaxy |
| D3 Races & Traits | ✅ | Race catalog, trait modifiers |
| D4 Turn Cycle | ✅ | Master `step` operator |
| D5 Economy | ✅ | Food, industry, research, income, morale, queues |
| D6 Research & Tech Tree | ✅ | 6 trees × 6 levels, 36 techs |
| D7 Ship Design | ✅ | Hulls, weapons, specials, validation, stats |
| D8 Fleet & Movement | ✅ | Movement, arrival, encounter detection |
| D9 Space Combat | ✅ | Pilot; full tactical combat |
| D10 Ground Combat | ✅ | Invasion, conquest |
| D11 Diplomacy | ✅ | Treaties, council, trade |
| D12 Espionage | ✅ | Spy missions, intel, sabotage |
| D13 AI Decision Logic | ✅ | 5 strategies, personality presets |
| D14 Victory Conditions | ✅ | Conquest, Tech, Diplomatic, Score |

After this commit, all 14 D-sections are decomposed. The remaining Phase 1 work is the **A-layer** (Application: state store, persistence, orchestration), **P-layer** (Presentation: screens), and **I-layer** (Infrastructure: tooling).

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), after D14 the **D-layer is complete**. The next Phase 1 sections are A1-A5 (Application layer: state store, persistence, command queue, turn manager, event sourcing). A1-A5 depend on D1 (done) and define the runtime that wraps the pure domain logic.

D14's Quint spec can be written **now** — D14's dependencies (D1, D6, D11) are all done.