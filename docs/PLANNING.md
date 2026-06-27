# Planning — Master Plan

This is the master planning document for the Galexp2 (Master of Orion reimagining) project. It is intentionally lightweight: each major decision lives in its own dedicated document, and this file tracks the overall phasing, status, and workflow.

## Phasing

The project is structured in five phases. We do **not** skip ahead — earlier phases produce artifacts that later phases consume.

### Phase 1 — Scope and decomposition
Status: **scope decisions complete; D-layer decomposition complete (D1–D14); A-layer and P-layer decomposition pending**.

- Decide the technology stack → [`ARCHITECTURE.md`](ARCHITECTURE.md) ✅ done
- Top-level decomposition of the game → [`DECOMPOSITION.md`](DECOMPOSITION.md) ✅ done
- Recursive decomposition into manageable chunks → D-layer complete (D1–D14); A-layer and P-layer next
- Testing strategy → [`TESTING.md`](TESTING.md) ✅ done
- Phasing roadmap → [`ROADMAP.md`](ROADMAP.md) ✅ done (refreshed to reflect D-layer completion)

Deliverables of this phase:
1. A single agreed-upon tech stack. ✅
2. A tree of game sections where every leaf is small enough to be specified and implemented in a single focused work session. ✅ for the D-layer; pending for A-layer, P-layer, I-layer.
3. A phasing roadmap that tells us in what order to tackle the leaves.

### Phase 2 — Quint specification
Once decomposition is stable, write Quint specs for the domain layer:
- Galaxy generation
- Race definitions and trait effects
- Turn cycle (economy, research, production, fleet movement, contact resolution)
- Research / tech tree
- Ship design and stats
- Tactical space combat resolution
- Ground combat resolution
- Diplomacy state machine
- Espionage mission resolution
- AI decision logic (or at least its decision *interface*)

For each spec:
- Define the state, actions, and `step` operator
- Add invariants (e.g. "the sum of fleet sizes is conserved")
- Add unit tests using Quint's `test` blocks
- Run the simulator to random-explore
- Run the model checker on the small cases

### Phase 3 — Domain implementation
Translate the Quint specs into TypeScript. The goal is *fidelity*, not cleverness: the TypeScript should be a near-mechanical rendering of the Quint.

- One TypeScript module per Quint spec
- Use `quint compile` to generate TS types where useful
- Use Vitest for unit tests that mirror the Quint tests

### Phase 4 — Persistence and application layer
- A1 Game State Store: Zustand store wired to Dexie persistence (`src/application/store.ts`).
- A2 Persistence: Dexie schema, save/load serializer/deserializer, migration framework.
- A3 Input Commands: typed command catalog, validation, per-turn command queue.
- A4 Turn Manager: orchestrator that collects player+AI commands, runs `step` via D4, persists state, handles save migration on load.
- A5 Event Log: append-only event stream from the domain, with aggregation/filtering for UI consumption.
- Save-game versioning handled by `migrate(v_nState) -> v_(n+1)State`; D4's `initTurn` calls `migrateIfNeeded` before each step (cheap insurance).

### Phase 5 — Presentation and integration
- React UI for each screen (star map, planet, research, fleet, diplomacy, etc.)
- Tactical combat renderer (Canvas)
- Input handling, navigation
- Splash / menu / settings screens
- End-to-end tests via Playwright

## Recursive decomposition workflow

When decomposing a section into chunks:

1. **State the section's contract** — one sentence describing what this section owns and what it doesn't.
2. **List the responsibilities** — bullet list of behaviors the section must implement.
3. **Group responsibilities into chunks** — each chunk is a leaf if it can be:
   - Specified in Quint in roughly 50–300 lines
   - Implemented in TypeScript in roughly 100–500 lines
   - Tested in isolation (no flaky cross-section dependencies)
4. **Document dependencies** — each chunk lists which other chunks it depends on. The dependency graph must be acyclic.
5. **Sequence** — assign a phase and ordering relative to other chunks.

The decomposition lives in [`DECOMPOSITION.md`](DECOMPOSITION.md). When a section is decomposed, it gets its own sub-document under `docs/sections/`.

## Decision log

Decisions get appended here with date and short rationale. Full reasoning lives in the relevant doc.

| Date       | Decision | Rationale |
|------------|----------|-----------|
| 2026-06-05 | Repository layout established | Documents-first workflow for planning phase |
| 2026-06-05 | Quint-first methodology adopted | Game rules naturally fit state-machine specs; simulator catches bugs early |
| 2026-06-05 | PixiJS chosen for star map and tactical combat | Mature 2D renderer; React+SVG covers UI screens |
| 2026-06-05 | Four-layer decomposition (Domain/Application/Presentation/Infrastructure) | Makes dependency direction explicit; aligns with subsystem isolation goal |
| 2026-06-05 | Subsystem isolation is top-priority architectural constraint | Every domain chunk testable without booting UI/store/persistence |
| 2026-06-05 | All 35 chunks ship in v1, simpler implementations allowed | Contracts are stable; v2 experiments swap impls behind same interface |
| 2026-06-05 | D9 (Space Combat) chosen as pilot for recursive decomposition | Most algorithmically complex; most isolated; most fun to playtest |
| 2026-06-05 | D9: auto-resolve is part of v1 | Needed under the hood for AI-vs-AI combat; both auto and tactical paths share the same resolveSpaceCombat function |
| 2026-06-05 | D9: combat is always 2-sided | Fleets of the same owner at the same star auto-merge (D8.4 produces one fleet per side); when 3+ sides contest a star in the same turn, sides are paired randomly per pairing |
| 2026-06-05 | D9: no boarding in v1 | Reserved as D9.7 if revisited |
| 2026-06-05 | D9: no stellar converters / planet busters in v1 | Optional v2 chunks |
| 2026-06-05 | D2: D2.5 selects candidate homeworlds only; final assignment is D3.5 | Keeps D2 focused on world generation; D3 brings preference logic |
| 2026-06-05 | D3: trait modifiers are summed, not multiplied | Simpler reasoning; matches MoO behavior |
| 2026-06-05 | D3: modifiers form an open ADT (new kinds addable without changing consumers) | Lets us discover needed modifiers as we implement |
| 2026-06-05 | D3: no government types in v1; no custom races in v1; no race-specific hulls in v1 | All deferred to v2 |
| 2026-06-05 | D7: tech-prereq enforcement lives in D5/P6, not D7.4 | Keeps D7.4 purely structural; UX prereq feedback still possible via UI |
| 2026-06-05 | D7: per-instance modifiers (race traits, owner tech) applied at the consumer, not D7.5 | Keeps D7.5 pure per-design; trivially testable |
| 2026-06-05 | D7: special catalog includes placeholder entries for v2 specials | Forward-compatible; v2 turns on real effects without catalog changes |
| 2026-06-05 | D8: D8.2 reframed as "distance & range" (Euclidean warp; no warp lanes in v1) | Warp lanes deferred to v2 |
| 2026-06-05 | D8: same-owner auto-merge happens in D8.4 (not a manual command); split is player-initiated | Keeps D9's contract simple — D9 only sees one fleet per side |
| 2026-06-05 | D8: fleet warpRange is the min of ship ranges | Fleet moves as slow as its slowest ship |
| 2026-06-05 | D8: random shuffling takes rng parameter (no Math.random) | Per Architecture Principle 1 |
| 2026-06-05 | D4: phase order is init → economy → research → production → fleetMovement → contactResolution → combatResolution → espionage → diplomacy → victoryCheck → endTurn | Espionage before diplomacy so spy results inform this turn's offers; victory before cleanup so win ends mid-step |
| 2026-06-05 | D4: no partial rollback in v1 (a phase error fails the whole step) | Simpler; v2 can add per-phase rollback |
| 2026-06-05 | D4: AI commands batched with human commands (no mid-turn AI execution) | Matches v1 simplicity |
| 2026-06-05 | D5: linear population growth (no logistic curve) | v1 simplicity |
| 2026-06-05 | D5: single-item production queue per planet | Matches MoO; multi-item queues deferred to v2 |
| 2026-06-05 | D5: one tax slider (income vs research split); default 30 | v1 simplicity |
| 2026-06-05 | D5: debt reduces morale rather than capping treasury at zero | Players can go negative; v1 doesn't enforce solvency |
| 2026-06-05 | D5: Lithovore ignores food (no starvation) | Trait interaction lives in D3 modifiers, applied in D5.1 |
| 2026-06-05 | D6: 6 tech trees × 6 levels = 36 techs in v1 | Roughly MoO's shape; variety without slog |
| 2026-06-05 | D6: single active research per player; switching loses accumulated points | Matches MoO behavior; v2 could allow carry-over |
| 2026-06-05 | D6: prereq-strict trade (recipients must have own prereqs) | No "gift without prereq" in v1 |
| 2026-06-05 | D6: TechEffect ADT is open (unlocks, stat bonuses, production bonuses, special bonuses) | Same principle as D3.2 Modifier ADT; new effect kinds addable without consumer changes |
| 2026-06-05 | D10: no bombardment in v1 (only invasion) | Bombardment deferred to v2 |
| 2026-06-05 | D10: no forced treaties from conquest in v1 | Forced-treaty mechanic deferred to v2 (would live in D11 or D14) |
| 2026-06-05 | D10: MAX_GROUND_ROUNDS = 30 prevents stalemates | After 30 rounds, draw; invader must withdraw |
| 2026-06-05 | D10: Subterranean trait gives "no surrender until fully eliminated" | Defensive trait bonus |
| 2026-06-05 | D10: pillage damages but doesn't destroy buildings (auto-repair next turn) | Buildings recover without player intervention |
| 2026-06-05 | D11: relation score drifts toward 0 without contact (1/turn after 10 turns of no contact) | Cooling relations are realistic |
| 2026-06-05 | D11: wars end by peace treaty or conquest (no automatic surrender) | Players must negotiate; v2 could add surrender |
| 2026-06-05 | D11: Council every 25 turns, simple majority election | MoO-like cadence; v2 could add denouncement |
| 2026-06-05 | D11: trade routes created manually by player command | No auto-trade; v2 could add AI-suggested routes |
| 2026-06-05 | D11: subjugation is one-way in v1 (no liberation mechanic) | Simpler; vassals stay vassals |
| 2026-06-05 | D11: NAP enables trade at reduced income | Trade Pact gives full income; NAP gives partial |
| 2026-06-05 | D12: StealTech produces intel only (not tech transfer) | Espionage gives info; diplomacy gives tech |
| 2026-06-05 | D12: most missions resolve in 1 turn (StealTech takes 3) | Trade-off between risk and reward |
| 2026-06-05 | D12: spy skill caps at 10 | Max effectiveness; tune during playtesting |
| 2026-06-05 | D12: detected spy is captured (no escape mechanic) | v1 simplicity |
| 2026-06-05 | D12: AssassinateLeader is a stub event in v1 (no leader system) | Will affect governance/AI when leaders land |
| 2026-06-05 | D13: 5 strategies, each a weight preset + custom heuristics | Strategies are personality presets, not full search AIs |
| 2026-06-05 | D13: no search/minimax in v1 (heuristic scoring only) | v2 could add real game-tree search |
| 2026-06-05 | D13: AIs run sequentially, ordered by threat | Parallelism deferred to v2 |
| 2026-06-05 | D13: AIMemory persists across turns (per-player) | Stores grudges, threats, target stars |
| 2026-06-05 | D13: all AIs use shared decision logic (D13.1) | Strategies are just weight overrides |
| 2026-06-05 | D14: four victory kinds (Conquest, Tech, Diplomatic, Score) | MoO-style mix |
| 2026-06-05 | D14: conquest threshold 80% planets + no rival above 10% OR last empire standing | Two conditions for robustness |
| 2026-06-05 | D14: tech victory = 6 key techs (top of each tree) | Simpler than specific named techs |
| 2026-06-05 | D14: MAX_TURN = 200 for score/time victory | v1 default; tunable |
| 2026-06-05 | D14: ties in score = draw (no winner) | v1 simplicity |
| 2026-06-05 | D-layer complete (D1-D14 all decomposed) | Phase 1 next moves to A-layer |
| 2026-06-05 | D5: revolt events are emitted but don't flip ownership in v1 | Full revolt mechanics deferred to v2 |
| 2026-06-05 | A2: save migration happens in D4's `initTurn` via `migrateIfNeeded(state)` | Cheap insurance; spec phase locks the contract |
| 2026-06-05 | D4: AI scheduling resolved — AIs are batched with human commands; A4 Turn Manager calls D13 once per turn before D4 step | AI commands may use last turn's intel (D12 ran in previous turn); "espionage informs offers" is interpreted as "this turn's offers can use the latest available intel ledger" |
| 2026-06-05 | D4: pause-on-detection deferred to v2 / UI concern | UI handles "interesting event" surfacing after `step`; no domain-level pause |
| 2026-06-05 | D10: ground combat is phase 7b in D4's step (between `combatResolution` and `espionage`); runs once per invasion order, sequentially in fleet-arrival order | Multi-invasion per planet is handled by the per-invasion loop; D8.4 arrival events feed D10 |
| 2026-06-05 | D11/D14 cycle broken: `Player.score` becomes a derived field, recomputed once per turn by a helper called from D4 | D11.5 council voting and D14.4 endgame scoring both read this single derived value; no circular import |
| 2026-06-05 | D11: relation score is `clamp(rawScore, -100, 100)` after all modifiers | Doc had no clamp; now explicit |
| 2026-06-05 | D11: trade route income is computed at end-of-turn (D11.6) and cached on `TradeRoute.income`; D5.5 reads the cached value from the previous turn | Phase order (economy before diplomacy) makes "compute in D11, read in D5" impossible without caching |
| 2026-06-05 | D11: research agreement is a per-turn transfer applied after D6.3 tech-cost deduction; each partner receives 50% of the other's unspent research | Mechanism pinned to a phase boundary; otherwise the "50%" was floating |
| 2026-06-05 | D11: subjugation in v1 is mechanical — vassal is auto-set to `AtWar` with suzerain's enemies and pays 10% of gross income to suzerain per turn in D5.5 | No AI-heuristic hand-waving |
| 2026-06-05 | D13: strategy selection *is* the difficulty mechanism in v1 (no weight scaling per difficulty) | Strategy choice gives variety; difficulty sliders deferred to v2 |
| 2026-06-05 | D8: ETA is locked at order issue; range upgrades do not recompute ETA | Doc had contradictory body text; open question promoted to decision |
| 2026-06-05 | D7/D8: `CombatStats.range` removed; per-ship warp range comes from `Hull.baseWarpRange` (new field); v1 ignores HyperDrive specials | Range=1 for every ship was a placeholder; now each hull carries its own range |
| 2026-06-05 | D2.6: candidate homeworld filter expanded to `{Terrestrial, Oceanic, Arid, Tundra, Barren, Volcanic}` | Lets Bulrathi/Klackons/Sakkra/Darloks/Mrrshan get preferred-type homeworlds without always-fallback |
| 2026-06-05 | D9: `BattleResolvedEvent { star, winner, loser, turn }` emitted by D9.6 on combat end; D14.5 increments `battlesWon`/`battlesLost` | Required for end-game stats |
| 2026-06-05 | D9: stance reserved as data in v1 (D8.1 defines it; D9 does not consume it); D9.2 targeting consumes it in v2 | Removes dead-data concern |
| 2026-06-05 | D9: D9.6.3 (XP) and D9.6.4 (scrap) deferred to v2; chunks defined for spec completeness, no v1 effect | Reconciles chunk list with "no XP, no scrap" v1 simplification |
| 2026-06-05 | D9: initiative is per-ship (sorted by computer, ties by hull size) — explicit v1 simplification vs MoO per-stack | Per-stack is v2 |
| 2026-06-05 | D1: `Building.baseEffect` is a typed ADT (FoodOutput, IndustryOutput, ResearchOutput, TradeIncome, EntertainmentBonus, DefenseBonus, IntelligenceCenter, ShipRepairBonus) | Replaces vague `baseEffect` with explicit per-output kinds consumers pattern-match on |
| 2026-06-05 | D3: `Modifier` ADT enumerated fully — `GroundAttack`, `GroundDefense`, `CounterEspionage`, `GrowthMod`, etc. added; building/planet fields (foodOutput, intelligenceCenterCount, garrison) live in `Building.baseEffect` / `Planet`, not as race modifiers | Separates race modifiers (D3.2) from building effects (D1.2) |
| 2026-06-05 | D11: relation drift toward 0 begins after 10 turns without contact, at 1 point/turn starting at turn 11 | Doc's "10 turns then drift" was ambiguous about when drift starts |
| 2026-06-05 | D1: `BC_PER_TAX_POINT = 1` constant added (1 bc per tax point per turn is the v1 scale) | Pins treasury scale referenced by D5.5, D11.3, D14 |
| 2026-06-05 | D13: `WeightVector` fields sum to 1.0 by construction (constraint documented; constructors enforce it) | Implicit in existing examples; now explicit |
| 2026-06-05 | D13: `AIMemory.targetPlayer: Option<PlayerId>` added (used by D13.2 Aggressive strategy) | Doc referenced the field but didn't declare it |
| 2026-06-05 | D5: `RevoltRiskEvent` is emitted by D5.1 (population growth chunk) when morale is low; D5.6 just computes the morale value | Doc had three conflicting statements of the emission site |
| 2026-06-05 | D4: `step` passes `cmds` to every phase that consumes commands (economy, research, production, espionage, diplomacy); other phases get `(state, ctx)` only | Doc's pseudocode inconsistently dropped `cmds` from some phases |

## Open questions

See the chat conversation for current open questions. Once resolved, they get converted into decisions here and detailed in their owning doc.
