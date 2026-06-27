# Roadmap

The phasing roadmap for Galexp2. This file is the single place to look for "what's next?". Phase boundaries come from [`PLANNING.md`](PLANNING.md); this document sequences the work inside each phase.

We decompose sections in **dependency order**, one section per commit. That way, by the time a section is decomposed, every chunk it depends on already has a contract and a chunk list.

## Phase 1 — Scope and decomposition

Status: in progress. Each row is one commit.

| Order | Section | Why this order | Doc |
|---|---|---|---|
| 1 | **D9 Space Combat** (pilot) | Most algorithmically complex and most isolated; sets the recursive-decomposition template. Done. | [`sections/D9-space-combat.md`](sections/D9-space-combat.md) |
| 2 | **D1 Core Types** | Root — every later section imports from this. Defines the type vocabulary. | [`sections/D1-core-types.md`](sections/D1-core-types.md) (to write) |
| 3 | **D2 Galaxy Generation** | Pure generator; D8 pathfinding depends on galaxy topology. | [`sections/D2-galaxy-generation.md`](sections/D2-galaxy-generation.md) (to write) |
| 4 | **D3 Races & Traits** | Small; needed by D5/D6/D7/D11/D13. | [`sections/D3-races-and-traits.md`](sections/D3-races-and-traits.md) (to write) |
| 5 | **D7 Ship Design & Combat Stats** | D9's main dependency is D7.5 (stat computation). | [`sections/D7-ship-design.md`](sections/D7-ship-design.md) (to write) |
| 6 | **D8 Fleet & Movement** | D9's other dependency: D8.4 (arrival + same-owner merge) reflects the resolved D9 decision. | [`sections/D8-fleet-movement.md`](sections/D8-fleet-movement.md) (to write) |
| 7 | **D4 Turn Cycle** | Small central orchestrator. Now safe to specify because the sections it calls into have contracts. | [`sections/D4-turn-cycle.md`](sections/D4-turn-cycle.md) (to write) |
| 8 | **D5 Economy** | Touches most other systems; good integration test once D4 is in place. | [`sections/D5-economy.md`](sections/D5-economy.md) (to write) |
| 9 | **D6 Research & Tech Tree** | Depends on D1, D3. | [`sections/D6-research.md`](sections/D6-research.md) (to write) |
| 10 | **D10 Ground Combat** | Depends on D1, D8, D9. | [`sections/D10-ground-combat.md`](sections/D10-ground-combat.md) (to write) |
| 11 | **D11 Diplomacy** | Depends on D1, D3. | [`sections/D11-diplomacy.md`](sections/D11-diplomacy.md) (to write) |
| 12 | **D12 Espionage** | Depends on D1, D11. | [`sections/D12-espionage.md`](sections/D12-espionage.md) (to write) |
| 13 | **D13 AI Decision Logic** | Reads from many subsystems; deferred until most of the domain is in. | [`sections/D13-ai.md`](sections/D13-ai.md) (to write) |
| 14 | **D14 Victory Conditions** | Last domain section; depends on D4. | [`sections/D14-victory.md`](sections/D14-victory.md) (to write) |
| 15 | **A-layer** (A1–A5) | Application/orchestration layer; depends on all domain sections. | [`sections/A1..A5.md`](sections/) (to write) |
| 16 | **P-layer** (P1–P12) | Presentation; depends on A-layer. | [`sections/P1..P12.md`](sections/) (to write) |
| 17 | **I-layer** (I1–I4) | Infrastructure (Quint toolchain, test harness, asset pipeline, build/deploy). Decompose when toolchain is set up. | [`sections/I1..I4.md`](sections/) (to write) |

Phase 1 deliverables, restated from PLANNING.md:
1. A single agreed-upon tech stack. ✅
2. A tree of game sections where every leaf is small enough to be specified and implemented in a single focused work session.
3. This roadmap. ✅ (this file)

## Phase 2 — Quint specification

Trigger: every D-section above has a leaf-sized chunk list.

Order within Phase 2 (again, dependency-driven):
1. `specs/core/types.qnt` — mirror D1.
2. `specs/galaxy/galaxyGen.qnt` — mirror D2.
3. `specs/races/races.qnt` — mirror D3.
4. `specs/ships/shipDesign.qnt` (+ `combatStats.qnt`) — mirror D7.
5. `specs/fleet/fleet.qnt` — mirror D8.
6. `specs/combat/spaceCombat.qnt` (+ targeting/damage/specials/retreat/outcome) — mirror D9. This is the pilot spec.
7. ...then D4, D5, D6, D10, D11, D12, D13, D14.

For each spec: define state + actions + `step`; add invariants; add Quint `test` blocks; run the simulator and model-checker.

## Phase 3 — Domain implementation

Trigger: corresponding Quint spec is stable.

Translate each spec into TypeScript, 1 module per spec. Run `quint compile` for types where useful. Mirror Quint tests in Vitest.

## Phase 4 — Persistence and application

- Dexie schema (A2.1)
- Serializer / deserializer / migrations (A2.2–A2.4)
- Zustand store wired to Dexie (A1, A2.5)

## Phase 5 — Presentation and integration

- React UI for each screen (P3–P12)
- PixiJS renderers for star map (P3.1–P3.7) and tactical combat (P10.1–P10.5)
- Splash / menus / settings (P2)
- Playwright end-to-end tests

## Working agreements

- **One section per commit.** Each decomposition lands in its own commit on `planning-foundation`, so it's reviewable in isolation.
- **Stop and review at each commit.** User reviews the doc before the next section is started.
- **Open questions in a section become decisions in PLANNING.md once resolved.** A section is not "decomposed" until its open questions are resolved or explicitly deferred.
- **v2 chunks are not in this roadmap.** They're listed as future work in their section docs and picked up after v1 ships.
