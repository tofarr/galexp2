# Planning — Master Plan

This is the master planning document for the Galexp2 (Master of Orion reimagining) project. It is intentionally lightweight: each major decision lives in its own dedicated document, and this file tracks the overall phasing, status, and workflow.

## Phasing

The project is structured in five phases. We do **not** skip ahead — earlier phases produce artifacts that later phases consume.

### Phase 1 — Scope and decomposition
Status: **scope decisions complete, decomposition in progress**.

- Decide the technology stack → [`ARCHITECTURE.md`](ARCHITECTURE.md) ✅ done
- Top-level decomposition of the game → [`DECOMPOSITION.md`](DECOMPOSITION.md) ✅ done
- Recursive decomposition into manageable chunks → in progress (starts with D9)
- Testing strategy → [`TESTING.md`](TESTING.md) ✅ done
- Phasing roadmap → [`ROADMAP.md`](ROADMAP.md)

Deliverables of this phase:
1. A single agreed-upon tech stack. ✅
2. A tree of game sections where every leaf is small enough to be specified and implemented in a single focused work session. (top-level done; per-section docs in progress)
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
- Dexie schema for IndexedDB
- Save / load game state
- Zustand store wired to Dexie persistence
- Save-game versioning and migration

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

## Open questions

See the chat conversation for current open questions. Once resolved, they get converted into decisions here and detailed in their owning doc.
