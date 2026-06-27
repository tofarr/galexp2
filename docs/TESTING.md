# Testing — Strategy & Subsystem Isolation Rules

Testing is the **highest-priority architectural concern** for this project. Per [`ARCHITECTURE.md`](ARCHITECTURE.md), every domain subsystem must be independently testable. This document spells out how.

## 1. The golden rule

> **No subsystem test should require booting the game UI, the store, or persistence.**

Every domain subsystem is a pure function. A test calls it with hand-built fixtures. Done.

If you ever find yourself reaching for a mock of the store, the persistence layer, or React in a domain test, that's a signal the domain layer leaked. Fix the leak, not the test.

## 2. Test pyramid

```
                ╱╲
               ╱  ╲         E2E (Playwright)
              ╱ UI ╲        — full game flow, smoke tests
             ╱──────╲
            ╱        ╲      Integration (Vitest)
           ╱ subsystem╲     — domain subsystems wired together, no UI
          ╱────────────╲
         ╱              ╲   Unit (Vitest + Quint `test`)
        ╱   per-chunk    ╲  — one chunk, fixture inputs, expected outputs
       ╱──────────────────╲
```

For v1 we focus on the bottom two layers. Playwright comes in Phase 5.

## 3. Two parallel test tracks: Quint and TypeScript

Every domain chunk has **both** a Quint test suite and a TypeScript (Vitest) test suite. They must agree.

### Quint tests (in `specs/<area>/<chunk>.qnt`)

```quint
module spaceCombat {
  // ... state, step, etc.

  run resolveSpaceCombatTest =
      init
      .then(resolveSpaceCombat(destroyerVsFighter))
      .expect(fleets.defender.ships.length() == 0)
}
```

Run via `quint test specs/spaceCombat.qnt`.

### TypeScript tests (in `src/domain/<area>/__tests__/<chunk>.test.ts`)

```ts
import { describe, it, expect } from 'vitest';
import { resolveSpaceCombat } from '../spaceCombat';
import { makeGame } from '@/test/factories';

describe('resolveSpaceCombat', () => {
  it('destroyer beats unshielded fighter', () => {
    const game = makeGame()
      .withFleet('attacker', [{ hull: 'Destroyer', count: 1 }])
      .withFleet('defender', [{ hull: 'Fighter', count: 1 }])
      .build();

    const { state, events } = resolveSpaceCombat(game, {
      attacker: 'attacker',
      defender: 'defender',
    }, { rng: seedRng(42) });

    expect(state.fleets.defender.ships).toHaveLength(0);
    expect(events).toContainEqual(
      expect.objectContaining({ type: 'ship-destroyed' })
    );
  });
});
```

### Cross-checking

When implementing, if the TS test passes but the Quint test fails (or vice versa), **one of them is wrong.** Quint is the spec; if TS "passes" by giving a different answer, the TS is wrong. We resolve by:
1. Reading the Quint spec carefully.
2. Making sure the TS implementation matches.
3. Adding to a "spec discrepancies" log.

This is what makes Quint worth its weight: it's an executable spec.

## 4. Test fixtures live in the data model

Subsystem tests need to construct valid game states. We don't fake the world — we use real factories.

### `src/test/factories/`

```
src/test/factories/
  index.ts
  game.ts          ← makeGame(): factory with builder methods
  player.ts        ← makePlayer({ race, traits, ... })
  planet.ts
  fleet.ts
  shipDesign.ts
  tech.ts
```

### Builder pattern for incremental setup

```ts
const game = makeGame()
  .withPlayers(3)
  .withGalaxy({ size: 'medium', seed: 42 })
  .withPlayer('p1', { race: 'Humans' })
  .withFleet('p1', { ships: [{ hull: 'Destroyer', count: 5 }] })
  .withPlanet({ owner: 'p1', population: 5_000_000 })
  .build();
```

The builder enforces invariants: you can't `.build()` a game state where two players control the same planet, or where a fleet references an unknown hull.

### Snapshots

For known-good states (e.g., "the canonical small galaxy used in combat tests"), we have `.qnt` snapshot files that get deserialized into TS:

```
specs/fixtures/smallGalaxy.json
```

The TS factory reads these via the persistence layer's deserializer (so persistence gets tested too).

## 5. Determinism and seeded RNG

Every domain function takes an `rng: () => number` (or a seed) parameter. Never call `Math.random()` inside domain code.

```ts
type Ctx = {
  rng: () => number;        // [0, 1)
  // future: maybe a clock for time-dependent rules
};

function resolveSpaceCombat(state, commands, ctx: Ctx) { ... }
```

In tests:

```ts
import seedrandom from 'seedrandom';
const rng = seedrandom('42');
const result = resolveSpaceCombat(state, commands, { rng });
```

In production, the turn manager passes a freshly-seeded RNG per turn (seed recorded in the save so replays work).

## 6. Property-based / fuzz testing

Quint's `simulate` and `repl` are essentially property-based testing: they generate random action sequences and run them. We use this to find invariant violations:

```
$ quint run specs/spaceCombat.qnt --invariant=shipsConserved \
    --max-samples=1000 --max-steps=50
```

Any counterexample is a bug. We add a regression test for it.

Vitest's `fast-check` integration is also valuable for property tests in TypeScript:

```ts
import fc from 'fast-check';

it('combat never creates ships', () => {
  fc.assert(fc.property(arbitraryCombatInput(), (input) => {
    const { state } = resolveSpaceCombat(input.state, input.commands, { rng: seedRng(0) });
    const totalAfter = sumShips(state.fleets);
    const totalBefore = sumShips(input.state.fleets);
    expect(totalAfter).toBeLessThanOrEqual(totalBefore);
  }));
});
```

## 7. Testing complex subsystems in isolation — worked example

**Goal**: test "when a fleet of 5 destroyers battles a fleet of 10 fighters at a star, the fighters always lose."

Without subsystem isolation, you'd have to:
1. Start a new game.
2. Build a destroyer (requires research, industry, turns).
3. Build fighters.
4. Position them at the same star.
5. End turn until they fight.
6. Observe outcome.

That takes minutes and depends on ~6 other subsystems.

**With subsystem isolation**, the test is:

```ts
it('5 destroyers beat 10 fighters', () => {
  const state = makeGame()
    .withFleet('att', { ships: [destroyer(5)] })
    .withFleet('def', { ships: [fighter(10)] })
    .withStar('origin', { occupants: ['att', 'def'] })
    .build();

  // Run 1000 simulated battles to make sure it's deterministic-ish
  for (let i = 0; i < 1000; i++) {
    const { state: result } = resolveSpaceCombat(state, {
      attacker: 'att',
      defender: 'def',
    }, { rng: seedRng(i) });

    expect(sumShips(result.fleets.def)).toBeLessThan(sumShips(state.fleets.def));
  }
});
```

Takes milliseconds. Doesn't depend on research, industry, fleet orders, AI, etc.

This is the **only** testing approach we'll use for the domain. The UI can later do end-to-end tests for player flow.

## 8. What "isolation" explicitly forbids in subsystem tests

A subsystem test **must not**:

- Import from `src/application/` (no store, no persistence).
- Import from `src/presentation/` (no React, no PixiJS).
- Reach into global state.
- Call `Date.now()`, `Math.random()`, or anything non-deterministic.
- Mock another subsystem. If subsystem A needs subsystem B's output, that's a dependency — declare it explicitly and test the integration instead.

A subsystem test **may**:

- Import factories from `src/test/factories/`.
- Import other pure helper modules (e.g., math utilities).
- Use Vitest's built-in matchers and `fast-check`.

## 9. CI pipeline

```
on push / on PR:
  1. Install deps
  2. quint typecheck specs/        ← Quint spec types valid
  3. quint test specs/             ← Quint tests pass
  4. quint simulate specs/ --invariant=...   ← invariants hold
  5. tsc --noEmit                  ← TS types valid
  6. vitest run                    ← TS tests pass
  7. (later) playwright test       ← E2E smoke
```

A failing step blocks merge.

## 10. Open testing questions

- **Snapshot tests**: useful but brittle for game states with hundreds of fields. Lean toward targeted assertions.
- **Coverage targets**: aim for 80%+ in `src/domain/`, but coverage isn't the goal — *invariant* coverage is. "Are the rules that matter all tested?" matters more than line %.
- **Performance benchmarks**: not yet. Add when combat with 30+ ships feels slow.
