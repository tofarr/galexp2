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

## How to use this document

When you make changes that resolve a finding above:

1. Remove the finding from the "Still open" list (or move it to
   "Resolved").
2. Add a 2026-06-05 (or later) entry to the PLANNING.md decision log
   describing the resolution.

The next docs review pass should start by re-reading this file's
"Still open" section.