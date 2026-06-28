# Section D4 — Turn Cycle (the master `step`)

D4 is the **orchestrator** — it doesn't compute anything itself; it calls each domain subsystem in the right order and threads state through them. Small but central: every other section is reachable from here.

## Section contract

> **Turn Cycle**: given `(state, playerCommands, aiCommands) -> newState`. Defines the order of phases, the contract each phase obeys, and the turn boundary cleanup.

**Owns**:
- The master `step` operator (one turn's worth of computation).
- The fixed phase ordering.
- The phase protocol: each phase is a function `(state, commands, ctx) -> (newState, events)`; phases are pure.
- Turn boundary bookkeeping (init / cleanup helpers).

**Does not own**:
- Any phase's implementation (D5, D6, D8, D9, D11, D12).
- AI command generation (D13) — D4 only consumes the commands D13 produced.
- Persistence (A2).
- The `GameState` shape (D1) — D4 reads/writes it but doesn't define it.

## Top-level chunks

Three top-level chunks from `DECOMPOSITION.md`. Each is small; together they fit in one Quint file.

- **D4.1 Phase ordering** — the sequence of calls in one `step`.
- **D4.2 Turn boundary** — init (turn-counter increment, per-turn flag reset) and end-of-turn cleanup.
- **D4.3 Event emission contract** — every phase returns events; the orchestrator appends them to the event log.

## Recursive decomposition

D4 is intentionally small — three chunks, each leaf-sized. Sub-decomposition just clarifies the phase protocol.

### D4.1 → Phase ordering

The fixed phase order is locked in for v1:

```
1. initTurn          (D4.2) — reset per-turn flags, migrate if needed
2. economy           (D5)   — food/industry/research/income per colony
3. research          (D6)   — accumulate research, emit tech-acquired events
3a. applyTreatyTransfers (D4 helper) — research-agreement 50/50 split, subjugation-tribute prep (read from D6.3's post-deduction state)
4. production        (D5.7) — consume industry, emit new ships/buildings
4a. recomputeScore   (D4 helper) — write state.score[player] via D14.4's computeScore (cached for D11.5 council vote + D14.4 endgame)
5. fleetMovement     (D8.3) — advance in-transit fleets
6. contactResolution (D8.4) — detect arrivals, pair encounters, emit FleetEvents
7. combatResolution  (D9)   — resolve Encounters into CombatEvents
8. groundCombat      (D10)  — resolve ground invasions at each conquered star (only if any)
9. espionage         (D12)  — resolve spy missions
10. diplomacy         (D11)  — process offers, update treaties, run council election
11. victoryCheck     (D14)  — does anyone win this turn? (runs *before* end-of-turn cleanup)
12. endTurn          (D4.2) — final cleanup, emit "end of turn N" event
```

The order has these notable decisions baked in:

- **Espionage runs before diplomacy.** Espionage results can influence this turn's diplomatic posture, but not the reverse. (AI offers were generated *before* `step` using last turn's intel; "espionage informs this turn's offers" is interpreted as "offers can use the latest intel ledger, which was refreshed this turn.")
- **Victory check runs before end-of-turn cleanup.** If someone won, the game ends mid-step rather than after cleanup. (Cleaner handoff to the post-game screen.)
- **Ground combat is phase 8**, between space combat and espionage. It runs once per invasion order, sequentially in fleet-arrival order. Multi-invasion on the same planet is handled by the per-invasion loop.
- **Save migration happens inside `initTurn`**: `initTurn` calls `migrateIfNeeded(state)` before any other work (cheap insurance; spec phase locks the contract).

Each phase is a call:

```
phase(state, commands, ctx) -> (state', events)
```

The orchestrator threads state through them:

```
step(s, cmds, ctx) =
    let (s1, e1)   = initTurn(s, ctx)              in
    let (s2, e2)   = economy(s1, cmds, ctx)        in    // cmds: SetTaxRate, SetQueue, etc.
    let (s3, e3)   = research(s2, cmds, ctx)       in    // cmds: SetResearch, ClearResearch
    let (s4, e4)   = production(s3, cmds, ctx)     in    // cmds: SetQueue updates
    let (s5, e5)   = fleetMovement(s4, ctx)        in
    let (s6, e6)   = contactResolution(s5, ctx)    in
    let (s7, e7)   = combatResolution(s6, ctx)     in
    let (s8, e8)   = groundCombat(s7, cmds, ctx)   in    // cmds: Invade per fleet
    let (s9, e9)   = espionage(s8, cmds, ctx)      in    // cmds: AssignMission
    let (s10, e10) = diplomacy(s9, cmds, ctx)      in    // cmds: ProposeTreaty, DeclareWar, etc.
    let (s11, e11) = victoryCheck(s10, ctx)        in
    let (s12, e12) = endTurn(s11, ctx)             in
    (s12, flatten([e1, e2, ..., e12]))
```

The orchestrator doesn't know what's inside each phase. It just threads state and collects events. Phases that accept commands receive `cmds`; phases that don't (initTurn, endTurn, fleetMovement, contactResolution, combatResolution, victoryCheck) ignore it. `groundCombat` accepts `cmds` only because `Invade` orders are conditional commands dispatched at the start of D10.

### D4.2 → Turn boundary

Two helpers, both pure:

**`initTurn(state, ctx) -> (state', events)`**:
- Call `migrateIfNeeded(state)` to upgrade any loaded save to the current schema version (cheap insurance; spec phase locks the migration framework).
- Increment `state.turn`.
- Reset per-turn flags (e.g., `fleet.movedThisTurn = false`, `player.researchedThisTurn = false`).
- Emit `TurnStartedEvent { turn }`.

**`applyTreatyTransfers(state, ctx) -> (state', events)`** (helper, runs between `research` and `production`):
- Reads `Player.researchAccumulated` *after* D6.3 has deducted tech-acquisition cost.
- For each pair `(A, B)` with an active `ResearchAgreement` treaty:
  - For each side, push half of its unspent `researchAccumulated` into the other side's `treasury` at a fixed `RESEARCH_AGREEMENT_RATE = 0.5` (D1.1). Source: `player.researchAccumulated`. Sink: `otherPlayer.treasury`. The source side's `researchAccumulated` is decremented accordingly. Total conserved.
- No event emitted (the per-player income event is emitted by D5.5; this helper just adjusts pre-income state). If v2 wants observability, add a `ResearchTransferredEvent`.

**`recomputeScore(state, ctx) -> (state', events)`** (helper, runs between `production` and `fleetMovement`):
- For each player, write `state.score[player] := computeScore(player, state)` (D14.4's pure function).
- No event emitted (a no-op in the event log; consumers see the new field on next read).

**`endTurn(state, ctx) -> (state', events)`**:
- Final cleanup (e.g., clear `orders` flags, expire transient state).
- Emit `TurnEndedEvent { turn, summary }`.
- Trim `state.events` to last `MAX_EVENTS_IN_MEMORY` (from D1.1) if exceeded.
- If `state.turn >= MAX_TURN - TURN_WARNING_THRESHOLD` (e.g., 10), emit `ApproachingEndOfGameEvent`.

Both helpers are tiny. They're the only state-mutating code in D4 outside the orchestrator.

### D4.3 → Event emission contract

The contract each phase obeys:

> Every phase function has signature `(state, commands, ctx) -> (state', events)`. The returned `events` list is appended to `state.events` by the orchestrator. Phases never modify `state.events` directly.

This is mostly a **documentation contract** — there's no code in D4.3 beyond `appendEvents` (a one-liner that extends `state.events`). The contract is enforced by code review and by the test suite (each phase's tests assert on its returned events).

The events log itself (the `events: list<Event>` field on `GameState`) is defined in D1.3. Event types (the `Event` ADT with `Combat`, `Diplomacy`, `TechAcquired`, etc. variants) live in D1.2.

D4.3 is the **protocol** — what each phase's signature looks like. The orchestrator's job is to enforce this by typing.

## Dependency graph (within D4)

D4 has no internal dependencies; each chunk is independent. The only dependency is on D1 (for `GameState`, `Event`, etc.).

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `GameState`, `Event`, `Command`, `Ctx` (rng, time, etc.) | D1.2, D1.3 |
| D5 Economy | `economy(state, commands, ctx)` (called from step) | D5 |
| D6 Research | `research(state, ctx)` | D6 |
| D5.7 Production | `production(state, ctx)` | D5.7 |
| D8 Fleet Movement | `fleetMovement`, `contactResolution` | D8.3, D8.4 |
| D9 Space Combat | `combatResolution` | D9 |
| D11 Diplomacy | `diplomacy(state, commands, ctx)` | D11 |
| D12 Espionage | `espionage(state, commands, ctx)` | D12 |
| D14 Victory | `victoryCheck(state, ctx)` | D14 |

D4 has the most outbound dependencies of any section — that's the nature of an orchestrator.

| Section | What it imports from D4 |
|---|---|
| A1 Game State Store | Calls `step` on End Turn |
| A4 Turn Manager | Uses `step` and `initTurn`/`endTurn` to drive the game loop |
| I1 Quint toolchain | `step` is the canonical Quint operator; the simulator and model-checker target it |

## Quint-spec-sized leaves (the actual implementation units)

One Quint file (D4 is small). The orchestrator is just a fixed sequence of calls:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/orchestrator/turnCycle.qnt` | D4.1 + D4.2 + D4.3 | `src/domain/orchestrator/turnCycle.ts` | ~180 |

The TS module is similarly small — a `step()` function plus `initTurn()` and `endTurn()` helpers. The bulk of the work is in each phase's module that D4 calls.

We do **not** split this into multiple files because D4 is intentionally small and the three chunks are tightly coupled (they share the `step` operator's flow). Splitting would create artificial boundaries.

## What "simple v1" looks like for D4

- **One fixed phase order.** No conditional phases, no skipping phases, no per-game-type variations.
- **No partial rollback.** If a phase errors (returns invalid state), the whole `step` fails. v2 could add per-phase rollback, but that's complex.
- **No interactive phases.** Diplomacy and espionage in v1 are batch-processed at the orchestrator's call; no mid-turn player prompts. (UI handles interactivity via commands queued before `step`.)
- **No parallel phases.** All phases run sequentially. v2 could parallelize independent phases (e.g., economy for different empires), but v1 keeps it simple.
- **No event filtering.** All events flow through. UI can filter at read time.
- **No event ordering guarantees beyond "happens in this phase."** Within a phase, event order is whatever the phase emits. Across phases, the order matches the phase order in `step`.

## Resolved decisions for D4

- **Fixed phase order** as listed above; no skipping or reordering.
- **Espionage before diplomacy** (results can inform this turn's diplomacy, not vice versa).
- **Victory check before end-of-turn cleanup** (game ends mid-step on win).
- **Ground combat is phase 8**, runs once per invasion order sequentially.
- **No partial rollback** in v1.
- **No interactive phases** — UI queues commands, then `step` runs.
- **AI scheduling**: AI commands are generated *before* the human's turn ends by A4 Turn Manager calling D13; submitted along with human commands; processed in one batch by `step`.
- **Pause-on-detection**: deferred to v2 / UI concern. UI surfaces interesting events after `step`.
- **Save migration in `initTurn`** via `migrateIfNeeded(state)`. Cheap insurance.
- **Per-turn flag reset** lives in `initTurn`; final cleanup in `endTurn`.
- **`MAX_TURN` warning threshold** for end-of-game tension.

## Open questions for D4

None remaining. All D4 design questions have been resolved; see [`PLANNING.md` decision log](../PLANNING.md).

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D5 Economy** (touches most other systems; good integration test once D4 is in place).

D4's Quint spec can be written **now** — but it imports from every other section. Each phase is currently a stub. We can either:
- (a) Write `turnCycle.qnt` with stub phase calls (`economyStub`, `combatStub`, etc.) and fill them in as each section lands.
- (b) Skip D4's spec until at least D5 (Economy) and D9 (Space Combat) are specified.

I'd recommend (a) — the orchestrator's structure can be specified once and the stubs replaced. This also gives us a clean place to discover phase ordering issues during spec work.

Awaiting your direction on whether to continue straight to D5 next, or pause for review here.
