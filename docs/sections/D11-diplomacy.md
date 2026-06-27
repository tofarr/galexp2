# Section D11 — Diplomacy

D11 is the **state machine for diplomacy** between empires. Players (human and AI) submit offers — treaties, trades, peace, war — and the resolver evaluates them, applies accepted changes, and emits events. D11 also owns trade routes, the Galactic Council, and the peace/war declarations that bind empires together or push them apart.

## Section contract

> **Diplomacy**: state machine for diplomacy. Players and AI submit offers (trade, tech, treaties, war); the resolver produces new treaties and relation changes. Also owns ongoing trade routes, the Galactic Council election, and peace declarations.

**Owns**:
- Per-pair relation state (`Relation` record with score -100..+100 and modifiers).
- Treaty kinds and their effects on state.
- Offer evaluation (does this offer appeal given AI personality + current relations?).
- Treaty acceptance/rejection logic.
- Galactic Council vote accumulation and election.
- Trade route state and per-turn trade income computation.

**Does not own**:
- Tech trade primitives (D6.5 owns `canReceiveTech` / `receiveTech`; D11 just composes offers).
- AI decision-making (D13) — D13 decides what offers to make; D11 evaluates them.
- Combat (D9) — wars happen, but D11 just declares them; D9 resolves the actual battles.
- Production (D5) — D5.5 reads trade income; D11 owns the trade routes that produce it.

## Top-level chunks

Six top-level chunks from `DECOMPOSITION.md`:

- **D11.1 Relation state** — per-pair relation level (-100..+100) and modifiers.
- **D11.2 Treaty kinds** — peace, alliance, trade pact, NAP, etc.
- **D11.3 Offer evaluation** — does this offer appeal given the AI's personality and current relations?
- **D11.4 Treaty acceptance/rejection**.
- **D11.5 Council / Galactic Emperor** — vote accumulation, election.
- **D11.6 Trade** — ongoing per-turn trade income.

## Recursive decomposition

### D11.1 → Relation state

Per-pair state. `Relation` is symmetric (player A's relation with B = player B's relation with A):

```
type Relation = {
  playerA: PlayerId,
  playerB: PlayerId,
  score: int,                       // -100..+100
  treaties: Set<TreatyId>,          // active treaties between them
  tradeRoutes: List<TradeRouteId>,
  warState: WarState,               // AtPeace | AtWar(since: TurnId) | JustMadePeace(until: TurnId)
  lastContactTurn: int,
  // modifiers are computed, not stored — derived from treaties + history
}
```

**Score modifiers** (computed live from the relation's treaties and state):

```
relationScore(a, b) =
    base 0
  + treatiesScore(a, b)            // +5 per Trade Pact, +20 per Alliance, etc.
  - warWeariness(a, b)             // -10 if at war
  - recentBetrayal(a, b)           // -25 if broke a treaty in last 10 turns
  + tradeIncome(a, b) * 0.01       // small bonus for active trade
  + sharedEnemyBonus(a, b)         // +5 per shared enemy
  + raceCompat(a.race, b.race)     // some races like/dislike each other (D3)
```

The final score is `clamp(rawScore, -100, +100)`. Without the clamp, accumulated modifiers could push the score outside the documented range.

**Score drifts over time** (every turn): if `state.turn - relation.lastContactTurn > 10`, the score drifts toward 0 at 1 point per turn starting at the 11th turn. If at peace and score < 0, score improves by 1 per turn (peace heals).

Stored on `GameState.relations: Map<RelationKey, Relation>` where `RelationKey = (minPlayerId, maxPlayerId)` (canonicalized pair).

### D11.2 → Treaty kinds

`Treaty` is an ADT:

```
type Treaty =
  | NonAggressionPact    // can't attack each other
  | TradePact            // open trade routes; +5 relation
  | Alliance             // full military alliance; defend each other when attacked
  | PeaceTreaty          // ends a war (signed at end of hostilities)
  | Subjugation          // vassal state; pays tribute, follows suzerain's wars
  | ResearchAgreement    // share research progress (reduced cost for both)
```

Each treaty has effects:
- **NAP**: `warState = AtPeace`; if either breaks it (attacks), `recentBetrayal += 25`.
- **TradePact**: enables `TradeRoute`; +5 relation.
- **Alliance**: mutual defense obligation. If either is attacked, the other declares war on the aggressor automatically. +20 relation.
- **PeaceTreaty**: ends war; restores AtPeace; -25 relation if forced (conquest-driven).
- **Subjugation**: vassal is automatically set to `AtWar` with any suzerain's enemy; vassal cannot independently declare war; vassal pays 10% of gross income to suzerain each turn (applied as a maintenance line in D5.5). v2 could add suzerain diplomacy on behalf of vassals.
- **ResearchAgreement**: each player gets 50% of the other's *unspent* research. Transfer is applied as a maintenance line in D5.5 after D6.3 has deducted the cost of any tech-acquired this turn (so the total is conserved).

**Subjugation**: v1 includes basic vassal mechanics with the above mechanical effects. v2 could add suzerain diplomacy on behalf of vassals and diplomatic-relation rules for subjugated races.

### D11.3 → Offer evaluation

Pure function: `evaluateOffer(offer, recipient, recipientRace, aiPersonality, relation, ctx) -> OfferDecision`.

`Offer` types:
```
type Offer =
  | ProposeTreaty(from: PlayerId, to: PlayerId, treaty: Treaty)
  | DeclareWar(from: PlayerId, to: PlayerId)
  | ProposeTrade(from: PlayerId, to: PlayerId, give: OfferSide, receive: OfferSide)
  | CancelTreaty(from: PlayerId, treatyId: TreatyId)

type OfferSide =
  | Tech(TechId)
  | Money(int)
  | TreatyCancel(TreatyId)
  | Planet(PlanetId)
```

Decision logic:
1. **Relation filter**: if relation < -50, the AI rejects most offers automatically ("we will not negotiate with you").
2. **Personality filter**: each personality has weights for what they value.
   ```
   Pragmatic: { Money: +10, Tech: +5, Peace: +3, Conquest: -10 }
   Aggressive: { Money: -5, Tech: -5, Conquest: +15, Alliance: -10 }
   Diplomat: { Tech: +10, Peace: +10, Alliance: +5, Conquest: -15 }
   ```
3. **Value computation**: assign a value to `give` and `receive` based on recipient's perspective.
4. **Net value**: `receiveValue - giveValue + personalityAdjustment`.
5. **Accept if net value > 0** (with a small buffer like `> -5` for borderline offers).

Returns `OfferDecision.Accept` or `OfferDecision.Reject(reason: string)`.

### D11.4 → Treaty acceptance/rejection

When D11.3 returns `Accept`:
1. Create the treaty record.
2. Add to `relation.treaties`.
3. Apply treaty effects (relation score, trade routes, defense obligations).
4. Emit `TreatySignedEvent { from, to, treaty }`.

When `Reject`:
- Minor relation penalty (`-2`): "they rejected our offer, slight insult."
- Emit `OfferRejectedEvent { from, to, offer, reason }`.

For `DeclareWar`: unilateral; no acceptance needed. Just sets `warState = AtWar(since: turn)`, breaks all NAPs/Alliances, and emits `WarDeclaredEvent`.

### D11.5 → Council / Galactic Emperor

The Galactic Council meets every `COUNCIL_INTERVAL = 25` turns (from D1.1 constants). The council turn is detected as `state.turn % COUNCIL_INTERVAL == 0` (turn 25, 50, 75, …). On council turn:

1. Compute each player's `voteCount` based on `state.score[player]` — the derived per-player score field on `GameState` (D1.3, D14.4). The score is recomputed once per turn by D4 between economy and victory, so D11.5 simply reads the cached value:
   ```
   voteCount = state.score[player]
   ```
   This avoids a D11↔D14 circular dependency; both sections read from the same derived field.

2. Sum all votes; if anyone has `> 50%` of total, they're elected Galactic Emperor.

3. **Victory check**: if elected, emit `GalacticEmperorVictoryEvent { player }`. D14.3 picks this up and ends the game.

If no one has >50%, no winner this round; next council meets in 25 turns.

**Alliance voting**: in MoO, your allies' votes could count toward yours (with reduced weight). v1: simplified — only your own votes count. v2: alliance-weight.

**Voting at council**: in MoO, you could vote against someone (denounce them) or for yourself. v1: simplified — votes are positive (for yourself). v2: denouncement.

### D11.6 → Trade

Per-turn trade income computation:

```
tradeIncome(player) = Σ route.income
                     for each active trade route owned by player

TradeRoute {
  fromPlanet: PlanetId,
  toPlanet: PlanetId,
  income: int,                  // recomputed each turn
  active: bool,                 // false if either planet conquered/loses owner
}
```

Income formula:
```
income = baseTrade
         × techLevelBonus(player, route.toPlanet)   // +10% per Computer level
         × treatyBonus(relation.treaties)            // +20% with Trade Pact
         × distancePenalty(fromPlanet, toPlanet)    // -1% per parsec, capped at -50%
```

Trade requires:
- Both planets owned (not native/uncolonized).
- Player A owns one, Player B owns the other (or same player trades with self).
- A treaty allows trade (Trade Pact, Alliance, NAP — NAP allows trade in v1).

Trade routes are created via player command (`CreateTradeRoute(from, to)`); they persist across turns.

`TradeRoute.income` is **cached at end of turn** (D11.6 runs after D5.5 in the same step) so D5.5 in the *next* turn can read the cached value. This resolves the phase-order conflict: economy runs before diplomacy, but trade income is a diplomacy-owned computation. The cache is just `state.tradeRoutes[id].income = recomputed_value` written in D11.6.

If either endpoint planet changes ownership, the trade route is broken (becomes `active = false` and is removed at end-of-turn cleanup).

## Dependency graph (within D11)

```
D11.1 (relations) ────────────────────────────────────┐
  ↓                                                    │
D11.2 (treaties) ── reads D11.1, modifies relations     │
  ↓                                                    │
D11.3 (offer eval) ← reads D11.1, D11.2, D3 (race)     │
  ↓                                                    │
D11.4 (acceptance) ← reads D11.3 result, applies via D11.2
  ↓
D11.5 (council) ← reads D11.1 (relation) and the derived `state.score` field (D1.3, written by D14.4)
  ↓
D11.6 (trade) ← reads D11.1 (treaty), D6 (tech level)
```

Linear with one side-branch (D11.5).

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Player`, `Relation`, `Treaty`, `Offer`, `OfferSide`, `TradeRoute`, `PlayerId`, `TreatyId`, `WarState` | D1.2 |
| D3 Races & Traits | `raceCompat(a, b)` for relation modifiers, `aiPersonality(race)` (race-base personality) | D3.2, D3.4 |
| D6 Research | `techLevel(player, tree)` for trade income; `canReceiveTech`/`receiveTech` for trade offers | D6.5 |
| D13 AI | (no import) — D13 generates AI offers; D11 evaluates them | D13 |
| D14 Victory | (no import) — D11.5 reads the derived `state.score` field (D1.3) which D14 writes | D14.4 |

D11 has moderate imports from the domain sections.

| Section | What it imports from D11 |
|---|---|
| D4 Turn Cycle | Calls D11 once per turn (after espionage) |
| D5 Economy | Reads trade route income (D11.6) for net income |
| D8 Fleet & Movement | Reads `warState` to determine if fleet can attack a star |
| D9 Space Combat | Reads `relation.warState` for combat eligibility (only attack at war) |
| D14 Victory | Reads `GalacticEmperorVictoryEvent` for game-end |
| P7 Diplomacy | Reads/writes diplomatic state |
| P12 Council | Renders council election |

## Quint-spec-sized leaves (the actual implementation units)

Four Quint files:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/diplomacy/relationsAndTreaties.qnt` | D11.1 + D11.2 (state + treaty kinds) | `src/domain/diplomacy/relationsAndTreaties.ts` | ~200 |
| `specs/diplomacy/offers.qnt` | D11.3 + D11.4 (evaluation + acceptance) | `src/domain/diplomacy/offers.ts` | ~200 |
| `specs/diplomacy/council.qnt` | D11.5 (council election) | `src/domain/diplomacy/council.ts` | ~120 |
| `specs/diplomacy/trade.qnt` | D11.6 (trade routes) | `src/domain/diplomacy/trade.ts` | ~120 |

Total ~640 lines of Quint.

D11.1 and D11.2 share a file because treaty effects are how relations change. D11.3 and D11.4 share a file because evaluation and application are tightly coupled.

No top-level orchestrator — D4 calls each chunk in sequence: `applyRelationDrift → processOffers → runCouncilElection → applyTradeIncome`.

## What "simple v1" looks like for D11

- **Relation score drifts toward 0** without contact. No contact for 10 turns → drift by 1 per turn.
- **Wars end by peace treaty or conquest** — no surrender mechanics in v1 (D10 doesn't force treaties; players have to negotiate).
- **Council every 25 turns.** Election by simple majority.
- **Trade routes created manually** via player command (`CreateTradeRoute(from, to)`). No auto-trade.
- **Trade income is per-turn and per-route**, not a global "trade" slider.
- **Subjugation is one-way** (vassal → suzerain, no reversal mechanic in v1). v2 could add liberation.
- **No denouncement** at council. v2.
- **No shared-enemy bonus** in v1 (simplified from MoO). Just treaty bonuses and trade.

## Resolved decisions for D11

- **Relation score is `clamp(rawScore, -100, +100)`** after all modifiers.
- **Relation score drifts toward 0** without contact (after 10 turns of no contact, drift at 1 point/turn starting at turn 11). Cooling relations is realistic.
- **Wars end by peace treaty or total conquest.** No automatic surrender.
- **Council every 25 turns** (turn 25, 50, 75, …), simple majority election. Council vote count reads from the derived `state.score` field (D1.3) computed once per turn by D14.4.
- **Trade routes manual** — created by player command, not auto-generated.
- **Trade income cached on `TradeRoute.income`** by D11.6; D5.5 reads the previous turn's cached value.
- **Subjugation is mechanical in v1** — vassal auto-AtWar with suzerain's enemies, pays 10% tribute. No AI heuristic.
- **Subjugation is one-way in v1** (no liberation mechanic).
- **Trade Pact, NAP, Alliance all enable trade routes** (NAP enables at reduced income).
- **Treaty breakage (e.g., attacking during NAP) adds 25 to `recentBetrayal`**, decaying slowly.
- **Research agreement is a per-turn 50/50 split of unspent research**, applied as a maintenance line in D5.5 after D6.3.

## Open questions for D11

- **Council vote count formula**: in MoO it was based on population + tech + production. **Default v1: just `Player.score`** (D14 computes score). Could change formula later.
- **Alliance defense obligation**: if player A has Alliance with B and is attacked by C, does A auto-declare war on C? **Default v1: yes**, with a one-turn delay (player can choose to honor or break it).
- **Research agreement mechanics**: 50/50 split of research progress seems fair. **Default v1: 50%**.
- **Trade distance penalty**: -1% per parsec capped at -50%. **Default v1.** Closer routes = more income.
- **Subjugation tribute**: 10% of gross income. **Default v1.** Tune during playtesting.

No open questions block starting the D11 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D12 Espionage** (small focused section).

D11's Quint spec can be written **now** — D11's dependencies (D1, D3, D6, D14) are all done or about to be.