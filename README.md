# Galexp2 — Master of Orion Reimagining

A faithful reimagining of the 1993 turn-based 4X space strategy game **Master of Orion**, built to run entirely in a modern web browser with no server-side components.

## Status

**Planning phase.** Top-level scope and decomposition are done. Per-section recursive decomposition is in progress (starting with Space Combat as pilot). No Quint specifications or TypeScript code has been written yet.

## Goals

- Capture the core mechanics of the original Master of Orion (1993):
  - Procedurally generated galaxy
  - 10 races with distinct traits
  - Planetary colonization, population, food/industry/research economy
  - Multi-tree research (weapons, propulsion, construction, etc.)
  - Ship design and fleet management
  - Tactical space combat and ground invasion combat
  - Diplomacy, trade, treaties, alliances
  - Espionage and sabotage
  - Multiple AI opponents with personalities
  - Council / Galactic Emperor diplomatic victory
- Modern browser-native UX: fast, accessible, responsive
- Single-player only, no server, state persisted in IndexedDB
- **Quint-first**: game rules specified in Quint, validated via simulation and model checking, implemented in TypeScript that mirrors the spec
- **Subsystem isolation is the top architectural constraint**: every domain chunk is a pure function, testable in isolation without booting the UI, store, or persistence

## Methodology

1. **Scope** — decide what's in and what's out ([`ARCHITECTURE.md`](docs/ARCHITECTURE.md))
2. **Decompose** — break the game into discrete sections, recursively, into chunks that can each be implemented in 1–3 days ([`DECOMPOSITION.md`](docs/DECOMPOSITION.md))
3. **Specify** — write Quint formal specs for each chunk, plus unit tests in Quint
4. **Simulate** — run Quint's simulator to flush out bugs in the rules before implementation
5. **Implement** — write TypeScript that mirrors the Quint spec, unit-test it with Vitest ([`TESTING.md`](docs/TESTING.md))
6. **Compose** — wire sections together, run integration tests

## Repository layout

```
README.md              ← this file
docs/
  PLANNING.md          ← master planning doc, phasing, decision log
  ARCHITECTURE.md      ← tech stack & system architecture
  DECOMPOSITION.md     ← game section breakdown (Domain/Application/Presentation/Infrastructure)
  TESTING.md           ← testing strategy, subsystem isolation rules
  ROADMAP.md           ← iteration plan (pending)
  sections/            ← per-section detailed docs (D9 pilot in progress)
specs/                 ← Quint formal specifications (created later)
src/                   ← TypeScript implementation (created later)
```

## Contributing / Next steps

The current focus is recursive decomposition of the domain sections, one section per commit. See [`docs/ROADMAP.md`](docs/ROADMAP.md) for the ordered list and [`docs/PLANNING.md`](docs/PLANNING.md) for the active decisions.
