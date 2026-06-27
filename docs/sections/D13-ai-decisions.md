# Section D13 — AI Decision Logic

D13 generates `Command[]` for each AI player at the start of each turn. It's the largest section by line count, but most of the volume is the **common scaffolding** — the strategies themselves are mostly personality presets that override a few weights in the shared heuristic.

## Section contract

> **AI Decision Logic**: given an AI's `state` and its personality, produce a `Command[]` for the next turn. Each AI strategy is a separate chunk.

**Owns**:
- Common AI input/output types (`AIInput`, `AIOutput`, `Personality`).
- Personality traits and weights.
- Per-strategy decision heuristics (Aggressive, Builder, Technologist, Diplomat, Balanced).
- The per-turn pipeline that runs each AI in order.

**Does not own**:
- Command execution — D13 generates commands; D4 applies them.
- Combat resolution (D9), economy (D5), etc. — AIs issue commands to those subsystems but don't run them.
- Pathfinding / target selection at the movement level — D13 picks destinations; D8 handles the actual movement.

## Top-level chunks

Seven top-level chunks from `DECOMPOSITION.md`:

- **D13.1 Common AI scaffolding** — input/output types, personality traits.
- **D13.2 Strategy: Aggressive Conqueror** — prioritizes military, picks fights with weakest neighbors.
- **D13.3 Strategy: Builder** — colonizes widely, develops economy, late-game military.
- **D13.4 Strategy: Technologist** — rushes research, trades for tech.
- **D13.5 Strategy: Diplomat** — pursues alliances and council victory.
- **D13.6 Strategy: Balanced** — weighted blend of above.
- **D13.7 Per-turn AI evaluation pipeline** — when each AI runs, in what order.

## A note on chunk size

For v1, the strategies are mostly **personality presets** — each defines a weight vector and a few custom heuristics, then delegates to the shared decision logic. So D13.2-D13.6 are small. D13.1 (the common scaffolding) is large. This is intentional: the heavy lifting happens once in shared code, and the strategies differentiate on weights.

v2 can grow the strategies independently (e.g., a real search-based AI, a "turtle" AI, etc.) without touching the shared code.

## Recursive decomposition

### D13.1 → Common AI scaffolding

The shared infrastructure. Everything else builds on this.

**Input type:**

```
type AIInput = {
  playerId: PlayerId,
  playerState: Player,
  gameState: GameState,
  personality: Personality,
  intelLedger: PlayerIntel,                  // from D12
  turnNumber: int,
}
```

**Output type:**

```
type AIOutput = {
  commands: List<Command>,                   // ready to be applied by D4
  reasoning: List<string>,                   // debug log (used by P14 AI Inspector)
}
```

**Personality:**

```
type Personality = {
  strategy: StrategyKind,                    // Aggressive | Builder | Technologist | Diplomat | Balanced
  weights: WeightVector,
  aiMemory: AIMemory,                        // persistent across turns
}

type WeightVector = {
  military: float,                           // 0..1
  economic: float,                           // 0..1
  research: float,                           // 0..1
  diplomatic: float,                         // 0..1
  espionage: float,                          // 0..1
}
```

**Constraint**: the five fields of `WeightVector` sum to 1.0 by construction. Constructors enforce this; tests assert it. All five pre-set strategies respect this (e.g., Aggressive = 0.8+0.1+0.05+0.05+0.0 = 1.0). The constraint exists so that score formulas using the vector as a multiplier produce comparable totals across strategies.

The strategies are essentially pre-set weight vectors + a few custom heuristics. See D13.2-D13.6. Strategy selection *is* the difficulty mechanism in v1: the human player chooses each AI's strategy at game start, giving them a wide difficulty range from "Balanced" (default) to "Aggressive Conqueror" (hardest). A separate per-AI difficulty slider is deferred to v2.

**Common decisions (shared by all strategies):**

1. **Build queue management**: which item to put at the head of each planet's queue.
   - Score each candidate item (building, ship) by `item.value × weights.X × context_multiplier`.
   - Pick the highest-scoring item per planet.

2. **Research selection**: which tech to research next.
   - Score each available tech by `tech.value × weights.research × race.techAffinity(tree)`.
   - Pick the highest-scoring tech.

3. **Fleet movement**: where to send idle fleets.
   - Score each destination by `destination.value × weights.X × personality_modifier`.
   - Pick the highest-scoring destination per idle fleet.

4. **Diplomatic stance**: which treaties to propose / declare war on.
   - Score each relation by `relation.value × weights.diplomatic`.
   - Pick actions (propose / cancel / declare war) per relation.

5. **Spy missions**: which missions to assign.
   - Score each mission type by `mission.value × weights.espionage`.
   - Pick missions to assign.

These are all "score candidates, pick best" decisions. The strategies differ only in the weights and a few custom heuristics.

**AIMemory:**

```
type AIMemory = {
  lastActionByRelation: Map<PlayerId, DiplomaticAction>,
  grudgeList: Set<PlayerId>,                 // players we want to attack eventually
  threatList: Map<PlayerId, int>,            // how threatened we feel by each player
  targetStars: Map<FleetId, StarId>,         // fleet → long-term destination
  targetPlayer: Option<PlayerId>,            // current "weakest neighbor" focus (Aggressive strategy)
  grudgesExpireAfterTurns: int,              // default 50
}
```

AIs remember past events. The grudge list drives target selection for aggressive AIs; the threat list drives defensive behavior. `targetPlayer` is used by the Aggressive Conqueror strategy (D13.2) to hold focus on the weakest neighbor; it is reset to `None` when the target player is destroyed.

### D13.2 → Strategy: Aggressive Conqueror

Weights: `{ military: 0.8, economic: 0.1, research: 0.05, diplomatic: 0.05, espionage: 0.0 }`.

Custom heuristics:
- **Target selection**: pick the weakest neighbor (lowest fleet strength + planet count) and focus attacks there. The chosen target is stored in `AIMemory.targetPlayer: Some(playerId)`. The AI doesn't switch targets unless the current target is destroyed (in which case `targetPlayer` is reset and a new weakest neighbor is chosen).
- **Build priority**: 80% of industry → military ships; 20% → economy.
- **Diplomacy**: ignore most offers; only NAP with stronger players as a temporary measure.
- **Fleet movement**: idle fleets automatically move toward nearest enemy star.

The "weakest neighbor" target is held in `AIMemory.targetPlayer`. The AI doesn't switch targets unless the current target is destroyed.

### D13.3 → Strategy: Builder

Weights: `{ military: 0.2, economic: 0.5, research: 0.2, diplomatic: 0.05, espionage: 0.05 }`.

Custom heuristics:
- **Colonize every habitable planet** within range (highest priority).
- **Build order**: Farm → Factory → Research Lab → Market (skip until colonies are stable).
- **Military**: only when at war or threatened; build defensive ships (Cruisers, Battleships).
- **Diplomacy**: pursue Trade Pacts with strong neighbors.

This AI's economic expansion is its defining trait. Late-game, it transitions to military once it has a stable economic base.

### D13.4 → Strategy: Technologist

Weights: `{ military: 0.1, economic: 0.2, research: 0.6, diplomatic: 0.1, espionage: 0.0 }`.

Custom heuristics:
- **Research speed**: pick the cheapest available tech with the most `ProductionBonus` or `UnlockHull` effects (techs that unlock ship components are highest priority).
- **Tech trade**: actively propose Research Agreements; trade duplicate techs (e.g., techs the AI has and another player also has prereqs for) for techs it lacks.
- **Military**: minimal — only enough to defend. Ships designed for computer/accuracy, not damage.
- **Espionage**: zero — doesn't bother with spies.

The Technologist tries to win via D14.2 (Technological Victory — research X key techs).

### D13.5 → Strategy: Diplomat

Weights: `{ military: 0.2, economic: 0.2, research: 0.1, diplomatic: 0.5, espionage: 0.0 }`.

Custom heuristics:
- **Council focus**: builds council votes by developing planets (council vote count = the derived `state.score` field on `GameState`, written by D14.4).
- **Alliances**: form Alliances with all possible players; defend allies when attacked.
- **Trade**: open Trade Routes with everyone who'll accept.
- **War avoidance**: only declares war if absolutely necessary (e.g., ally is being destroyed).

The Diplomat tries to win via D14.3 (Diplomatic Victory — elected by council).

### D13.6 → Strategy: Balanced

Weights: `{ military: 0.3, economic: 0.3, research: 0.2, diplomatic: 0.1, espionage: 0.1 }`.

This is the default strategy when no specific personality is set. It uses the shared decision logic with default weights and no custom heuristics.

It's the "AI does a bit of everything" strategy. Effective at the start of the game; less effective against specialized strategies.

### D13.7 → Per-turn AI evaluation pipeline

For each AI player, in **threat-descending order** (most-threatened first):

```
runAIPipeline(state, ctx) =
    for player in sortByThreat(aiPlayers):
        commands = runStrategy(player.personality, state, ctx)
        emit AIPipelineEvent { player, commandCount: |commands| }
    return commands collected per player
```

"Threat" is computed as: `Σ (relationScore * -1) for each at-war relation + ownPlanetCount / totalPlanets`. (This is *not* D14.4's score formula; threat is a per-player pressure measure, while score is a long-game total.)

The order doesn't affect the result (each AI acts on its own state), but it affects debug logs and any per-player UI animations.

For v1, AIs run **sequentially** in a single batch with all human commands. v2 could run them in parallel (multiple Quint/TS evaluations concurrently).

## Dependency graph (within D13)

```
D13.1 (scaffolding) ────────────────────────────────┐
  ↓                                                  │
D13.2-D13.6 (strategies) ← each overrides weights    │
  ↓                                                  │
D13.7 (pipeline) ← runs each strategy in sequence   ┘
```

D13.7 calls the others; the strategies depend on D13.1's scaffolding.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Player`, `GameState`, `Command`, `Personality` | D1.2 |
| D3 Races & Traits | `techAffinity(race, tree)`, `aiPersonality(race)` for base personality | D3.2, D3.4 |
| D5 Economy | Reads `player.economy` for build decisions | D5 (read-only) |
| D6 Research | Reads `player.techs` for research selection | D6 (read-only) |
| D8 Fleet & Movement | Generates `MoveTo` commands | D8.1 |
| D9 Space Combat | (no direct call) — AIs predict combat outcomes based on fleet strength | (heuristic) |
| D10 Ground Combat | (no direct call) — AIs predict ground combat for invasion decisions | (heuristic) |
| D11 Diplomacy | Generates `ProposeTreaty`, `DeclareWar` commands | D11 |
| D12 Espionage | Reads `Player.intelLedger`; generates `AssignMission` commands | D12 |

D13 imports from every other section (it's the orchestrator of AI behavior).

| Section | What it imports from D13 |
|---|---|
| D4 Turn Cycle | Calls D13 to generate AI commands |
| A1 Store | Calls D13 |
| P14 AI Inspector | Displays AI reasoning logs |

## Quint-spec-sized leaves (the actual implementation units)

Three Quint files:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/ai/aiCommon.qnt` | D13.1 (scaffolding, common decisions) | `src/domain/ai/aiCommon.ts` | ~300 |
| `specs/ai/aiStrategies.qnt` | D13.2 + D13.3 + D13.4 + D13.5 + D13.6 (strategies as weight presets) | `src/domain/ai/aiStrategies.ts` | ~250 |
| `specs/ai/aiPipeline.qnt` | D13.7 (orchestrator) | `src/domain/ai/aiPipeline.ts` | ~80 |

Total ~630 lines of Quint. The strategies file is short because each strategy is just a personality preset (~50 lines), and the heavy lifting is in `aiCommon.qnt`.

No top-level orchestrator beyond `aiPipeline.qnt` — D4 calls it directly.

## What "simple v1" looks like for D13

- **5 strategies**, each defined by a weight vector + a few custom heuristics.
- **No search / minimax** — AIs use heuristic scoring of candidates, not game-tree search.
- **AIs remember grudges and threats** (in `AIMemory`), but no deeper history.
- **AIs run sequentially**, not in parallel. (Parallelism is v2.)
- **AIs are deterministic given the same state** — no internal randomness except via the `ctx.rng` parameter.
- **All AIs use the same shared decision logic** (D13.1). Strategies are just presets.

## Resolved decisions for D13

- **5 strategies**, each a weight preset + custom heuristics.
- **Strategy selection *is* the difficulty mechanism** in v1 (no per-AI difficulty slider; that's v2).
- **No search / minimax in v1.** Heuristic scoring only.
- **AIs run sequentially**, ordered by **threat** (per-player pressure, not D14.4's score).
- **AIs use `ctx.rng`** for any randomness (per Architecture Principle 1).
- **`AIMemory` is per-player, persists across turns** (stored on `Player.aiMemory`); includes `targetPlayer` for the Aggressive strategy.
- **Grudges expire after 50 turns** (default).
- **`WeightVector` fields sum to 1.0 by construction** (constraint enforced by constructors; tests assert it).
- **All strategies share the common decision logic** in D13.1.

## Open questions for D13

- **AI intelligence about player state**: do AIs share information via intel or just observe what they can see? **Default v1: AIs observe their own state and their `intelLedger` from D12.** They don't know enemy fleet strengths unless they sent a spy.
- **AI cooperation**: do allied AIs coordinate? E.g., if two AIs are allied, do they share fleets? **Default v1: no coordination** — each AI acts independently. v2: alliance coordination.
- **AI colonies on captured planets**: does the AI keep native populations or exterminate them (Ruthless trait)? **Default v1: keep them**; population is population.
- **AI scoring of "value"**: how do AIs score things like "build a Cruiser" vs "build a Factory"? The default heuristic: `value = immediate_benefit × race_modifier`. **Default v1: this heuristic; tune during playtesting.**

No open questions block starting the D13 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D14 Victory Conditions** (small focused section; depends on D1, D11 which are done).

D13's Quint spec can be written **now** — D13's dependencies (D1, D3, D5, D6, D8, D9, D10, D11, D12) are all done.