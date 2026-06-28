# Open Questions and Inconsistencies — Pre-Quint Audit

**Date**: 2026-06-05
**Scope**: 17 docs reviewed (`README.md`, `PLANNING.md`, `ARCHITECTURE.md`,
`DECOMPOSITION.md`, `TESTING.md`, `ROADMAP.md`, `REVIEW-NOTES.md`, and all
14 `docs/sections/D<n>-*.md`). No Quint specs or TypeScript exist yet.
**Source of truth**: I cross-checked the docs themselves; this document does
not restate findings the docs have already resolved in REVIEW-NOTES.md
(round 1–3) or PLANNING.md's decision log.

The project has already done substantial self-review — REVIEW-NOTES.md has
three rounds of audit and PLANNING.md's decision log has 60+ entries from
2026-06-05. This report surfaces (A) items the docs themselves flag as
still open, (B) cross-section inconsistencies I found that look unresolved,
and (C) methodological / architectural gaps that matter for Phase 2.

---

## A. Items the docs themselves still flag as open

### A1. Per-spec questions deferred to Phase 2 (REVIEW-NOTES §4)

These are tracked in `docs/REVIEW-NOTES.md` and explicitly *not* resolved:

| # | Finding | Where it bites |
|---|---|---|
| 4.2 | **10×10 per-pair starting-relations matrix** for D11.1 | D11.1's Quint spec cannot be unit-tested without it; current text says only "default to 0 (neutral) — decide and pin." |
| 4.3 | **Final trait list** (16 traits proposed in D3.1, but exact names/assignments not pinned) | D3.1 has names ("Skilled, Versatile, Flight, Warrior, Militaristic, Subterranean, Stealthy, Cybernetic, Tolerant, Repulsive, Industrial, Creative, Honorbound, Telepathic, Erudite, Lithovore") but REVIEW-NOTES says "the exact final list of ~16 traits still needs to be pinned at the spec phase." |
| 4.4 | **Tech-victory key techs** — D14.2 says "top of each tree (6 techs)" but the 6 specific `TechId`s are not named | D14.2 test cannot run without the set. |
| 4.5 | **Score-formula weights** are placeholders | D14.4 ships `1 × planets + 5 × techs + 0.001 × pop + 0.1 × fleet + 0.001 × treasury`. Documented as "tune during playtesting." Council-vote rank ordering will mirror this and inherit the same balance issues. |
| 4.6 | **"Parsec" unit for trade distance penalty** is undefined | D11.6 uses `euclideanDistance(parentStar(p1), parentStar(p2))` but `Star.x` / `Star.y` are integers with no documented scale. "Parsec" is mentioned but not pinned to a constant. |
| 4.7 | **Council on turn 0** | D11.5's check `state.turn % COUNCIL_INTERVAL == 0` fires on turn 0 too. No relations or meaningful scores exist then; doc says "spec needs to confirm … or skips to turn 25." |
| 4.8 | **Subjugation edge cases** | If A is vassal of B and B is at war with C, vassal becomes AtWar with C. But what if vassal is *already* at war with C from before subjugation (or has its own alliance with C)? No rule specified. |
| 4.9 | **"Forced peace treaty"** is a victory-related concept in D14.4 body text but the rule for when a treaty is "forced" is unspecified. | Unused by v1 victory checks today, but it lingers as an under-defined term. |
| 4.10 | **Retreat destination algorithm** — "nearest star within warpRange, ties random via rng" is documented but warp-range usage and "nearest" metric are open. | Both D9.5.4 and D10 open-question 4 share the same algorithm; one helper in D8 would remove duplication. |
| 4.11 | **Spy production cost / prereqs / limits** | `SPY_PRODUCTION_INTERVAL = 10` is pinned, but cost (bc?) and per-player max spies are loose. |
| 4.13 | **Full v1 weapon catalog** — D7.2 lists shape + kind names ("Laser, Ion Cannon, Phaser, Disruptor, Death Ray, Missile, Heavy Missile, Multi-Warhead Missile, Torpedo, FighterBay, BomberBay") but no `damage/range/shots/accuracy/spaceCost` numbers are pinned. | D9.3 damage formulas need these. |
| 4.17 | **AI cooperation on captured planets** — "AI cooperation on captured planets" mentioned but not specified. | D13. |
| 4.18 | **AI "value" heuristic specifics** — `value = immediate_benefit × race_modifier` with no values pinned | D13.6/13.7. Tests cannot pin coefficients until they're specified. |
| 4.19 | **`BC_PER_TAX_POINT = 1` scale not validated** against maintenance costs | D1.1; D5.5 doesn't actually reference `BC_PER_TAX_POINT` anywhere — see inconsistency B8. |
| 4.20 | **`BuildingEffect.Maintenance` variant** doesn't exist; v1 sum is 0 | D5.5 acknowledges but doesn't add a "v1: no building maintenance" resolved decision. |

### A2. Architectural / methodological items (REVIEW-NOTES §5)

- **5.1 A-layer and P-layer not decomposed** (acknowledged). The Phase 1 roadmap still needs A1–A5 and P1–P14 to be broken down.
- **5.2 Quint-spec modularization of D4** — how to split `step` across Quint files is a spec-phase decision.
- **5.3 A4 Turn Manager order** — "AI vs human turns needs to be pinned." Tied to inconsistency B25 below.
- **5.4 A2 persistence schema-versioning shape** — JSON shape and migration table not defined.
- **5.5 Verify all `Math.random()` sources removed** during Phase 2 spec work.
- **5.6 No spec for observer players / spectators** — out of v1 scope, fine, but should be explicitly excluded.
- **5.7 Save-migration placement** — `initTurn` calls `migrateIfNeeded` every turn: is this idempotent or re-checks? Pin during Phase 2.
- **5.8 "Pause-on-detection" deferred to v2** — D12.5 mentions it; no chunk assignment.

### A3. Open questions still in per-section docs

Items each section doc explicitly lists under "Open questions for D<n>" but
does not resolve:

- **D2.6**: "guarantee a minimum number of candidates" (deferred to v2 with a UI re-roll fallback).
- **D2.0**: "observer race count field" — deferred.
- **D3.1**: trait list scope and exact names (deferred to spec phase).
- **D3.4**: per-pair diplomacy disposition values (deferred to spec phase).
- **D6.2**: tech-cost scaling — linear vs quadratic (`Default v1: linear with level`).
- **D6**: cross-tree prereq count (~5 for variety), tech discovery from exploration (deferred), negative-effect techs (deferred), tech refund (deferred).
- **D7.1 / D7.2**: exact stat numbers (default to MoO values; pin during spec).
- **D7.5**: whether `computeCombatStats` validates first or assumes valid input (`Default: assumes valid input`).
- **D7**: transport hull weapons check (default: unarmed, `slotCount = 0`).
- **D8**: minimum fleet size for combat (default: no minimum).
- **D10.4**: pillage % baseline (default 10%), native resistance (default 5%), defensive buildings effect (default +10 morale).
- **D11.3**: alliance defense obligation (default: yes, with one-turn delay).
- **D11.2**: research-agreement mechanics (default: 50/50 split).
- **D11.6**: trade distance penalty (default -1%/parsec, capped -50%).
- **D11**: subjugation tribute (default 10% of gross income).
- **D12**: spy skill growth (+1/success, cap 10), `SPY_PRODUCTION_INTERVAL = 10`, initial spy count = 1, `StealTech` duration = 3.
- **D13.1**: AI intelligence sharing (no), AI cooperation (no), AI colony treatment (keep natives), AI value heuristic (heuristic, tune in playtesting).
- **D14.4**: score-formula balance (placeholder), tech-victory key techs (top of each tree), draw condition at MAX_TURN (draw, no tiebreaker in v1).

The sections explicitly state these do **not** block starting the Quint
specs, but the bulk of them ("tune during playtesting") won't be
test-pinnable until they're actually tuned.

---

## B. Inconsistencies / underspecifications I found that aren't (yet) in REVIEW-NOTES

These are the items I'd flag if I were reviewing this PR. None block
starting Phase 2, but several need resolution before the relevant spec
can be written.

### B1. *Research-agreement transfer has an ordering bug.* (severity: high)

D5.5 owns the 50/50 split of unspent research between research-agreement
parties. The doc says:

> "The transfer is applied **inside D5.5** as a maintenance line … so the
> total is conserved … D5.5 reads the treaties map and applies the
> transfer inline."

And:

> "split 50/50 between them after D6.3 has deducted the cost of any
> tech-acquired this turn"

But **D4's phase order runs `economy` (which contains D5.5) before
`research` (which contains D6.3)**:

```
1. initTurn  → 2. economy (D5: D5.6 morale → D5.2/D5.3 food&industry → D5.4 research → D5.5 income → D5.7 queues)
3. research (D6.3 applyResearch)
4. production …
```

D5.5 cannot read `Player.researchAccumulated` *after* D6.3 has deducted
the tech cost, because D6.3 has not run yet when D5.5 executes. Three
possible fixes; the doc needs to pick one:

1. Move the research-agreement split out of D5.5 into a new step that
   runs *after* D6.3 (e.g., add a "post-research" phase between
   `research` and `production`, or run the split inside D6.3 itself).
2. Reorder D4 phases so research runs before economy (would require
   D5.4 to read `Player.researchAccumulated` from last turn, breaking the
   "current turn" semantics).
3. Have D5.5 read `player.grossResearch` (produced by D5.4 earlier in
   the same economy pass) and do the split on that pre-acquisition
   number, accepting that the 50/50 number may include the cost of a
   soon-to-be-acquired tech (slight total imbalance on tech-completion
   turns).

The decision log says "D5.5 owns the inline implementation; D11.2's
Treaty description references it" but never resolved the ordering
collision. This is the kind of thing that will silently break the
conservation invariant in tests.

### B2. *Native-planet industry cap after colonization is ambiguous.* (severity: medium)

D5.3 says:

> "If the planet is `Native` or `Artifact`, the industry is capped
> (natives/artifacts resist colonization-industry). v1: capped at 50%
> until colonized (no native resistance after colonization)."

But D8.6 colonization allows colonizing Native planets (the colony ship
claims them). The phrasing "until colonized" is ambiguous:

- After colonization, does the 50% cap lift entirely (the natives accept
  the new owner)?
- Or does the 50% cap persist as long as `specials.contains(Native)` (the
  natives are still there, regardless of ownership)?
- D2.6 explicitly excludes Native planets from the candidate-homeworld
  set, but D8.6 colonization does not — so Native *non-candidate* planets
  can become colonized during play.

The interaction between `Planet.specials.contains(Native)` and
`Planet.owner.isSome()` needs an explicit rule. D5.3 should pin:
`(Planet.owner.isSome() && specials.contains(Native)) → ?`.

### B3. *Council on turn 0.* (severity: medium)

D11.5: `state.turn % COUNCIL_INTERVAL == 0` fires on turn 0, 25, 50, 75, ….
REVIEW-NOTES 4.7 acknowledges this. Concretely: at turn 0 there are no
diplomatic relations yet (D11.1 has not run any drift or treaty updates),
and `state.score[player]` is the turn-0 score (probably all-zero or all-equal).

Two reasonable resolutions:
- Detect `state.turn == 0` and skip the council on the first turn.
- Run the council on turn 0 with everyone having 1 vote (or score-based
  but all scores equal → "no winner," next council in 25 turns).

The doc needs to pick one.

### B4. *D14.5 battle-event stats need a real source.* (severity: medium)

D14.5 says `battlesWon` and `battlesLost` are computed by counting
`BattleResolvedEvent`s "across the game's event log (plus persisted events
flushed to disk)."

But D1.1 caps the in-memory event log at `MAX_EVENTS_IN_MEMORY = 200`
with "older events flushed to disk on save." For end-of-game stats on
turn 200, the in-memory log has at most 200 events. Many `BattleResolved`
events may be on disk, not in memory. The doc says "plus persisted events
flushed to disk" but does not say *who* reads them back at game end.

Options:
- A4 Turn Manager replays the persisted event log into D14.5.
- D14.5 directly queries the persistence layer (which violates the
  "domain is pure" principle in ARCHITECTURE.md Principle 1).
- `Player.battlesWon` and `Player.battlesLost` become maintained fields,
  incremented each time D9.6.5 emits `BattleResolvedEvent`.

The third option is consistent with the pattern used for treasury
(`Player.treasury` is maintained, not derived). Pin one during Phase 2.

### B5. *D14.4 score is not written in any D4 phase.* (severity: medium)

D1.2 and D14.4 both say `state.score[player]` is recomputed once per turn
by a "D4 helper" that runs between economy and victory check. D14.4 also
says the helper runs between economy and victory. **But D4's documented
phase order is `init → economy → research → production → fleetMovement →
contactResolution → combatResolution → espionage → diplomacy →
victoryCheck → endTurn` — there is no phase between economy and victory
check that owns the score recompute.**

Phase order in D4 says victoryCheck uses the cached score. So the
recompute must be one of:

1. Part of `endTurn` (just before victoryCheck — but endTurn is *after*
   victoryCheck in the listed order).
2. A new unnamed phase between diplomacy and victoryCheck.
3. Run as part of `initTurn` of the *next* turn (one-turn lag, breaks
   D14.4 / D11.5 "same-turn" semantics).

D4's phase table needs a row for this. The current doc just refers to
"D4 helper" without saying where it lives.

### B6. *AI-vs-human command ordering is underspecified.* (severity: medium)

D13 decision: "AI commands are generated *before* the human's turn ends
by A4 Turn Manager calling D13; submitted along with human commands;
processed in one batch by `step`."

But the human's commands are issued across the turn via P3 / P4 / P7 /
etc. and queued in `state.commandQueue`. D13's `AIInput` reads `state`
to produce commands. The doc does not pin whether D13 sees:

1. State at the start of the turn (pre-human-commands).
2. State after human commands are queued but before step runs
   (post-human-commands, pre-step).

Option 2 means D13 sees an intermediate snapshot that doesn't have
human commands applied to state yet. Option 1 means D13 sees stale
state. The doc should pin which.

Related (B20 below): if AIs run *sequentially* with state updated between
each AI's decision, the AI ordering is dynamic. If AIs see a single shared
snapshot, ordering is deterministic. Pin one.

### B7. *The Trait→Modifier system has one special-case trait: Lithovore.* (severity: low)

D3 documents 16 traits. Effects are expressed through the `Modifier` ADT
*except* Lithovore, whose effect ("ignores food / no starvation") is
special-cased in D5.1:

> "If `food < 0` and race is Lithovore: no starvation (Lithovore eats
> minerals, not food)."

This works but is inconsistent with how every other trait expresses its
effect. Cleaner: add a `NoFoodRequirement: bool` (or `FoodOptional`) variant
to `Modifier` (D3.2) and have D5.1 read it via `totalModifiers(race)`.
Same for `Ruthless` AI personality hint affecting pillage (D10.4 doubles
population loss) — this is a "trait-or-personality?" question that
currently lives as an open-coded `if` in D10.4.

Decide during Phase 2 whether Lithovore and Ruthless stay special-cased
or become `Modifier` variants.

### B8. *`BC_PER_TAX_POINT` is defined but unused.* (severity: low)

D1.1 declares `BC_PER_TAX_POINT = 1` ("1 bc per tax point per turn — the
v1 scale used by D5.5 and D11.3"). But:

- D5.5's formulas use `PLANET_BASE_TAX_INCOME = 1` × `(taxRate / 100)`,
  not `BC_PER_TAX_POINT × taxRate`.
- D11.3's evaluation table uses absolute values ("Money: +10"), not
  scaled by `BC_PER_TAX_POINT`.

`BC_PER_TAX_POINT` may be vestigial, or the docs may have intended the
income formula to be `BC_PER_TAX_POINT × taxRate` (one number per tax
point per turn, summing across planets). The naming implies the latter
but the formulas use the former. Reconcile, or remove the constant.

### B9. *Per-ship initiative vs. per-stack HP needs an ordering rule.* (severity: medium)

D9.1.2: "Per-ship initiative (v1 simplification vs. MoO's per-stack
initiative; v2 may restore per-stack)."

D9.3.2: damage formula reads `hpCurrent / hpMax` per *stack* (each stack
entry is `{designId, count, hpCurrent, hpMax, experience}`).

So ships fire in some per-ship initiative order but their HP is pooled
per stack. When does a stack die (HP = 0)? When does a single ship within
a stack die (count decremented)? D9.3 doesn't pin this. Options:

- Per-stack HP: damage reduces `hpCurrent`; stack is "destroyed" when
  `hpCurrent == 0`. `count` stays the same (no per-individual ships).
- Per-ship HP: damage to a stack first reduces `count`, then on `count == 0`
  removes the stack entry.

D1.2 says `Ship` is per-individual but `Fleet.ships` is a stack grouping
— so per-stack HP with stack-count decrement (option 2) is probably
intended, but D9.3 needs to spell it out.

### B10. *Trade-route tech-level helper isn't defined where it's used.* (severity: low)

D11.6 uses `techLevel(player, TechTree.Computer)` as a multiplier on
trade-route income. But D6 (Research) defines `Tech` records with a
`level` field per tech, not a `techLevel(player, tree)` function that
counts a player's techs in a tree.

D6 should declare a derived helper, similar to D1.2's cross-entity
helpers (`playerTotalPlanets`, `habitablePlanetIds`). Otherwise the
trade-income formula references an undefined symbol.

### B11. *D13 threat metric's stability across AI turns is unspecified.* (severity: low)

D13.5: "Threat" is `Σ (relationScore * -1) for each at-war relation +
ownPlanetCount / totalPlanets`.

AIs run *sequentially*. After AI #1 produces commands, does the state
update before AI #2's threat is computed? If yes, AI ordering is dynamic
— AI #2 might "see" AI #1's threat change. If no (snapshot once at
batch start), AI ordering is deterministic.

The resolved-decisions list says "AIs run sequentially, ordered by
threat" without saying when threat is measured. Pin during spec phase.

### B12. *Retreat destination logic is duplicated between D9 and D10.* (severity: low)

D9.5.4 (space-combat retreat) and D10 open question (invader retreat)
both describe "nearest star within warp range, picked randomly if multiple
options." These should share one helper in D8 (e.g.,
`pickRetreatDestination(fleet, state, rng)`). Currently each section
documents it independently and the doc has them inconsistent (D9.5.4 uses
`state.stars`; D10 says D8 picks the destination — different ownership).

### B13. *Council + conquest in the same turn.* (severity: low)

D11.5 emits `GalacticEmperorVictoryEvent` if a player has > 50% of vote
count. D14.3 reads this event to declare diplomatic victory. But D14.1
(conquest) is checked at the same point in the phase order, and D14.4
(score) runs at MAX_TURN only. If two players simultaneously meet
different victory conditions in the same turn, what happens? The doc
says "victoryCheck before end-of-turn cleanup (game ends mid-step)" but
doesn't pin resolution order. Likely "first-match-wins" but should be
explicit.

### B14. *D14.4 score formula doesn't include production; P12 reads the same field for council votes.* (severity: low)

D14.4 weights: planets (1×), techs (5×), pop (0.001×), fleet (0.1×),
treasury (0.001×). Production is *not* in the score.

D11.5 council-vote count = `state.score[player]`. So a player with a big
treasury but few planets dominates the council.

The doc acknowledges this is "acceptable for v1 council voting because the
council is a soft diplomatic mechanism" but it's a deliberate
simplification worth pinning in the decision log. Note also that
REVIEW-NOTES 6.7 originally flagged `Player.totalProduction` as a
"dangling dependency" — production was removed from the score formula
and the dependency was cleaned up. The D5 cross-section table still says
"D14 Victory | Reads `Player.treasury` for score (production is not in
the v1 score formula)" which is correct, but the council-vote
implication isn't stated there.

### B15. *D8.5 split validation message talks about ships, not design slots.* (severity: low)

D8.1 validation: "fleet has more ships than the requested split (must
leave ≥1 ship in each result)."

D8.5 split: `shipIndices` is "ship-design slots" (one per
`ships: List<{designId, count}>` entry). v1 splits only by *whole* slots,
not partial counts.

The validation message ("more ships than the requested split") doesn't
match the actual operation (split-by-design). Either:
- Update validation to "fleet has more *design entries* than the
  requested split" (clearer).
- Or rename `shipIndices` → `designSlotIndices` everywhere.

### B16. *D5.7 ship build destination ("designated build fleet") is undefined.* (severity: low)

> "The completed ship is added to a designated build fleet at the planet
> (creating it if it doesn't exist; ownership is the player)."

What is the "designated build fleet"? Does each owned planet always have
exactly one build fleet? Does it survive between turns? If a planet is
unowned (`owner == None`) but somehow has a queue (shouldn't happen, but
the doc doesn't preclude it), where does the ship go?

D1.2 should declare a `Planet.buildFleetId: Option<FleetId>` field, or
D5.7 should pin the rule ("if planet has no build fleet, create one
named `FleetId` `planet.buildFleetId`; otherwise append ships to it").
Currently under-specified.

### B17. *D6.3 implicit assumption: the "treasury pays for tech" model.* (severity: low)

D6.3 says "Cost is paid in accumulated research points" — research is a
separate currency from treasury. Fine. But D5.4's `grossResearch` is in
research points; D6.3 consumes them; D5.5 then has the research-agreement
split issue (B1). The flow of research-points-as-currency is consistent,
but the boundary between research-points and treasury is murky: does the
research slider convert research-points to treasury bc? The formula
`floor(player.grossResearch × (taxRate / 100))` adds bc to treasury.
Where does the source research-points go? Probably they're consumed by
D6.3's accumulator (`player.researchAccumulated`), and any unspent
amount is lost (D6.3: "switching research loses accumulated points").
Fine, but the doc doesn't say so explicitly. D1.2's `Player` definition
has both `treasury` and `researchAccumulated` — two currencies. Pin
which operations consume which.

### B18. *`PLANET_BASE_TAX_INCOME` baseline interaction with high-tax worlds.* (severity: low)

D5.5: `grossIncome = Σ planet.baseTaxIncome + Σ tradeRoutes(player).income
+ floor(player.grossResearch × (taxRate / 100))`.

`planet.baseTaxIncome = PLANET_BASE_TAX_INCOME × (taxRate / 100) = 1 ×
(taxRate / 100)`.

So at `taxRate = 100`, a player with 10 planets gets `10 × 1 = 10` bc/turn
from base tax. At `taxRate = 0`, they get 0 bc/turn from base tax but
100% of their research stays as research. MoO behavior.

But: research is then a separate income stream at high tax. The "tax
slider" actually controls two things (income from planets AND
research→income split). The doc treats this as a single "tax rate"
slider. Worth a one-sentence explicit note: "the taxRate slider controls
both the per-planet base income and the research→bc conversion rate;
players trade research progress for short-term cash by raising tax."

### B19. *D5.6 morale formula references an undefined `recentlyConquered` function.* (severity: low)

D5.6 lists `+recentlyConquered(planet)` as a modifier, with value "-20 if
conquered last 5 turns." This function isn't defined anywhere. It should
read `Planet.conqueredOnTurn` (D1.2) and compare to `state.turn - 5`.

D5.6 should explicitly spell out: `recentlyConquered(p) = if
p.conqueredOnTurn.isSome() and state.turn - p.conqueredOnTurn.unwrap() <
RECENTLY_CONQUERED_TURNS then -20 else 0`. Currently the function is
used but not declared.

### B20. *D8 ETA-locked rule contradicts D8.3's first draft.* (severity: cosmetic)

D8.3 says "ETA is **locked at order issue**" but earlier in D8 there's a
note "A previous draft of this section said the opposite; that was an
error." The note is a fossil — fine — but a clean reader might still
notice it as a discrepancy. Consider removing the meta-note once the
spec phase starts.

### B21. *D9 phase "7b" numbering is inconsistent with the chunk list.* (severity: cosmetic, already retired in REVIEW-NOTES 6.2)

D9.6.5 mentions phase 7b (ground combat), and D4's orchestrator lists
"7b. groundCombat (D10)" between combat and espionage. The D9 chunk
list doesn't have a "7b" reference; D9 itself has no ground-combat phase.
REVIEW-NOTES 6.2 says this was retired by round-3 audit. Worth a final
sanity-check that all "7b" references in D4's pseudocode are removed or
renamed consistently.

---

## C. Methodological / architectural gaps

These are not inconsistencies in the docs themselves, but things the
project will need to decide before/early in Phase 2.

### C1. The Quint spec modularization for D4 is unresolved.

D4 is the only section small enough to live in one Quint file (per
D4's section doc). But D4 imports every other D-section's `step` / `resolve`
function. In Quint, `import` is at the module level — does D4's
`turnCycle.qnt` import every chunk? If so, what's the natural unit of
modularization (per-section chunk? per-phase chunk?)? REVIEW-NOTES 5.2
flags this as a spec-phase decision. Worth resolving before writing any
spec, since it dictates file layout for the whole `specs/` tree.

### C2. Quint type system vs. the open ADTs.

Several ADTs are explicitly described as "open" (D3.2 `Modifier`, D6
`TechEffect`, D1.2 `BuildingEffect`, D7.3 `SpecialEffect`). In Quint,
variant types are open by default (`type` syntax), so this is fine. But
the docs should be explicit that consumers must pattern-match
exhaustively (otherwise v2 additions break callers). Consider adding a
"consumers must use exhaustive match" note to each open ADT.

### C3. No fixture format decided.

TESTING.md says fixtures live in `specs/fixtures/smallGalaxy.json`,
serialized through the persistence layer. But A2 persistence hasn't been
decomposed, and the format isn't specified. Pick JSON shape (or
MessagePack if size becomes a concern — ARCHITECTURE.md §7 leaves this
open) before Phase 3.

### C4. Property-test strategy uses fast-check + Quint simulate.

TESTING.md mentions both fast-check (for TypeScript) and Quint
`simulate` (for Quint). These need a shared property-set: "fleet size
is conserved," "no negative treasury," etc. Decide which invariants
each tracks. Easy to defer, but document it.

### C5. The D5 dependency table references a comment about production / score.

D5's cross-section table line: "D14 Victory | Reads `Player.treasury`
for score (production is not in the v1 score formula — see REVIEW-NOTES
6.7/8.3; if v2 adds production, it would read the per-turn gross via
`state.score` after D4's recompute)."

This is correct but the parenthetical references review-notes that may
go stale. Consider pinning as a resolved decision in PLANNING.md rather
than a doc-internal reference.

### C6. The "v2 placeholder" entries should live in a single catalogue.

D7.3 has 4 functional specials + 6 placeholder entries
(PointDefense, HyperDrive, FuelTank, BattleScanner, DamageControl,
AutomatedRepair). D9 has D9.7 boarding as a placeholder. D8 has warp
lanes deferred. D11 has alliance-weight + denouncement + liberation
deferred. D13 has alliance coordination + per-AI difficulty deferred.

These are scattered. A `docs/V2-CHUNKS.md` (or appendix) listing every
v2 placeholder would make future work easier and clarify what's *not* in
v1. Optional, but useful for anyone picking the project up after the v1
ship.

### C7. Some tuning knobs lack a target benchmark.

"Default v1: 10% pillage," "Default v1: 5% native resistance," "Default
v1: -10 per war (capped -30)" — these are tagged "tune during
playtesting" but no target benchmark is documented (e.g., "an average
game should produce 5 successful invasions per game before end-game").
Without a benchmark, "tune" means "eyeball it." Not blocking, but worth
flagging during Phase 2.

---

## D. Summary of severity

| Severity | Count | Items |
|---|---|---|
| **High** (likely breaks a Quint spec / invariant test) | 1 | B1 (research-agreement ordering) |
| **Medium** (real ambiguity, needs decision) | 5 | B2, B3, B4, B5, B6, B9 |
| **Low** (cosmetic / doc-cleanup) | 13 | B7, B8, B10–B21 |
| **Methodological** (Phase 2 prep) | 7 | C1–C7 |

## E. Recommended next steps

1. **Resolve B1 (research-agreement ordering) before writing the D5.5 or D6.3 specs.** This is the only "high" severity item and it has three plausible fixes; pick one.
2. **Pin B5 (where score recompute lives in D4)** during the D4 spec work — it's a one-line addition to D4's phase table.
3. **Convert A1 / A2 items above into PLANNING.md decision-log entries.** Most of these are 1-line resolutions.
4. **Open a follow-up docs PR for the medium-severity items** (B2, B3, B4, B6, B9) — each is small but they cross section boundaries and benefit from a single coherent revision.
5. **Consider writing a `docs/V2-CHUNKS.md`** that catalogs every deferred v2 placeholder (C6). This makes the v1/v2 boundary explicit for future contributors.
6. **No code yet.** All of the above is pre-Quint; the project is in the right phase to catch these before they ossify in code.

---

*This report is a snapshot. The next docs-review pass should start by
re-reading this document's sections A and B and the still-open items in
`docs/REVIEW-NOTES.md`.*