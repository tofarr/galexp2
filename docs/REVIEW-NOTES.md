# Documentation Review — Open Questions & Inconsistencies

**Scope**: full review of `docs/PLANNING.md`, `docs/ARCHITECTURE.md`,
`docs/DECOMPOSITION.md`, `docs/TESTING.md`, `docs/ROADMAP.md`, and all 14
`docs/sections/D*.md` files. No Quint specs or TypeScript exist yet, so
all findings concern cross-document consistency, missing fields, and
unresolved design choices.

**Status snapshot** (from `PLANNING.md` and `ROADMAP.md`):

- D-layer decomposition (D1–D14): marked ✅.
- A-layer (A1–A5), P-layer (P1–P14), I-layer (I1–I4): pending.
- Phase 2 (Quint specs) and beyond: not started.

## Resolved in the 2026-06-05 docs-only PR (this PR)

The following findings from the initial review pass were resolved:

| # | Finding | Resolution |
|---|---|---|
| 1.1 | Hull record missing `troopCapacity` and `baseSupply` | Both fields added to `Hull` (D1.2, D7.1) with v1 values; D7.1 catalog pinned |
| 1.2 | Building record missing `cost` | `Building` gets `baseCost` and `buildIndustry` (D1.2) |
| 1.3 | Planet record missing `conqueredOnTurn` | Field added to `Planet` (D1.2); set by D10.5; consumed by D5.6 via `RECENTLY_CONQUERED_TURNS = 5` |
| 1.4 | TradeRoute missing `active` | Field added (D1.2); set to false by D10.5 on conquest |
| 1.5 | `BattleResolvedEvent` can't represent draws | Shape now `{star, winner: Option<FleetId>, loser: Option<FleetId>, outcome: Won \| Drawn, turn}` (D1, D9.6.5, D14.5) |
| 1.6 | D14.4 contradicts itself on tie-breaking | Resolved: tie at `MAX_TURN` is a draw; the "ties broken by turn of last score update" wording is superseded (D14.4 body + resolved decisions updated) |
| 1.7 | Three competing personality taxonomies | Unified: D3.4's `Race.aiPersonalityHints: Set<AiPersonalityTrait>` (7 traits) is a soft default mapped to D13.1's `StrategyKind` (5 kinds) via `aiDefaultStrategy(race)`; human player can override at game start. D11.3's offer-evaluation table is keyed by the same `StrategyKind`. |
| 1.8 | Council vote formula uses endgame score | Accepted: both D11.5 and D14.4 read the cached `state.score[player]`. v2 may separate them. |
| 1.9 | D11.1 drift vs peace-heal contradiction | Single rule: drift toward 0 (subsumes peace-heal); documented in D11.1 |
| 1.10 | `sharedEnemyBonus` in formula but v1 simple says no | Removed from formula; rationale recorded (O(N²) per turn is wasteful) |
| 1.11 | D14 score ownership ambiguous | Resolved: D14.4 owns the formula (pure `computeScore`); D4 helper owns the side-effect of writing `state.score[player]` once per turn |
| 2.1 | Modifier ADT gaps (`pillageModifier`, `researchEfficiency`) | Added `Pillage(float)` and `ResearchEfficiency(float)` variants to D3.2 `Modifier` ADT; trait modifier table extended to cover every trait in D3.1 |
| 2.2 | Ship record's role in troop formula | D10.1 now reads `shipDesign(state, ship).hull.troopCapacity` |
| 2.3 | Relation stored vs derived | D1.3 clarified: `relations` is derived, rebuilt by D11.1 each turn |
| 2.4 | Trade-route income formula wrong argument | `techLevelBonus(player, TechTree.Computer)`; `baseTrade = 2` pinned |
| 2.5 | `Ship.experience` unused in v1 | Documented as v2-placeholder; D8.5 merge still uses `max` (no-op in v1) |
| 2.6 | Constants missing from D1.1 | Pinned: `COMBAT_MAX_ROUNDS = 50`, `MAX_TURN = 200`, galaxy sizes, `SPY_UPKEEP_PER_TURN = 1`, `PLANET_BASE_TAX_INCOME = 1`, `CONQUEST_VICTORY_THRESHOLD = 0.80`, `RECENTLY_CONQUERED_TURNS = 5` |
| 2.7 | "derived" fields stored on `GameState` | D1.3 clarified ownership for `relations`, `events`, `score` |
| 2.8 | Event vs CombatEvent duality | D1 adds a section "Event vs CombatEvent relationship" defining how the two coexist |
| 3.1 | D8.5 split-by-ship-indices ambiguity | D8.5 now documents that `shipIndices` are **designId slot indices** (one per `ships: List<{designId, count}>` entry); partial-count splits are v2 |
| 3.2 | D5.7 queue depth contradiction | D5.7 rewritten: single-item queue; on completion `queueItem = None`; player must issue a fresh `SetQueue` to produce more |
| 3.3 | D9.2 targeting vs v1 simple | D9.2 chunk descriptions now mark each as "v1 simplification vs full MoO behavior" |
| 3.4 | D9.5.1 retreat references boarding (v2) | D9.5 retreat eligibility text rewritten without v2 references |
| 3.5 | Research-agreement transfer chunk unclear | D5.5 owns the inline implementation; D11.2's Treaty description references it; PLANNING.md decision log updated |
| 3.6 | Spy upkeep cost undefined | Pinned at `SPY_UPKEEP_PER_TURN = 1` (D1.1); D5.5 reads it |
| 3.7 | P-section numbering inconsistent | P13 (AI Inspector) and P14 (Hall of Fame) added to `DECOMPOSITION.md`; D12, D13, D14 references updated |
| 3.8 | D8 ETA contradiction | "v1 simple" line in D8 cleaned up; the rule is now consistent across D8.3, D8.5, and the resolved-decisions summary |
| 3.9 | `DeclareWar` in Offer ADT | Removed from `Offer`; introduced `DeclareWarCommand(from, to)`; D11.4 processes it as a non-acceptance command |
| 3.10 | `PlanetaryShield` not a `BuildingKind` | Replaced with `Defense` (which is in D1.2's `BuildingKind` enum) |
| 3.11 | D11.5 detection phrasing | Phrasing made consistent with D14.3's checkDiplomaticVictory |
| 4.1 | `COMBAT_MAX_ROUNDS` value | Resolved: `50` (D1.1) |
| 4.12 | Hull stat numbers | Resolved: D7.1 table pinned v1 values for all 10 hulls |
| 4.14 | `player.researchEfficiency` source | Resolved: D3.2 `Modifier.ResearchEfficiency(float)`; D6.3 reads it via `researchEfficiencyModifier(race)` |
| 4.15 | Buildings for v1 | Resolved: 7 buildings (`Factory, ResearchLab, Farm, Market, Defense, Spaceport, Capital`) |
| 4.16 | `PLANNING.md` open-questions pointer | Updated: now points at the per-section Open questions sections + this REVIEW-NOTES.md |

All of the above are also recorded in the PLANNING.md decision log
(2026-06-05 entries).

---

## Still open (deferred to Phase 2 or follow-up review)

The following findings remain and should be addressed either in
Phase 2 (Quint specs) or in a follow-up docs review:

### Section 4 (open questions not yet resolved)

- **4.2 Per-pair starting relations** — what relation value do players
  start with vs each other in a new game? Currently implicit; MoO
  defaults to "neutral" (0). Decide and pin.
- **4.3 Trait list scope and names** — D3.1 uses 16 traits (Skilled,
  Versatile, …); D3.2's table now covers them. The exact final list of
  ~16 traits still needs to be pinned at the spec phase.
- **4.4 Tech victory key techs** — D14 says "top of each tree (6
  techs)". Final list of those 6 techs to be pinned at spec phase.
- **4.5 Score formula** — weights are placeholders. Tune during
  playtesting.
- **4.6 Trade distance penalty** — `distancePenalty` formula is
  sketched ("-1% per parsec, capped at -50%") but the implementation
  (parsec distance function, what counts as "parsec") is unspecified.
- **4.7 Council detection on turn 0** — D11.5 fires on `turn % 25 == 0`,
  but the spec needs to confirm the council runs on turn 0 (before
  any player has meaningful stats) or skips to turn 25.
- **4.8 Subjugation edge cases** — what happens if a vassal is already
  at war with the suzerain's enemy when subjugation begins? Spec
  needs an explicit rule.
- **4.9 "Forced" peace treaty detection** — D14.4 lists "forced
  peace treaty" as a condition for the diplomatic victory; the rule
  for when a treaty is "forced" (e.g., broken under duress) is not
  specified.
- **4.10 Retreat destination algorithm** — D9.5.4 now describes the
  v1 algorithm ("nearest star within warpRange, ties broken randomly
  via `ctx.rng`") but the exact warp range usage and "nearest" metric
  are open.
- **4.11 Spy production cadence** — `SPY_PRODUCTION_INTERVAL = 10` is
  pinned but the formula for spy production itself (cost, prereqs,
  limits) is still loose.
- **4.13 Weapon catalog details** — D7.2 lists the *shape* but the
  full v1 weapon catalog is not yet pinned.
- **4.17 AI cooperation on captured planets** — D11.5 cites "AI
  cooperation on captured planets" as a future rule; not specified
  for v1.
- **4.18 AI "value" heuristic specifics** — D13.1 mentions "value"
  functions in several places; the actual implementation of these
  heuristics is not specified.
- **4.19 `BC_PER_TAX_POINT` scale vs other income** — pinned at 1 but
  the total income scale vs maintenance cost isn't validated yet.
- **4.20 `BuildingEffect.Maintenance` variant** — D5.5 filters for
  "Maintenance" variants but the `BuildingEffect` ADT doesn't include
  one. v1 ships sum=0; D5.5 has been clarified to note this.

### Section 5 (architectural / methodological)

- **5.1 A-layer and P-layer not yet decomposed** — Phase 2 will start
  the A-layer decomposition.
- **5.2 Quint spec strategy for D4 unresolved** — how to modularize
  `step` across many Quint files is a spec-phase decision.
- **5.3 A4 Turn Manager: which AI runs when, exactly** — order of AI
  vs human player turns needs to be pinned.
- **5.4 A2 persistence: schema versioning shape** — JSON shape and
  migration table; needed at spec phase.
- **5.5 Architecture says no `Math.random()` but several places still
  seed randomness by external time** — verify all randomness sources
  during Phase 2 Quint spec writing.
- **5.6 No specification for "observer" players / spectators** —
  mentioned in passing; not in scope for v1.
- **5.7 Save migration placement** — D4.2 says `initTurn` calls
  `migrateIfNeeded` every turn; pin whether this is idempotent
  (no-op after migration) or re-checks every turn.
- **5.8 "Pause-on-detection" deferred — but to where?** — D12.5
  mentions a pause-on-detection rule; not assigned to a chunk.

### Section 6 (minor wording / cross-reference issues)

- **6.1 `rng` parameter shape varies across docs** — some call sites
  pass `rng: () => number`, others `ctx: { rng, … }`. Pin one shape
  during Quint spec writing.
- **6.2 D9 phase 7b numbering** — D9 references "phase 7b" (retreat)
  in a way that's inconsistent with the D9 chunk list. Renumber or
  clean up.
- **6.3 `DECOMPOSITION.md` vs section docs: chunk counts differ** —
  e.g., D9 has 18 chunks; DECOMPOSITION.md D9 has 16. Reconcile.
- **6.4 D1.3 says relations "rebuilt each turn by the economy/
  diplomacy phase"** — phrasing updated to "rebuilt each turn by
  D11.1 (after drift, after treaty updates)" but the wording in D4
  still uses the old phrase. Update D4.
- **6.5 `FleetEvent` defined twice** — D9 and D10 both define a
  `FleetEvent` type with overlapping fields. Consolidate or rename
  one.
- **6.6 Save versioning: `initTurn` calls `migrateIfNeeded` every
  turn** — see 5.7.
- **6.7 Inconsistent dependency on Player's `treasury` and
  `totalProduction`** — D5 line 240 lists `Player.totalProduction` as
  a D14 dependency, but D14.4's score formula doesn't include
  production. Either remove the dependency or add production to the
  score formula.

---

## Round 3 — additional findings (2026-06-05)

A third docs-review pass over the full D-layer surfaced the findings
below. They were not present in the resolved list above nor in the
"Still open" 4.x/5.x/6.x sections, and they should be addressed in a
follow-up docs review PR or during Phase 2 spec work.

### Section 7 (critical)

- **7.1 D5 orchestrator order contradicts D5.3's morale dependency**
  — D5.2+D5.3+D5.4 run before D5.6 (morale), but D5.3's formula scales
  industry by `morale/100`. The first pass silently uses the previous
  turn's morale. Reorder (`D5.6` before `D5.2/3/4`) or document the
  one-turn lag. **Must be resolved before D5 spec work.**

### Section 8 (high — missing fields/types referenced but not defined)

- **8.1 `Player.aiMemory`** — referenced by D13 (`docs/sections/D13-ai-decisions.md:277`
  says "stored on `Player.aiMemory`"), not declared on D1.2's `Player`.
  Add the field or move ownership to `GameState` keyed by `PlayerId`.
- **8.2 `Player.intelLedger`** — referenced by D12 (`Player.intelLedger:
  Map<PlayerId, PlayerIntel>`, `docs/sections/D12-espionage.md:132`) and
  used in D13.1's `AIInput`. Not on D1.2's `Player`. Add the field.
- **8.3 `Player.totalProduction`** — referenced by D5 (`docs/sections/D5-economy.md:258, 263`)
  as a D14 / P12 dependency, not on D1.2's `Player`, not in D14.4's
  formula. (Also captured in 6.7.) Either drop the dependency or add
  production to the score formula.
- **8.4 `Spy` record defined twice with mismatched fields** — D1.2 has
  `{id, ownerId, locationPlanetId, mission, detectionRisk}`; D12.1 has
  `{id, owner, skill, status}`. Consolidate into D1.2 and add the
  missing fields, or have D12 reference D1.2's shape.
- **8.5 `Mission` vs `SpyMission` / `MissionType` duplication** — D1.2
  says `Mission — kind, targetPlayerId, startedOnTurn`. D12.1 says
  `MissionType` is a richer sum type and `SpyMission = {id, spy, type,
  assignedTurn, progressTurns, difficulty}`. Pick one.
- **8.6 `TURN_WARNING_THRESHOLD` constant** — referenced by D4.2 with a
  placeholder "e.g., 10". Not in D1.1's constants list. Pin.
- **8.7 `TAX_RATE_MIN` / `TAX_RATE_MAX` / `TAX_RATE_DEFAULT` constants**
  — D5.5 says `taxRate ∈ [0, 100]`, default 30; D5.6 reads `if
  taxRate > 60`. None are in D1.1.
- **8.8 `homeworldBonus()` function** — used in D5.6
  (`docs/sections/D5-economy.md:173`) but not declared anywhere.
  Likely a one-liner (`planet.id == player.homeworldPlanetId ? +10 : 0`)
  in D1.2 or a small helper in D5.
- **8.9 `Building` has no `level` field but cost formula uses one** —
  D5.7's `~10 bc × level` cost formula has no operand. Either add
  `level: int` to D1.2's `Building` and a `levelMultiplier(level)`
  function, or pin the cost as a flat value per `BuildingKind`.
- **8.10 `Ship` semantic ambiguity** — D1.2 has `count` and
  `hp (current/max)`; D8.5 split uses stack semantics (`ships:
  List<{designId, count}>`); D9.3.2 talks about "remaining-hp-fraction
  per ship". Either make `Ship` per-individual (no `count`) or
  per-stack (and pin HP semantics at the stack level). Foundational for
  D7.5, D8.5, D9, D10.
- **8.11 `CombatEvent` vs `Event` overlap not reconciled** — D1.2
  declares `CombatEvent` as a separate type but D1's event-kind index
  lists `CombatHit/CombatMiss/CombatCrit/ShipDestroyed/Retreat` as
  `Event` variants. Pick one or document the layering clearly.
- **8.12 `Hull.Fighter` / `Weapon.Fighter` naming overlap** — D7.1
  lists `Fighter`/`Bomber` as hulls; D7.2 lists the same names as
  `WeaponKind` values. Standard 4X distinguishes hull from fighter-bay.
  Rename one (e.g., `Weapon.FighterBay`) or document.
- **8.13 D3.2 trait table gaps: `Tolerant` missing, `HyperExpansion`
  is dead code** — D3.1 lists 16 unique traits; D3.2's `traitModifiers`
  table covers 15 but omits `Tolerant`. D3.2 defines `HyperExpansion`
  but no race in D3.1 has it.
- **8.14 `Subterranean` "no surrender until fully eliminated" not
  expressed in D3.2** — D10 (`docs/sections/D10-ground-combat.md:272`)
  claims this is driven by `GroundDefense(1)` from D3.2's `Modifier`
  ADT, but D3.2's `traitModifiers(Subterranean) = { Morale(1) }` has no
  `GroundDefense(1)` and no surrender-suppression variant. Either
  update D3.2 or document D10.3's hard-coded special case.
- **8.15 `MaxColonies` and `ColonizationSpeed` modifiers defined but
  unused** — D3.2 declares them but no section reads them (D5.1,
  D3.5, D13). Remove from the ADT or declare the consumer.

### Section 9 (medium — open decisions)

- **9.1 Per-pair starting relations (4.2 — still open)** — the 10×10
  matrix is deferred to spec phase. D11.1 Quint spec can't be
  testable without it.
- **9.2 Council vote uses `state.score` which includes treasury** —
  D14's resolved decision (`docs/sections/D14-victory.md:220`)
  acknowledges this is "acceptable for v1" but invites v2 separation.
  Worth modeling deliberately or flagging as intentional.
- **9.3 AI scoring heuristics (4.18 — still open)** — `value =
  immediate_benefit × race_modifier` with no values pinned.
- **9.4 `D9.7` referenced but absent from DECOMPOSITION.md** — D9
  reserves `D9.7` for boarding (D9-space-combat.md:66, 167, 178) but
  DECOMPOSITION.md lists only D9.1–D9.6. Either add the v2 placeholder
  to DECOMPOSITION.md or note in D9 that it's explicitly out of scope.
- **9.5 `SpyDetection` modifier semantics ambiguous** — D3.2 says
  "multiplier on detection chance (passive — 'easier to spot others'
  spies')" but the `Stealthy` example says "passive: harder to detect
  own spies" — opposite effect on the same axis. D12.5's detection
  formula doesn't read `SpyDetection` at all, so the trait has no
  effect under the current formula.
- **9.6 `AIPipeline` event consumer unspecified** — listed in D1's
  event-kind index; emitted by D13.7; no consumer description (P13
  reads `AIOutput.reasoning` directly).
- **9.7 Trade-route distance — undefined "parsec" / planet pair
  function** — D11.6's `-1% per parsec` penalty is unspecified; planets
  don't have coordinates, only stars do. Pin the distance function.
  (Also captured in 4.6.)
- **9.8 `CombatStats.speed` is purely cosmetic in v1** — D9.1.2 uses
  computer + hull size for initiative (not `CombatStats.speed`); D8
  says `Hull.baseSpeed` is "for fleet speed display; not used in v1
  movement". Document explicitly.
- **9.9 REVIEW-NOTES 6.5 stale: "FleetEvent defined twice"** —
  couldn't reproduce; `FleetEvent` is defined only in D8.4. The
  actual residual issue: D8.4's `FleetEvent` ADT is not in D1's
  event-kind index. The 6.5 finding as written is stale; the
  underlying reconciliation task is real.

### Section 10 (low — wording / cross-references / cosmetics)

- **10.1 `Player.totalFleetStrength` "computed view function" not
  declared** — D1.2 mentions it as a view function but no such
  function exists. Multiple sections read "fleet strength"; pin one
  location (D7.5 is the natural home).
- **10.2 Event payload naming inconsistency** —
  `StarvationEvent {planet, lostPopulation}` (D5.1) vs
  `PillageEvent {planet, populationLost, buildingsDamaged}` (D10.4).
  Pick one word order.
- **10.3 Event field suffix inconsistency** — D8.4's `FleetEvent`
  uses `sideA: FleetId`; D1.2's `CombatEvent` uses `sideAId`. Standardize.
- **10.4 `player.researchAccumulated` referenced by D6.3 but missing
  from D1.2** — `docs/sections/D6-research.md:96-105` reads it; D1.2
  `Player` doesn't list it. Add the field.
- **10.5 `Player.score` ownership described three different ways** —
  D1.2 says "score (derived; see D14.4)", D1.3 says
  `score: PlayerId -> int` on `GameState`, D14.4 says "`Player.score`
  is a derived field stored on `GameState`". Pick one phrasing.
- **10.6 D13.1 strategy tie-breaking when hints are empty or
  contradictory** — `aiDefaultStrategy(race)` priority list
  (`docs/sections/D13-ai-decisions.md:80-84`) — what about
  contradictory hints (e.g., race has both `Aggressive` and
  `Diplomat`)? Document precedence.
- **10.7 `BuildingKind.cost(kind)` static-method call doesn't match
  the model** — D5.7 calls it; D1.2 has no such function. Should be
  `state.buildings[kind].baseCost`.
- **10.8 Mission durations scattered through prose, no table** —
  StealTech is 3 turns; other missions are 1 turn. No
  `duration(missionType)` table in D1.1.
- **10.9 REVIEW-NOTES 6.3 stale: D9 chunk count is actually 26
  sub-chunks** — D9 has 3+4+5+5+4+5 = 26 sub-chunks, not 18. The
  underlying reconciliation task is real but the number is stale.
- **10.10 `player.shipStatBonuses` referenced by D6.4 but missing
  from D1.2** — `docs/sections/D6-research.md:138` reads it; D1.2
  `Player` doesn't list it. Add the field.
- **10.11 D11.2 vs D5.5 phrasing inconsistency for research-agreement
  split** — "unspent research" vs "after tech-acquisition deduction".
  Pick one.
- **10.12 Colonization loop not defined (only homeworld assignment)** —
  D3.5 handles the *initial* homeworld at game start. No section
  describes in-game colonization: no `ColonizePlanet` event, no
  colonization command, no `Planet.isColonized` flag, no
  `MoveTo(uninhabitedPlanet)` flow.
- **10.13 Events log trimming + persisted events re-merge path is
  sketched, not specified** — D1.3 trims to last 200; D14.5 reads
  persisted events for stats. The flush/load path is A2's job but no
  section in scope describes it.

---

## How to use this document

When you make changes that resolve a finding above:

1. Remove the finding from the "Still open" list (or move it to
   "Resolved").
2. Add a 2026-06-05 (or later) entry to the PLANNING.md decision log
   describing the resolution.

The next docs review pass should start by re-reading this file's
"Still open" section.