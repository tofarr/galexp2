# Section D9 — Space Combat (Pilot Decomposition)

This is the **pilot** section for the recursive decomposition workflow. We go from the high-level chunk list in [`DECOMPOSITION.md`](../DECOMPOSITION.md#section-d9--space-combat-resolution) down to Quint-spec-sized leaves.

Once this section is decomposed, we apply the same workflow to every other section.

## Section contract

> **Space Combat Resolution**: given two **sides** at a star, resolve the tactical space battle. A side is a collection of fleets owned by one player that have been auto-grouped at the same star. Produces a new state for both sides (survivors, retreat positions) and a stream of combat events.

**Owns**: tactical space battle resolution, including auto-resolve. The PixiJS renderer (P10) consumes the events but does not compute anything.

**Does not own**: ship design (D7), fleet movement (D8), AI decision-making (D13), ground combat (D10).

## Top-level chunks (from DECOMPOSITION.md)

- D9.1 Combat setup — initiative ordering, stack positioning
- D9.2 Targeting AI — who shoots whom per round
- D9.3 To-hit & damage calculation — formulas
- D9.4 Specials in combat — shields, ECM, point defense, fighter screens
- D9.5 Retreat mechanics
- D9.6 Combat outcome — survivors, XP awarded, scrap

These are still too big. Each becomes its own sub-document or section.

## Recursive decomposition

We go one level deeper. Each sub-chunk is *almost* leaf-sized: it can be specified and tested, but might depend on smaller primitives.

### D9.1 → Combat setup

- **D9.1.1 Pre-battle validation** — both fleets exist, are at the same star, have positive ships. (Pure check; cheap.)
- **D9.1.2 Initiative ordering** — sort ships by computer stat, ties broken by ship size. **Per-ship** initiative (v1 simplification vs. MoO's per-stack initiative; v2 may restore per-stack). Pure sort.
- **D9.1.3 Stack positioning** — assign starting coordinates on the tactical map. Pure projection from fleet composition.

### D9.2 → Targeting AI

The chunks below describe the *full* targeting AI. v1 ships a deliberately
simpler version; see "What simple v1 looks like for D9" below.

- **D9.2.1 Targeting policy** — what fraction of firepower each ship allocates to which target? Pure function: `(attackerShip, defenders[]) -> targetIndex`.
- **D9.2.2 Beam weapon target selection** — beams need line-of-sight; pick nearest target of appropriate size. **v1 simplification**: drop line-of-sight and "appropriate size"; beams fire at the nearest enemy regardless.
- **D9.2.3 Missile/torpedo target selection** — fly at highest-value target. **v1 simplification**: missiles/torpedoes also use greedy nearest targeting, like beams.
- **D9.2.4 Fighter/bomber behavior** — fighters escort, bombers attack ships. **v1 simplification**: fighters escort the fleet's largest ship; bombers attack the nearest enemy capital ship.

### D9.3 → To-hit & damage

- **D9.3.1 To-hit formula** — `(attacker.attack + weapon.accuracy) - (defender.defense + defender.shield + defender.ecm)` clamped, with random roll. Pure.
- **D9.3.2 Damage formula** — `weapon.damage × multipliers × remaining-hp-fraction`. Pure.
- **D9.3.3 Shield absorption** — shields absorb X damage per round, then fail.
- **D9.3.4 Armor mitigation** — armor reduces damage by percentage.
- **D9.3.5 Critical hits** — chance of multiplier on max damage roll.

### D9.4 → Specials in combat

- **D9.4.1 Shields** — damage gate per round.
- **D9.4.2 ECM** — affects to-hit on incoming fire.
- **D9.4.3 Point defense** — intercepts missiles/torpedoes/fighters before they reach main target.
- **D9.4.4 Fighter screen** — enemy bombers must engage fighters first.
- **D9.4.5 Weapon specials** — e.g., "phaser" ignores shields, "disruptor" ignores armor.

### D9.5 → Retreat mechanics

The chunks below describe the *full* retreat system. v1 ships a
deliberately simpler version; see "What simple v1 looks like for D9"
below. Boarding-related conditions are v2 (D9.7) and not active in v1.

- **D9.5.1 Retreat eligibility** — when can a stack retreat? In v1, any
  non-destroyed fleet may attempt retreat at the end of the round.
  v2 may add engines/boarding/trap conditions.
- **D9.5.2 Retreat order sequencing** — who retreats first. **v1 simplification**: only one side can retreat per round (the player who issues the retreat command on their turn); the other side chooses to pursue or hold.
- **D9.5.3 Damage-during-retreat** — pursuing fleet gets free shots. **v1 simplification**: one volley from the pursuer at the retreating fleet before disengagement.
- **D9.5.4 Destination** — retreat to a star within warp range. **v1 simplification**: the retreating fleet picks the nearest star in `state.stars` within its `warpRange` (D8.2); ties broken randomly via the deterministic `ctx.rng`. The destination is then handed to D8 as a normal `MoveTo` order.

### D9.6 → Combat outcome

- **D9.6.1 Survivor determination** — which ships live, which die. **v1 active.**
- **D9.6.2 Damage tracking per ship** — HP remaining. **v1 active.**
- **D9.6.3 XP awarded** — surviving ships gain experience; new designs unlock. **Deferred to v2.** The chunk is defined here so the spec can include its type signatures and tests, but D9.6.3 emits no events and has no state effect in v1.
- **D9.6.4 Scrap salvage** — destroyed ships leave debris; winner can collect. **Deferred to v2.** Same — chunk defined, no v1 effect.
- **D9.6.5 Battle resolution event** — at combat end, emit
  ```
  BattleResolvedEvent {
    star: StarId,
    winner: Option<FleetId>,        // None if outcome = Drawn
    loser: Option<FleetId>,         // None if outcome = Drawn
    outcome: Won | Drawn,           // Won: exactly one survivor; Drawn: both destroyed or COMBAT_MAX_ROUNDS exceeded with no decisive result
    turn: TurnId
  }
  ```
  v1 active. Consumed by D14.5 to increment `battlesWon` / `battlesLost`
  for end-game stats (D14.5 counts a `Won` event as 1 win for the
  `winner`'s player and 1 loss for the `loser`'s player; a `Drawn` event
  counts as 1 loss for each side's player).

## Dependency graph (within D9)

```
D9.1.1 (validation)
  ↓
D9.1.2 (initiative) ─── depends on D7.5 (combat stats)
  ↓
D9.1.3 (positioning)
  ↓
  ┌─────────────────────────────┐
  │ combat round loop (orchestrator) │
  └─────────────────────────────┘
       ↓              ↓              ↓
  D9.2.* (targeting)  D9.3.* (damage)  D9.4.* (specials)
       ↓              ↓              ↓
       └──────────────┴──────────────┘
                      ↓
              D9.5.* (retreat)
                      ↓
              D9.6.* (outcome)
```

The **combat round loop** is not a chunk — it's the orchestrator. It's small (~30 lines of Quint + ~30 lines of TS) and we treat it as scaffolding inside D9 itself.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 (Core Types) | `Fleet`, `Ship`, `ShipDesign`, `CombatStats` | D1.5, D1.10 |
| D7 (Ship Design) | Combat stat computation | D7.5 |
| D8 (Fleet & Movement) | Arrival at star triggers combat | D8.4 |
| D13 (AI) | AI submits combat orders (auto-resolve vs. manual) | D13.x |

D9 has **no dependency** on UI, persistence, or other domain sections.

## Quint-spec-sized leaves (the actual implementation units)

After the above sub-decomposition, each leaf is small enough to spec independently. Some leaves will be combined into a single Quint file because they're tightly coupled (e.g., D9.3.1 + D9.3.2 are the same combat round). Others will be separate.

**Proposed Quint files for D9**:

1. `specs/combat/spaceCombat.qnt` — top-level module: state, `step`, validation (D9.1.1), initiative (D9.1.2), positioning (D9.1.3), and orchestrator.
2. `specs/combat/targeting.qnt` — D9.2.* (targeting policies).
3. `specs/combat/damage.qnt` — D9.3.* (to-hit, damage, shields, armor, crits).
4. `specs/combat/specials.qnt` — D9.4.* (specials).
5. `specs/combat/retreat.qnt` — D9.5.* (retreat).
6. `specs/combat/outcome.qnt` — D9.6.* (survivors, XP, scrap).

Each file ≈ 100–300 lines of Quint with tests. Each maps 1:1 to a TypeScript module in `src/domain/combat/`.

## What "simple v1" looks like for D9

For v1 we can ship simpler behavior:

- **D9.1**: simple "left vs. right" positioning, no terrain. Per-ship initiative (v1 simplification).
- **D9.2**: greedy targeting — each ship shoots the nearest available target of any size. Stance is **not** consumed in v1 (reserved for v2).
- **D9.3**: flat damage formula — `max(0, attack - defense) * weaponDamage`.
- **D9.4**: shields reduce damage by a flat amount per round; ECM adds to defense; no point defense.
- **D9.5**: each side gets one chance to retreat at end of round; pursued fleet takes one volley.
- **D9.6.1–2**: survivors + HP tracking.
- **D9.6.3** (XP), **D9.6.4** (scrap): deferred to v2. Chunks defined, no v1 effect.
- **D9.6.5**: emit `BattleResolvedEvent` at combat end for end-game stats.

This is a faithful v1: the *flow* matches the original, the *outcomes* are reasonable, the *complexity* is bounded.

For v2 experiments, we can swap in richer formulas without touching callers — the `resolveSpaceCombat` signature stays the same.

## Resolved decisions for D9

- **Auto-resolve vs. tactical**: both ship in v1. Auto-resolve is part of D9 (the same `resolveSpaceCombat` function feeds both auto and tactical paths; P10 only adds the visual playback layer). AI-vs-AI combat always auto-resolves. Human players pick per-battle from P10.
- **Side composition**: combat is always 2-sided. Fleets owned by the same player arriving at the same star auto-merge into one fleet before combat starts (this is D8.4's responsibility — D8.4 produces one fleet per player-side at the star). When 3+ player-sides contest a star in the same turn, the pairing is resolved by randomly shuffling sides and pairing adjacent ones.
- **Boarding**: out of v1. Reserved as a possible D9.7.
- **Stellar converters / planet busters**: out of v1. Optional v2 chunks.
- **Initiative is per-ship** in v1 (sorted by computer, ties by hull size). Per-stack initiative is v2.
- **`CombatStats.speed` is cosmetic in v1** — it feeds the fleet speed display in P3/P6 but does not affect initiative (computer stat does), movement (D8.2 uses `Hull.baseWarpRange`), or combat outcomes. v2 may reintroduce it for retreat sequencing.
- **Stance is reserved data** — D9 does not read it in v1; D9.2 targeting AI consumes it in v2.
- **D9.6.3 (XP) and D9.6.4 (scrap)** deferred to v2. Chunks defined for spec completeness; no v1 effect.
- **`BattleResolvedEvent`** is emitted by D9.6 at combat end with `{star, winner: Option<FleetId>, loser: Option<FleetId>, outcome: Won | Drawn, turn}` (D9.6.5; D1 event-kind index); consumed by D14.5 for end-game stats and by P10 for UI feedback. `Option` types let D9 represent draws (both sides destroyed or `COMBAT_MAX_ROUNDS` exceeded) without a sentinel value.

## Next step

Open questions are resolved (see above). Write `specs/combat/spaceCombat.qnt` and the dependent files (`targeting.qnt`, `damage.qnt`, `specials.qnt`, `retreat.qnt`, `outcome.qnt`). Then implement `src/domain/combat/` to match.

Note: `sidePairing` (the random shuffle + pairing for 3+ sides) belongs in **D9.1 (combat setup)** as a small extra sub-chunk, since it is part of producing the two sides that feed the rest of the resolver.
