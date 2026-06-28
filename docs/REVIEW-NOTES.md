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
| 4.17 | Round 4: `Toltec` orphan trait in D3.2 | Dropped; `Tolerant = { Diplomacy(1) }` is the canonical entry |
| 4.18 | Round 4: `Honorable` in D10 L135 comment | Replaced with `Honorbound +2` (matches D3.2) |
| 4.19 | Round 4: Psilons `Weapons +1 (extra)` typo | Clarified: Psilons have dual affinities (Weapons +2, Shields +1 per D3.3) |
| 4.20 | Round 4: `Relation` shape mismatch D1 vs D11 | Unified to the D11 record in D1.2; D11.1 references it |
| 4.21 | Round 4: `TreatyKind` enum mismatch D1 vs D11 | D1.1 now points at D11.2's authoritative ADT |
| 4.22 | Round 4: Redundant `Player.relations/spies/treaties` | Dropped from `Player`; storage location is `GameState` only |
| 4.23 | Round 4: `Player.aiPersonality` untyped | Typed as `Personality` (D13.1 record) |
| 4.24 | Round 4: `ShipDesign.buildQueue` never written | Dropped; queue is per-player (D5.7) |
| 4.25 | Round 4: Missing constants `STARTING_TECH_LEVEL`, `INITIAL_POPULATION` | Pinned: `STARTING_TECH_LEVEL = 1`, `INITIAL_POPULATION = 5` |
| 4.26 | Round 4: `JUST_MADE_PEACE_TURNS` undefined | Pinned: `JUST_MADE_PEACE_TURNS = 5` (D1.1); re-war rule documented |
| 4.27 | Round 4: `WarState` initial state | `WarState` ADT added to D1.1; new relations start at `AtPeace` |
| 4.28 | Round 4: `Command` ADT scattered across sections | Single ADT added to D1.2; A3 retains validation/queue only |
| 4.29 | Round 4: Stale `Tribute` enum value | Confirmed gone; "tribute" remains in noun sense (vassal maintenance) |
| 4.30 | Round 4: `aiPersonality(race)` references nonexistent helper | Renamed to `aiDefaultStrategy(race)` in D11/D13 dep tables |
| 4.31 | Round 4: `Alliance` trade-route enablement unclear | Pinned: TradePact ×1.20, Alliance ×1.20, NAP ×0.80; PeaceTreaty/Subjugation/ResearchAgreement do not enable trade |
| 4.32 | Round 4: `HyperExpansion` "fifth trait slot" comment | Reworded: "v1 ships without it. v2 could re-add it once a race's third trait slot exists." |
| 4.33 | Round 4: `CombatStats.cost` naming overlap | Inline comment clarifies: `// bc to build one ship of this design = …` |
| 4.34 | Round 4: D7 drive specials comment vague | Now lists `HyperDrive / FuelTank` as the v2 candidates |

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

## Round 4 — findings (2026-06-05, resolved in this PR)

A fourth docs-review pass, cross-checking every section's data-model
references against D1.2 (the canonical type catalogue) and D1.1
(constants), surfaced 21 findings. **All 21 resolved in the
2026-06-05 docs-only PR on this branch** (see "Resolved in the
2026-06-05 docs-only PR" at the top of this file for the
high-level summary; the per-item fix list is in PLANNING.md's
decision-log under the "Round 4" section). The findings are kept
here as the canonical audit trail — each item below records its
original diagnosis and the resolution applied.

### Critical (will block Quint spec writing if left as-is)

- **R4.1 `Trait.Tolerant` vs orphan `Trait.Toltec`** —
  `D1-core-types.md` L41 lists the v1 `Trait` enum as 16 entries and
  the Gnolams in `D3-races-and-traits.md` L46 carry `Tolerant`. The
  trait-modifier table in `D3-races-and-traits.md` L129 instead defines
  `traitModifiers(Toltec) = { Diplomacy(2) }`, and `Tolerant` is
  re-declared at L144 with a different modifier (`{ Diplomacy(1) }`,
  with a comment that says "was missing from prior drafts"). Both
  `Toltec` and `Tolerant` cannot coexist: `Toltec` is not in the v1
  catalog and no race carries it; `Tolerant` is the Gnolams' second
  trait. **Resolution**: drop `Toltec`, keep `Tolerant`. Update
  `D3-races-and-traits.md` L129 to `traitModifiers(Tolerant) = {
  Diplomacy(2) }` and decide whether the L144 entry is redundant
  (merge or delete).

- **R4.2 Stale "Honorable" comment in `D10-ground-combat.md` L135** —
  `// e.g., Honorable +1` refers to a trait that doesn't exist.
  `D1-core-types.md` L50 declares `Honorable` as an
  `AiPersonalityTrait` (the D3.4 hint taxonomy), not a `Trait`. The
  actual morale-bearing trait in D3.2 L141 is `Honorbound`. **Fix**:
  replace `Honorable` with `Honorbound` in the comment.

- **R4.3 `D3-races-and-traits.md` L51 — Psilons tech-affinity typo** —
  the Psilons row reads `Weapons +1 (extra)`. Every other race's
  `techAffinities` value is a single `TechTree` name (Construction,
  Computer, Propulsion, Weapons) or `none`. The `+1 (extra)` looks
  like a column-bleed (the affinity value is `+1`, and the "(extra)"
  is a footnote that wandered into the cell). **Fix**: row should be
  `Weapons` with the affinity mapping `+1` per turn, consistent with
  other races' numeric mapping (D3.3 sets the actual value).

- **R4.4 `Relation` shape inconsistency** — `D1-core-types.md` L86
  declares `Relation` as `{level: int (-100..+100), modifiers: list}`,
  while `D11-diplomacy.md` L41-49 declares it as `{playerA, playerB,
  score, treaties, tradeRoutes, warState, lastContactTurn}` with
  `score` (not `level`) and "modifiers are computed, not stored".
  These are two different record shapes for the same type. **Fix**:
  pick D11's shape (it includes the per-treaty/per-trade-route
  bookkeeping the diplomacy code actually reads) and rewrite D1 L86
  to match; drop `level` in favor of `score`.

- **R4.5 `TreatyKind` enum mismatch between D1 and D11** —
  `D1-core-types.md` L46 lists `{Peace, Alliance, TradePact, NAP,
  NonAggression, Tribute}` (6 variants, with "NAP" and
  "NonAggression" both listed and no `ResearchAgreement` or
  `Subjugation`). `D11-diplomacy.md` L97-103 lists
  `{NonAggressionPact, TradePact, Alliance, PeaceTreaty, Subjugation,
  ResearchAgreement}` (6 variants, different names). Several are
  synonyms: `NAP ≈ NonAggressionPact`, `Peace ≈ PeaceTreaty`,
  `Tribute ≈ Subjugation`. `ResearchAgreement` exists only in D11;
  `NonAggression` exists only in D1. **Fix**: D11's list is the
  authoritative one (it has the chunk-level code paths). Replace
  D1's enum with D11's, normalize `NAP`/`NonAggressionPact` to
  `NonAggressionPact`, and either drop the empty `terms` field on
  `Treaty` (L87) or document what goes in it (D11's variants have
  no parameter list).

- **R4.6 `Player.relations`, `Player.spies`, `Player.treaties`
  redundant storage** — `D1-core-types.md` L80 lists these as fields
  on `Player`, but `D1.3` (L187) and `D11.1` (L89) declare
  `relations` on `GameState` only (`(PlayerId, PlayerId) -> Relation`,
  rebuilt by D11.1 each turn). Likewise `spies` and `treaties` on
  `GameState` per L182-183. `Player.relations`, `Player.spies`,
  `Player.treaties` are either dead fields or duplicate the canonical
  copies. **Fix**: drop them from the Player record, or document them
  as "cached convenience views, kept in sync by D4" (and pin the
  invariant).

- **R4.7 `Player.aiPersonality` is untyped** — D1 L80 lists the field
  but doesn't type it. `D13-ai-decisions.md` L70-75 defines
  `Personality { strategy: StrategyKind, weights: WeightVector,
  aiMemory: AIMemory }`. **Fix**: type the field explicitly as
  `Personality` in D1 L80 and add a one-line note: "human player's
  `personality.strategy` is overridable at game start; AI players'
  `personality` is derived via `aiDefaultStrategy(race)` at game
  init and then mutated by D13 in-place across turns."

- **R4.8 `ShipDesign.buildQueue` on `ShipDesign`, not on Player** —
  D1 L85 declares `buildQueue` on `ShipDesign`, but D5.7's queue is
  per-player (`player.queueItem: Option<QueueItem>`), per
  `D5-economy.md` L189-220. A `buildQueue` on the ShipDesign record
  is never written by any chunk. **Fix**: remove the `buildQueue`
  field from `ShipDesign`, or document it as a v2 placeholder for
  per-design auto-build.

### High (should be resolved before Phase 2)

- **R4.9 `STARTING_TECH_LEVEL` and `INITIAL_POPULATION` not pinned** —
  `D1-core-types.md` L60 names both constants, but neither has a value.
  D5.1 (population growth) and D6.3 (tech acquisition) both reference
  initial values without stating them. **Fix**: pin values in D1.1
  (suggested: `STARTING_TECH_LEVEL = 1`, `INITIAL_POPULATION = 5`).

- **R4.10 `Relation.lastContactTurn` initial value unspecified** —
  D11.1's drift rule (L322) requires it, and the formula on L66-72
  uses it, but the field's initial value at game start isn't stated.
  Initialized to `state.turn` at pair-creation? To 0 (and then the
  first contact sets it)? Pin the rule.

- **R4.11 `Relation.warState: JustMadePeace(until: TurnId)` semantics
  undefined** — D11 L47 declares the variant but no chunk defines the
  window length (5 turns? 10?) or what the "until" turn enforces
  (no-attack cooldown? relation boost? both?). Pin the constant
  (`JUST_MADE_PEACE_TURNS = N`) and the rules.

- **R4.12 `WarState.AtPeace` initialization never pinned** — D11 L47
  declares `AtPeace | AtWar(since: TurnId) | JustMadePeace(until: TurnId)`
  but the default for a fresh `Relation` record is unspecified. The
  open question 4.2 ("per-pair starting relations") partly depends on
  this; pin the default (suggested: `AtPeace` with `score = 0`).

- **R4.13 `Command` ADT scattered across D-sections without a single
  source of truth** — D1 L288 says `Command` is "defined in A3" (the
  A-layer). D4 L72-80, D6 L120-121, D11 L122, D12 L59, D13 L3,
  D13 L246-250 each list command names (`SetTaxRate`, `SetQueue`,
  `SetResearch`, `BuildBuilding`, `BuildShip`, `ProposeTreaty`,
  `DeclareWarCommand`, `CreateTradeRoute`, `AssignMission`,
  `MoveTo`, `Invade`). D13 L65 declares `commands: List<Command>`
  but the type itself lives only in a forward reference. **Fix**:
  add a `Command` ADT declaration in D1.2 (list all variants
  cross-referenced above) so D13's spec is testable; A3 can then
  refine it without changing the variant list.

### Medium (low-risk; can defer to spec phase)

- **R4.14 `Trait.HyperExpansion` removal comment in D1 L41** — D1
  L41 says "HyperExpansion is no longer in the ADT — it had no race
  assigned; v2 could re-add it as a fifth trait slot." But D3.1's
  race catalog has every race carrying exactly **2** traits (D3.1
  table), not 3-5. The comment's wording ("fifth trait slot")
  implies a 3–5 trait-per-race structure that doesn't match the
  rest of the doc. **Fix**: rewrite the comment to match the actual
  2-trait invariant.

- **R4.15 "Tribute" vs "Subjugation" terminology** — D11 uses
  `Subjugation` for the vassal treaty kind (L101, L110-111) but the
  trait-modifier system and D11.3's strategy table never use the
  word "tribute" after R1 resolution. Verify no stale references to
  "Tribute" remain anywhere (`grep -rn Tribute docs/`).

- **R4.16 `D11-diplomacy.md` L274 dependency table** — cites
  `aiPersonality(race)` from D3.2/D3.4. There is no helper named
  `aiPersonality(race)` in D3; the helper is `aiDefaultStrategy(race)`
  (D13.1, returns a `StrategyKind`, not a `Personality`). **Fix**:
  replace with `aiDefaultStrategy(race)` or define a thin
  `aiPersonality(race)` wrapper if the term is needed for parity with
  D11.3's argument naming.

- **R4.17 `D11-diplomacy.md` L47 — `Relation.playerA / playerB`
  redundant given map keying** — `Relation` is stored at
  `state.relations[(a, b)]`, so embedding `playerA`/`playerB` in
  the value is duplicate. Either drop the fields or document them
  as "denormalized convenience for the UI layer; must equal the
  map key".

- **R4.18 `Trait.Tolerant` modifier value in D3.2 L144** — currently
  `{ Diplomacy(1) }` with the comment "was missing from prior
  drafts". R4.1's fix would consolidate this into L129. Decide
  whether the diplomacy bonus is +1 or +2 (the rest of D3.2's
  Diplomacy values are small ints: `Repulsive = -2`,
  `Tolerant = +1`).

### Low (housekeeping; non-blocking)

- **R4.19 `D5-economy.md` L213** — `BuildShip(designId, count) →
  count × design(state, designId).cost` (where `cost` is
  `CombatStats.cost`). D7.5's `CombatStats.cost` is computed from
  `hull.baseCost + weapon.cost + special.cost` (D7 L174). This is
  fine, but the field name `CombatStats.cost` overlaps semantically
  with `Hull.baseCost` and `Building.baseCost`. Consider renaming
  to `CombatStats.buildCost` for clarity, or add a one-liner
  explaining that "CombatStats.cost is the bc cost to build one
  ship of this design" (matching the build-economy concept, not
  the combat concept).

- **R4.20 `D7-ship-design.md` L171** — `speed: int, // hull.baseSpeed
  (v1; v2 may add drive specials)`. The comment implies drive
  specials are v2, but the v1 `SpecialKind` catalog (D1 L44) lists
  `FuelTank` and D7 L127 lists `FuelTank` as "Placeholder specials
  in catalog, but no v1 effect yet". So speed is *not* the only
  v2-affected combat stat — `HyperDrive` is also v2. **Fix**: the
  comment should say "drive specials (HyperDrive, FuelTank) deferred
  to v2; speed = hull.baseSpeed in v1".

- **R4.21 `D11-diplomacy.md` L243** — "A treaty allows trade (Trade
  Pact, Alliance, NAP — NAP allows trade in v1)." D11 L108 says
  `TradePact` enables `TradeRoute`, and L329 says "Trade Pact, NAP,
  Alliance all enable trade routes". But the `Alliance` description
  on L108 doesn't mention trade. Pin whether `Alliance` enables
  trade at full or reduced income (currently Trade Pact enables it;
  if Alliance enables it too, what's the bonus?).

### Resolutions applied in this PR

| ID | Resolution |
|---|---|
| R4.1 | Dropped `traitModifiers(Toltec)` from `D3.2`; kept `Tolerant = { Diplomacy(1) }` (Gnolams' actual trait). |
| R4.2 | `D10 L135`: `Honorable +1` → `Honorbound +2` (matches D3.2's `traitModifiers(Honorbound) = { Morale(2), … }`). |
| R4.3 | `D3.1` Psilons row: `Weapons +1 (extra)` → `Weapons, Shields (D3.3: Weapons +2, Shields +1)` (Psilons have dual affinities in D3.3). |
| R4.4 | Replaced `D1.2 Relation = { level, modifiers }` with the full D11.1 record (`playerA, playerB, score, treaties, tradeRoutes, warState, lastContactTurn`). D11.1 now references D1.2 instead of redefining. |
| R4.5 | Replaced D1.1's stale `TreatyKind` list with a pointer to D11.2's authoritative ADT (NonAggressionPact, TradePact, Alliance, PeaceTreaty, Subjugation, ResearchAgreement). |
| R4.6 | Dropped `Player.relations, spies, treaties` (declared on `Player` but only written on `GameState`). Added note explaining the storage location. |
| R4.7 | Typed `Player.aiPersonality` as `Personality` (D13.1 record) in D1.2. |
| R4.8 | Dropped `ShipDesign.buildQueue` (never written; build queue is per-player in D5.7). |
| R4.9 | Pinned `STARTING_TECH_LEVEL = 1`, `INITIAL_POPULATION = 5` in D1.1. |
| R4.10 | Pinned `Relation.lastContactTurn` initial value: `state.turn` at pair creation (D11.1). |
| R4.11 | Pinned `JUST_MADE_PEACE_TURNS = 5` in D1.1; documented `JustMadePeace(until: state.turn + JUST_MADE_PEACE_TURNS)` and re-war rule. |
| R4.12 | Added `WarState` ADT to D1.1 (AtPeace/AtWar/JustMadePeace) with initialization rule. |
| R4.13 | Added `Command` ADT to D1.2 as the single source of truth; D13 dependency table updated. A3 retains validation/queue responsibility but no longer defines variants. |
| R4.14 | Reworded the `HyperExpansion` comment: "v1 ships without it. v2 could re-add it once a race's third trait slot exists." |
| R4.15 | Verified: no stale `Tribute` enum value remains. D5/D11 use "tribute" only in the noun sense (a maintenance-line payment from vassal to suzerain). |
| R4.16 | `D11` and `D13` dependency tables now reference `aiDefaultStrategy(race)` instead of the nonexistent `aiPersonality(race)`. |
| R4.17 | Kept `Relation.playerA/playerB` for now (they're useful in event payloads and tests). Documented the redundancy in the D1.2 record comment. |
| R4.18 | Resolved by R4.1: `Tolerant = { Diplomacy(1) }` (the L144 Gnolams-correct value). |
| R4.19 | Added inline comment to `CombatStats.cost`: `// bc to build one ship of this design = …`. |
| R4.20 | `D7 L171`: "v2 may add drive specials" → "v2 may add HyperDrive / FuelTank specials that increase it". |
| R4.21 | Pinned trade-rate modifiers in D11: TradePact ×1.20, Alliance ×1.20, NAP ×0.80. PeaceTreaty/Subjugation/ResearchAgreement do not enable trade. |

---

## Round 3 — findings (2026-06-05)

A third docs-review pass over the full D-layer surfaced 37 findings
not captured in the prior resolved/still-open lists. Of these,
**35 were resolved in the same PR** (the section/doc edits and the
PLANNING.md decision-log entries above) and **2 remain open** (9.1
and 9.3 — both deferred to Phase 2 spec work).

### Resolved in this PR

See `PLANNING.md` decision-log entries dated 2026-06-05 (rows
197–218) for the full per-finding resolution trail. Summary:

- **Section 7 (critical, 1/1 resolved)** — D5 orchestrator reordered
  so D5.6 (morale) runs before D5.3 (industry).
- **Section 8 (high, 15/15 resolved)** — D1.2 Player/Building/Spy/
  Mission records extended with missing fields; Ship semantics
  pinned; FighterBay/BomberBay rename; cross-entity helpers declared;
  constants pinned.
- **Section 9 (medium, 7/9 resolved)** — `SpyDetection` semantic
  documented and wired into D12.5; D9.7 placeholder added to
  DECOMPOSITION.md; AIPipeline consumer documented; CombatStats.speed
  marked cosmetic; REVIEW-NOTES 6.5 stale claim retired; trade-route
  distance formula pinned; council-vote coupling flagged as
  deliberate.
- **Section 10 (low, 12/13 resolved)** — naming normalized (payload
  fields, sideAId vs sideA); missing D1.2 fields added; `Player.score`
  wording reconciled; `aiDefaultStrategy` first-match-wins documented;
  `BuildingKind.cost` corrected; mission duration table in D1.1;
  colonization loop added (D8.6); REVIEW-NOTES 6.3 stale count retired.

### Still open (deferred to Phase 2 spec work)

- **9.1 Per-pair starting relations (4.2 — still open)** — the 10×10
  matrix is deferred to spec phase. D11.1 Quint spec can't be
  testable without it. *Owner: D11.1 spec phase. Will pin during the
  D11 implementation.*
- **9.3 AI scoring heuristics (4.18 — still open)** — `value =
  immediate_benefit × race_modifier` with no values pinned. *Owner:
  D13.6/D13.7 spec phase. Tests will pin the coefficients when the
  AI command-evaluator is written.*

(The remaining "stale-finding" items 6.3/6.5/10.9 are *retired* —
the original numbers were wrong and the underlying reconciliation
tasks they described were addressed by the round-3 resolutions.)

### Section 7 — 10 history (kept for traceability)

The full finding text from the initial round-3 review is preserved
in `git log --follow docs/REVIEW-NOTES.md` (commit `879f84a`) for
any future reviewer who wants to see the as-found wording.

---

## How to use this document

When you make changes that resolve a finding above:

1. Remove the finding from the "Still open" list (or move it to
   "Resolved").
2. Add a 2026-06-05 (or later) entry to the PLANNING.md decision log
   describing the resolution.

The next docs review pass should start by re-reading this file's
"Still open" section.