# Architecture — Technology & System Design

This document records the agreed technology stack and the architectural principles that follow from it. It is the source of truth for "how is this project structured." Detailed section-level decomposition lives in [`DECOMPOSITION.md`](DECOMPOSITION.md).

## 1. Technology stack

| Concern | Choice | Notes |
|---|---|---|
| Language | **TypeScript 5.x** | Required for Quint's primary backend and for state-machine type safety. |
| Build / dev server | **Vite 5.x** | Fast HMR; pairs with Vitest. |
| UI framework | **React 18.x** | Component model fits MoO's many similar screens. |
| State management | **Zustand** | Tiny store; devtools; middleware for persistence. |
| Persistence | **Dexie 4.x** | Promise-based IndexedDB wrapper; schema migrations; compound indexes. |
| UI rendering (forms, lists, trees) | **SVG via React components** | Planet, tech, diplomacy, fleet list, etc. |
| Map & animated rendering | **PixiJS 7.x** | Star map and tactical space combat. |
| Ground combat rendering | **SVG + React** | Simple sprite layouts; resolution logic is what matters. |
| Formal specs | **Quint** | State-machine specs; simulator; model checker; built-in tests. |
| Unit tests (TS) | **Vitest** | Pairs with Vite; Jest-compatible API. |
| Spec tests | **`quint test`** | Tests live in the `.qnt` file; same harness as the simulator. |
| E2E tests (later) | **Playwright** | Standard. |
| Audio (later) | **Howler.js** | Deferred until core gameplay is playable. |

### What we explicitly do *not* use

- **Phaser, MelonJS, kaboom.js** — real-time game engines; MoO is turn-based.
- **WebGL directly, Three.js** — 2D game; PixiJS handles WebGL when needed.
- **Next.js / Remix** — no server component.
- **Redux / MobX** — Zustand is sufficient.
- **A "TS game engine" framework** — none mature enough for our needs.

## 2. Architectural principles

These principles flow from the goal of *subsystem isolation for independent testing* and from the Quint-first methodology.

### Principle 1: Domain is pure

The **domain layer** contains all game rules. Every domain subsystem is a pure function of the form:

```
(state, commands) -> { state: newState, events: [...] }
```

- No I/O (no DOM, no IndexedDB, no `Math.random()` seeded from external time).
- No global state. All state flows through the function arguments.
- No mutation of inputs.
- Determinism: identical inputs produce identical outputs. Randomness is injected via a `rng: () => number` parameter so tests can supply a fixed seed.

This is the single most important architectural constraint. It is what makes subsystems independently testable.

### Principle 2: Subsystems have stable contracts

Each domain subsystem exposes:
- A **TypeScript module** with a single exported `resolve(state, commands, ctx) -> { state, events }` function.
- A **Quint spec** that defines the canonical behavior of that function.
- A **Vitest test file** that tests the TypeScript implementation against fixtures built from the same data shapes.
- A **Quint test file** (within the spec) that tests the spec itself via simulation.

The contract is the *interface* between the subsystem and its callers. v1 may implement the contract simply (e.g. combat uses a coarse damage formula); later experiments (richer combat with shield facing, weapon mods, etc.) swap in a new implementation behind the same contract.

This is what supports your goal of "experiment with making them richer after initial release."

### Principle 3: Application layer orchestrates, domain layer computes

The **application layer** (turn manager, store, persistence) is the only place that:
- Touches the filesystem / IndexedDB
- Calls into the domain layer
- Emits UI updates in response to domain events

The application layer never contains rules. It composes them.

### Principle 4: UI is a function of state, plus commands

Every React component is a function `(state, dispatch) -> vdom`. Components:
- Read from the Zustand store via selectors.
- Dispatch commands (typed actions) via store actions.
- Never call domain functions directly. The store dispatches commands into the turn manager, which calls domain functions.

This keeps the UI thin and lets us test the domain without rendering.

### Principle 5: Persistence is a snapshot, not a side-channel

Game state is a single immutable value. Saving = serializing that value to IndexedDB. Loading = deserializing. There is no in-place mutation by the persistence layer.

Save-game versioning is handled by:
1. Storing a `version` field on the saved blob.
2. Migration functions `migrate(v1State, v2): v2State`.
3. The latest version is what the live game uses.

This makes the saved game a first-class artifact we can reason about in Quint.

## 3. Data flow

```
                ┌─────────────────────────────────────┐
                │  React UI                           │
                │  - reads from store via selectors   │
                │  - dispatches typed commands        │
                └────────────┬────────────────────────┘
                             │ commands
                             ▼
                ┌─────────────────────────────────────┐
                │  Zustand store + turn manager       │
                │  - queues commands                   │
                │  - on "End Turn": runs subsystems    │
                │  - persists state via Dexie          │
                └────────────┬────────────────────────┘
                             │ (state, commands, ctx)
                             ▼
   ┌──────────┬──────────┬──────────┬──────────┬──────────┐
   │ Galaxy   │ Economy  │ Combat   │ Diplomacy│ AI       │
   │ Gen      │          │          │          │          │  (all pure)
   │          │          │          │          │          │
   └──────────┴──────────┴──────────┴──────────┴──────────┘
                             │ (newState, events)
                             ▼
                ┌─────────────────────────────────────┐
                │  Zustand store (new state)          │
                │  + event log (rendered by UI)       │
                └─────────────────────────────────────┘
```

## 4. Subsystem isolation: testing strategy

This is the **highest-priority architectural concern**. Every domain subsystem must be testable in isolation.

### What "isolation" means

A subsystem test:
- Builds a `state` fixture directly (no need to play through the game).
- Calls the subsystem's `resolve(state, commands, ctx)` function.
- Asserts on the resulting `state` and `events`.

No setup of: UI, store, persistence, full game loop. Just call the function.

### How this is achieved

- **Pure functions** (Principle 1) make this possible.
- **Fixtures live in the data model**, not in test setup hacks. Every test file has helpers like `makePlayer({ race: 'Bulrathi', techs: [...] })` that produce valid game states.
- **Each subsystem test directory** lives next to its source: `src/domain/combat/__tests__/spaceCombat.test.ts` mirrors `src/domain/combat/spaceCombat.ts`.
- **Quint specs double as test blueprints.** The Quint `test` blocks define behaviors; the Vitest tests mirror those behaviors in TypeScript.

### Example: testing space combat

```ts
// src/domain/combat/__tests__/spaceCombat.test.ts
import { resolveSpaceCombat } from '../spaceCombat';
import { makeGame } from '@/test/factories';

it('destroyer beats unshielded fighter', () => {
  const game = makeGame()
    .withFleet('attacker', { ships: [{ hull: 'Destroyer', count: 1 }] })
    .withFleet('defender', { ships: [{ hull: 'Fighter', count: 1 }] })
    .build();

  const { state, events } = resolveSpaceCombat(game, { attacker: ..., defender: ... }, { rng: seedRng(42) });

  expect(state.fleets.defender.ships).toHaveLength(0);
  expect(events).toContainEqual(expect.objectContaining({ type: 'ship-destroyed' }));
});
```

No need to play through colonize → research → design → build → fight. Just construct the relevant slice.

## 5. Repository layout

```
README.md
docs/
  PLANNING.md           ← master plan
  ARCHITECTURE.md       ← this file
  DECOMPOSITION.md      ← game sections & chunks
  TESTING.md            ← testing strategy & subsystem isolation rules
  sections/             ← per-section detailed docs
specs/                  ← Quint specifications
  combat/
    spaceCombat.qnt
    groundCombat.qnt
  economy.qnt
  ...
src/
  domain/               ← pure game rules (mirrors specs/)
    combat/
      spaceCombat.ts
      groundCombat.ts
    economy.ts
    ...
  application/          ← state, persistence, orchestration
    store.ts
    persistence/
      dexie.ts
      migrations.ts
    turnManager.ts
    commands.ts
  presentation/         ← React UI
    screens/
      starMap/
      planet/
      tech/
      diplomacy/
      combat/
      ...
    components/         ← shared UI primitives
  test/
    factories/          ← makePlayer, makeGame, makeFleet, ...
    rng.ts              ← seedable RNG for tests
  assets/
    sprites/
    portraits/
    audio/
  main.tsx
  App.tsx
quint.config.json
package.json
tsconfig.json
vite.config.ts
vitest.config.ts
```

## 6. Quint integration

Quint specs in `specs/` are the canonical definitions of game behavior. TypeScript implementations in `src/domain/` are validated against them.

Workflow per chunk:
1. Write Quint spec in `specs/<area>/<chunk>.qnt` with state, `step`, invariants, and `test` blocks.
2. Run `quint typecheck specs/<area>/<chunk>.qnt` — catches type errors.
3. Run `quint test specs/<area>/<chunk>.qnt` — runs Quint's built-in test suite.
4. Run `quint simulate ...` — random exploration finds bugs (e.g., "AI declares war on itself after 50 turns").
5. Optionally run `quint verify` — model-check invariants on small state spaces.
6. Implement TypeScript to match.
7. Vitest tests in `src/domain/<area>/__tests__/` mirror the Quint tests.

`package.json` scripts wire these up:

```jsonc
{
  "scripts": {
    "quint:typecheck": "quint typecheck specs/",
    "quint:test": "quint test specs/",
    "quint:simulate": "quint simulate --max-samples=1000 --max-steps=100 specs/",
    "ts:test": "vitest",
    "ts:typecheck": "tsc --noEmit",
    "test": "npm run quint:typecheck && npm run quint:test && npm run ts:typecheck && npm run ts:test"
  }
}
```

## 7. Open architectural questions

These remain to be decided as we encounter them:

- **Save format**: JSON for simplicity vs. binary (e.g., MessagePack) for size? Lean JSON for v1; we can revisit.
- **Asset format**: SVG sprites vs. PNG sprites? Lean PNG sprite sheets for PixiJS efficiency, but SVG for UI icons.
- **i18n**: defer until strings stabilize. Out of scope for v1.
- **Multiplayer**: explicitly out of scope per requirements.
