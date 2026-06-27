# Decomposition — Game Sections & Chunks

This document holds the top-level decomposition of the game. Each section gets:
- A one-line **contract** (what it owns, what it doesn't).
- A list of **chunks** that decompose it.
- Links to a per-section doc in `docs/sections/` once we decompose it.

A *chunk* is a leaf when it can be:
- Specified in Quint in 50–300 lines.
- Implemented in TypeScript in 100–500 lines.
- Tested in isolation (no flaky cross-chunk dependencies).

We recursively decompose sections until every chunk meets that bar.

---

## Layer 0 — Domain (pure rules, Quint-spec'd)

### Section D1 — Core Types & Constants

**Contract**: defines all data types shared across the domain (`Player`, `Race`, `Planet`, `Star`, `Fleet`, `Ship`, `Tech`, `Weapon`, `Treaty`, `Spy`, `Mission`, etc.) and game-wide constants (max players, galaxy sizes, etc.). Owns nothing but data; no behavior.

**Chunks**:
- D1.1 Identifiers & enums (`PlayerId`, `StarId`, etc.) — primitive types and ID newtypes.
- D1.2 Race definition type — name, traits, AI personality hints.
- D1.3 Planet definition type — size, richness, type, population, buildings.
- D1.4 Star definition type — spectral class, planets owned.
- D1.5 Fleet & Ship types — fleet composition, ship design slots.
- D1.6 Tech & Weapon types — tech ID, effects, prereqs; weapon stats.
- D1.7 Treaty & Diplomacy types — relation level, treaty kinds, trade routes.
- D1.8 Spy & Mission types — spy assignment, mission kinds.
- D1.9 Game-wide constants — galaxy sizes, max players, turn budget.
- D1.10 Root `GameState` type — the single big record holding everything.

*Decomposition target*: D1 is mostly mechanical; if the types fit in ~300 lines combined, we treat D1.1–D1.10 as one chunk.

Per-section doc: [`sections/D1-core-types.md`](sections/D1-core-types.md). Consolidated into three Quint-spec-sized chunks: D1.1 primitives, D1.2 entities, D1.3 root GameState.

---

### Section D2 — Galaxy Generation

**Contract**: produces a deterministic `Galaxy` from `(seed, options)`. Does not place empires — that's D4/D3's responsibility (start position selection).

**Chunks**:
- D2.1 Star field generation — placement of N stars on a 2D map.
- D2.2 Star spectral classification — pick class (O/B/A/F/G/K/M) with realistic distribution.
- D2.3 Planet generation per star — number of planets, types, sizes.
- D2.4 Specials — pick which stars get specials (e.g., "Ancient", "Pirate Cache").
- D2.5 Start position selection — pick one good homeworld per race.
- D2.6 Galaxy options & validation — galaxy size, star count, race count constraints.

Per-section doc: [`sections/D2-galaxy-generation.md`](sections/D2-galaxy-generation.md). Splits into 5 Quint files: `galaxyOptions.qnt`, `starField.qnt`, `planetGen.qnt`, `specials.qnt`, `candidates.qnt`, plus a top-level `galaxyGen.qnt` orchestrator. D2.5 reframed as candidate-homeworld selection (final assignment is D3.5).

---

### Section D3 — Races & Traits

**Contract**: defines the 10 races and how their traits modify game rules. Pure data + small modifier functions.

**Chunks**:
- D3.1 Race catalog — the 10 races, base stats, portraits (asset ref).
- D3.2 Trait modifier system — how a `Race.traits[]` array changes formulas elsewhere.
- D3.3 Race-specific tech affinities — e.g., Meklons get +1 to Computers.
- D3.4 Diplomacy disposition — race-to-race starting relations and proclivities.

Per-section doc: [`sections/D3-races-and-traits.md`](sections/D3-races-and-traits.md). Adds **D3.5 homeworld assignment** (receives candidate list from D2.6). Four Quint files: `raceCatalog.qnt` (D3.1+D3.3+D3.4), `traits.qnt` (D3.2 modifier system), `homeworld.qnt` (D3.5), `races.qnt` (public API).

---

### Section D4 — Turn Cycle (the master `step`)

**Contract**: the master orchestrator's domain representation. Given `(state, playerCommands, aiCommands) -> newState`. Defines the order of phases and what each phase produces.

**Chunks**:
- D4.1 Phase ordering — init turn → economy → research → production → fleet movement → contact resolution → combat resolution → diplomacy → espionage resolution → end-of-turn cleanup.
- D4.2 Turn boundary — turn counter increment, cleanup of per-turn flags.
- D4.3 Event emission contract — every phase emits events into a log.

This is small but central. The real work happens in D5–D12; D4 just calls them in order.

Per-section doc: [`sections/D4-turn-cycle.md`](sections/D4-turn-cycle.md). One Quint file (`turnCycle.qnt`) since D4 is intentionally small. Phase order locked in: init → economy → research → production → fleetMovement → contactResolution → combatResolution → espionage → diplomacy → victoryCheck → endTurn. Espionage runs before diplomacy so spy results can inform this turn's offers. Victory check before end-of-turn cleanup so a win ends mid-step.

---

### Section D5 — Economy

**Contract**: resolves the economy phase. Computes food, production, research, income, morale, population growth for each colony.

**Chunks**:
- D5.1 Population growth & starvation.
- D5.2 Food production & surplus.
- D5.3 Industry production.
- D5.4 Research generation (before tax slider).
- D5.5 Income & maintenance costs.
- D5.6 Morale calculation (race traits, buildings, wars).
- D5.7 Building & ship production queues (consumes industry, emits new ships/buildings).

Per-section doc: [`sections/D5-economy.md`](sections/D5-economy.md). Seven Quint files (one per chunk, with D5.2 + D5.3 sharing `foodAndIndustry.qnt`): population, foodAndIndustry, research, income, morale, queues, plus a top-level economy orchestrator. Linear population growth, single-item queue per planet, one tax slider, debt-as-morale-penalty, Lithovore ignores food. v1 emits RevoltRiskEvent but doesn't actually flip ownership.

---

### Section D6 — Research & Tech Tree

**Contract**: defines the tech graph (5–6 trees), computes tech costs, applies research, and emits tech-acquired events.

**Chunks**:
- D6.1 Tech graph definition — nodes (techs) and edges (prereqs).
- D6.2 Tech cost calculation (level × base cost, with race modifiers).
- D6.3 Research selection & accumulation — what each empire is researching.
- D6.4 Tech acquisition events — when a tech completes, who learns it, who can benefit from trade.
- D6.5 Tech trade & gifts — pre-requisite for diplomacy.

---

### Section D7 — Ship Design & Combat Stats

**Contract**: given a ship design (hull + slots + weapons + specials), compute the combat stats used by D9.

**Chunks**:
- D7.1 Hull catalog — hull sizes, slot layouts, base stats.
- D7.2 Weapon catalog — beam, missile, torpedo, fighter, bomber with damage/range/shots.
- D7.3 Specials catalog — shields, armor, ECM, etc.
- D7.4 Design validation — rules of placement (e.g., small weapons can't go on huge hulls).
- D7.5 Stat computation — attack/defense/computer/speed/hp from components.

Per-section doc: [`sections/D7-ship-design.md`](sections/D7-ship-design.md). Five Quint files: `hulls.qnt`, `weapons.qnt`, `specials.qnt`, `designValidation.qnt`, `combatStats.qnt`. D7.4 is structural-only; tech-prereq enforcement lives in D5/P6. D7.5 is pure per-design; per-instance modifiers (race traits, owner tech) are applied at the consumer.

---

### Section D8 — Fleet & Movement

**Contract**: given movement orders and the galaxy state, advance fleets and resolve arrivals.

**Chunks**:
- D8.1 Fleet orders — destination, stance (defend, attack, explore).
- D8.2 Pathfinding (warp lanes) — given galaxy topology, compute routes.
- D8.3 Fuel/warp range limits.
- D8.4 Arrival & encounter detection — when a fleet arrives at a star, detect contact.
- D8.5 Fleet merge & split.

Per-section doc: [`sections/D8-fleet-movement.md`](sections/D8-fleet-movement.md). Four Quint files: `orders.qnt`, `movement.qnt`, `fleetOps.qnt`, `arrival.qnt`. D8.2 reframed as "distance & range" (Euclidean warp in v1; warp lanes deferred to v2). D8.4 enforces "one fleet per player-side at each star" before emitting Encounter events to D9 — D9 only ever sees pairwise encounters.

---

### Section D9 — Space Combat Resolution

**Contract**: given two (or more) fleets at a star, resolve the tactical space battle. Produces a new state for both fleets (survivors, retreat positions) and a stream of combat events.

**Chunks**:
- D9.1 Combat setup — initiative ordering, stack positioning.
- D9.2 Targeting AI — who shoots whom per round.
- D9.3 To-hit & damage calculation — formulas.
- D9.4 Specials in combat — shields, ECM, point defense, fighter screens.
- D9.5 Retreat mechanics.
- D9.6 Combat outcome — survivors, XP awarded, scrap.

This section is large enough to warrant its own detailed decomposition doc — see `docs/sections/D9-space-combat.md` (when written).

---

### Section D10 — Ground Combat Resolution

**Contract**: given an invading fleet and a defended colony, resolve the ground battle. Produces new population, buildings, and ground combat events.

**Chunks**:
- D10.1 Troop strength calculation — invasions, garrison, native populations.
- D10.2 Combat rounds — attacker/defender rolls, casualties.
- D10.3 Morale & surrender.
- D10.4 Pillage & collateral damage.
- D10.5 Outcome — conquest, repelled, treaty forced.

---

### Section D11 — Diplomacy

**Contract**: state machine for diplomacy. Players and AI submit offers (trade, tech, treaties, war); the resolver produces new treaties and relation changes.

**Chunks**:
- D11.1 Relation state — per-pair relation level (-100..+100) and modifiers.
- D11.2 Treaty kinds — peace, alliance, trade pact, NAP, etc.
- D11.3 Offer evaluation — does this offer appeal given the AI's personality and current relations?
- D11.4 Treaty acceptance/rejection.
- D11.5 Council / Galactic Emperor — vote accumulation, election.
- D11.6 Trade — ongoing per-turn trade income.

---

### Section D12 — Espionage

**Contract**: given active spy missions, resolve them. Produces intel events, sabotage, counter-espionage.

**Chunks**:
- D12.1 Spy mission assignment.
- D12.2 Mission resolution — success check, detection check.
- D12.3 Intel types — tech stolen, fleet spotted, treasury observed.
- D12.4 Sabotage effects — building destroyed, ship damaged, etc.
- D12.5 Counter-espionage — defending against enemy spies.

---

### Section D13 — AI Decision Logic

**Contract**: given an AI's `state` and its personality, produce a `Command[]` for the next turn. Each AI strategy is a separate chunk.

**Chunks**:
- D13.1 Common AI scaffolding — input/output types, personality traits.
- D13.2 Strategy: Aggressive Conqueror — prioritizes military, picks fights with weakest neighbors.
- D13.3 Strategy: Builder — colonizes widely, develops economy, late-game military.
- D13.4 Strategy: Technologist — rushes research, trades for tech.
- D13.5 Strategy: Diplomat — pursues alliances and council victory.
- D13.6 Strategy: Balanced — weighted blend of above.
- D13.7 Per-turn AI evaluation pipeline — when each AI runs, in what order.

---

### Section D14 — Victory Conditions

**Contract**: after each turn, check whether any player has won. If so, end the game.

**Chunks**:
- D14.1 Conquest victory — owns X% of galaxy, last empire standing.
- D14.2 Technological victory — researched X key techs.
- D14.3 Diplomatic victory — elected by council.
- D14.4 Score / time victory — at turn N, highest score wins.
- D14.5 End-game state — rankings, final stats, post-game screen.

---

## Layer 1 — Application (state, persistence, orchestration)

### Section A1 — Game State Store

**Contract**: wraps `GameState` (from D1.10) in a Zustand store with typed selectors and actions.

**Chunks**:
- A1.1 Store schema & selectors.
- A1.2 Command dispatch — typed actions.
- A1.3 Subsystem result application — how `resolve` results update store.

### Section A2 — Persistence

**Contract**: save/load game state to IndexedDB via Dexie. Schema versioning with migrations.

**Chunks**:
- A2.1 Dexie schema definition.
- A2.2 Save serializer — `GameState -> Blob`.
- A2.3 Load deserializer — `Blob -> GameState` with version check.
- A2.4 Migration framework — `(oldState, oldVersion) -> newState`.
- A2.5 Auto-save triggers.

### Section A3 — Input Commands

**Contract**: typed command set the player (and AI) issue. All commands are pure data — no behavior.

**Chunks**:
- A3.1 Command catalog — every legal player action as a discriminated union.
- A3.2 Command validation — domain rules check before enqueue.
- A3.3 Command queue per turn — collection of commands awaiting turn end.

### Section A4 — Turn Manager

**Contract**: the orchestrator. Collects player commands, runs AI turns, calls domain subsystems in order, emits events, persists.

**Chunks**:
- A4.1 Turn phases — explicit calls to each domain subsystem.
- A4.2 Command batching.
- A4.3 Event log emission.
- A4.4 AI scheduling — which AI runs when.
- A4.5 Turn boundary cleanup.

### Section A5 — Event Log

**Contract**: append-only event stream from the domain. UI renders from it (combat reports, diplomacy messages, discoveries).

**Chunks**:
- A5.1 Event types (mirrors domain event emissions).
- A5.2 Event log store.
- A5.3 Event aggregation & filtering (e.g., "show me only diplomacy events").
- A5.4 Read/unread tracking.

---

## Layer 2 — Presentation (UI)

### Section P1 — Layout & Navigation

**Chunks**:
- P1.1 App shell & routing.
- P1.2 Keyboard shortcuts & hotkeys.
- P1.3 Modal & dialog primitives.
- P1.4 Notification ticker (reads event log).

### Section P2 — Splash / Main Menu / Setup

**Chunks**:
- P2.1 Splash & main menu.
- P2.2 New game — galaxy options.
- P2.3 Race selection.
- P2.4 Load game — list saves.

### Section P3 — Star Map Screen

**Chunks**:
- P3.1 PixiJS app setup & render loop.
- P3.2 Star rendering (sprite + label).
- P3.3 Fleet rendering (icon + count).
- P3.4 Pan/zoom (mouse + keyboard).
- P3.5 Selection & info popover.
- P3.6 Movement order entry (click destination).
- P3.7 Range indicator (how far can selected fleet move).
- P3.8 Minimap (later).

### Section P4 — Planet Screen

**Chunks**:
- P4.1 Planet list.
- P4.2 Colony view header (planet image, name, race).
- P4.3 Population & growth panel.
- P4.4 Tax / research / construction sliders.
- P4.5 Production queue.
- P4.6 Buildings list & assign.
- P4.7 Garrison & invasion indicator.

### Section P5 — Research Screen

**Chunks**:
- P5.1 Tech tree layout & rendering.
- P5.2 Tech detail panel.
- P5.3 Research selection.
- P5.4 Trade offers.

### Section P6 — Fleet & Ship Design Screen

**Chunks**:
- P6.1 Fleet list.
- P6.2 Fleet detail (composition, orders).
- P6.3 Ship designer — hull picker.
- P6.4 Ship designer — slot fillers.
- P6.5 Ship designer — special equipment.
- P6.6 Stats preview.
- P6.7 Save design.

### Section P7 — Diplomacy Screen

**Chunks**:
- P7.1 Race portrait & leader.
- P7.2 Relation indicator.
- P7.3 Chat / message log.
- P7.4 Offer panel — propose trade, treaty, tech.
- P7.5 Incoming offer response.
- P7.6 Council / vote panel.

### Section P8 — Espionage Screen

**Chunks**:
- P8.1 Spy list.
- P8.2 Mission assignment.
- P8.3 Mission report.

### Section P9 — Reports & Lists

**Chunks**:
- P9.1 Production summary.
- P9.2 Fleet list (galaxy-wide).
- P9.3 Intelligence report.
- P9.4 Score / rankings.
- P9.5 Graph: economy over time (later).

### Section P10 — Tactical Space Combat Screen

**Chunks**:
- P10.1 PixiJS combat stage.
- P10.2 Ship sprites & formation positioning.
- P10.3 Combat playback engine (reads combat events).
- P10.4 HUD — round number, fleet status.
- P10.5 Auto-resolve option.

### Section P11 — Ground Combat Screen

**Chunks**:
- P11.1 Sprite layout.
- P11.2 Round-by-round playback.
- P11.3 Outcome panel.

### Section P12 — End-of-Turn Summary / Galactic Council

**Chunks**:
- P12.1 Summary panel — events of the turn.
- P12.2 Next-turn button.
- P12.3 Council vote screen (when triggered).

---

## Layer 3 — Infrastructure

### Section I1 — Quint toolchain integration
### Section I2 — Test harness (Vitest, Quint, Playwright)
### Section I3 — Asset pipeline (sprites, portraits, audio)
### Section I4 — Build & deploy (Vite, static hosting)

These are scaffolding chunks, not gameplay. They get decomposed when we set up the toolchain.

---

## Recursive decomposition status

Each section above will eventually get its own `docs/sections/D<n>-<name>.md` with finer-grained chunk breakdown and Quint-spec-sized leaves. We start with **D9 — Space Combat** as the pilot (see "Why start with D9" below).

### Why start with D9 (Space Combat)

- It is the **most algorithmically complex** subsystem, so designing the spec/impl pattern here pays off everywhere.
- It is **most isolated** — the inputs (two fleets + their designs + RNG seed) are simple to construct, and the outputs are pure data.
- It is **most fun to playtest** — gives early feedback on whether the architecture feels right.
- It directly exercises the "experiment with richer dynamics later" path — v1 simple combat, v2 with shield facings etc.

Other natural second picks: D2 (Galaxy Gen — pure generator, easy to test), D5 (Economy — touches most other systems, so good integration test).
