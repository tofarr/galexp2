# Section D3 — Races & Traits

D3 defines the ten playable races, the trait system that gives them mechanical personality, and the rules that turn "abstract races plus a galaxy" into "each race owns a homeworld."

## Section contract

> **Races & Traits**: defines the 10 races and how their traits modify game rules. Pure data + small modifier functions. D3.5 (added when D2 was decomposed) takes the candidate-homeworld list produced by D2 and assigns each race a homeworld.

**Owns**:
- The race catalog: 10 races with names, traits, portraits, homeworld preferences.
- The trait modifier system: a small set of `Modifier` values per trait, with a uniform apply API.
- Race-to-race diplomacy dispositions (starting relations).
- Race-specific tech affinities (per-tree research bonuses).
- Homeworld assignment from D2's candidate pool.

**Does not own**:
- Galaxy generation (D2) — D3 only consumes the candidate list.
- AI decision-making (D13) — D3 provides `aiPersonalityHints` but doesn't run AI.
- Government types (deferred to v2 — see "Open questions").
- Custom race designer (deferred to v2).
- Race-specific ship hulls (deferred to v2 — v1 uses universal hulls).

## Top-level chunks

The DECOMPOSITION.md entry lists D3.1–D3.4. We add **D3.5 (homeworld assignment)** to receive the candidate list D2.6 produces:

- **D3.1 Race catalog** — the 10 races, base stats, portraits (asset ref).
- **D3.2 Trait modifier system** — how a `Race.traits[]` array changes formulas elsewhere.
- **D3.3 Race-specific tech affinities** — e.g., Meklons get +1 to Computers.
- **D3.4 Diplomacy disposition** — race-to-race starting relations and proclivities.
- **D3.5 Homeworld assignment** — pick one D2 candidate per race, using race homeworld preferences.

## Recursive decomposition

Each top-level chunk is already small enough. The sub-decomposition is mostly about clarifying the trait modifier interface, because that's what every consumer (D5/D9/D10/D11) will read.

### D3.1 → Race catalog

Ten hardcoded races from the original MoO:

| Race | Traits | Homeworld preference | Tech affinity |
|---|---|---|---|
| Humans | Skilled, Versatile | Terrestrial | none |
| Alkari | Flight, Warrior | Arid | Propulsion |
| Bulrathi | Militaristic, Subterranean | Barren | Construction |
| Darloks | Stealthy, Subterranean | Arid/Tundra | Computer |
| Gnolams | Cybernetic, Tolerant | Terrestrial | none |
| Klackons | Repulsive, Industrial | Volcanic | Construction |
| Meklons | Cybernetic, Creative | Oceanic | Computer |
| Mrrshan | Warrior, Honorbound | Tundra | Weapons |
| Psilons | Telepathic, Erudite | Oceanic | Weapons +1 (extra) |
| Sakkra | Subterranean, Lithovore | Volcanic | none |

(Trait and affinity names are placeholders; final names come from the original MoO names when we get to the Quint spec. The structure — each race has 2 traits + homeworld type + optional affinity — is what matters here.)

Each race entry in the catalog is a `Race` record (defined in D1.2):

```
type Race = {
  id: RaceId,
  name: string,
  traits: Set<Trait>,
  homeworldPreference: PlanetType,
  techAffinities: Map<TechTree, int>,   // bonus research points per turn
  aiPersonalityHints: Set<AiPersonalityTrait>,   // soft defaults; D13.1 maps these to its own StrategyKind (Aggressive | Builder | Technologist | Diplomat | Balanced)
  portraitRef: string,                  // asset reference, not loaded by D3
}
```

The catalog itself is a `Map<RaceId, Race>` constant in the Quint spec. v1 uses the 10 hardcoded races only.

### D3.2 → Trait modifier system

This is the most-touched D3 chunk because it exports a uniform API that every formula in D5/D9/D10/D11 reads.

A `Modifier` is a typed effect on a specific formula. The full enumeration for v1:

```
type Modifier =
  // Economy (D5) — multipliers
  | FoodProduction(float)
  | IndustryProduction(float)
  | ResearchProduction(float)
  | TradeIncome(float)

  // Combat (D9, D10) — flat bonuses
  | ShipAttack(int)              // +N to CombatStats.attack
  | ShipDefense(int)             // +N to CombatStats.defense
  | ShipSpeed(int)               // +N to CombatStats.speed
  | GroundAttack(int)            // +N to ground-combat attacker roll (D10.1)
  | GroundDefense(int)           // +N to ground-combat defender roll (D10.1)

  // Diplomacy (D11)
  | Diplomacy(int)               // flat bonus to relation changes

  // Espionage (D12)
  | SpyDetection(float)          // multiplier applied to **defending** player's detection chance — the player with the trait detects spies on their planets more easily. A value < 1.0 makes the owner *worse* at detecting; 1.0 is neutral. (The trait's phrasing as "harder to detect own spies" is misleading — see the table footnote for `Stealthy`.)
  | CounterEspionage(int)        // flat bonus to counter-espionage skill (D12.5)

  // Population
  | GrowthMod(float)             // multiplier on population growth rate (D5.1)

  // Morale
  | Morale(int)                  // flat bonus to base morale (D5.6)

  // Pillage / conquest (D10.4)
  | Pillage(float)               // addend to pillage population-loss fraction (e.g., +0.05 = +5%)

  // Research efficiency (D6.3)
  | ResearchEfficiency(float)    // multiplier on grossResearch → accumulated research (e.g., Erudite 1.20)

  // Open for v2 additions (e.g., StealthBonus, TradeDetection, …)
  | ...                           // extensible; new modifiers added as needed
```

**Important distinction**: race *modifiers* (this list) are distinct from *building effects* (D1.2's `BuildingEffect` ADT) and *planet properties* (e.g., `Planet.garrison`, `Planet.specials`). Consumers pattern-match on whichever ADT carries the data they need. For example, "food production" is a race multiplier from D3.2's `FoodProduction` *plus* a sum of building `FoodOutput` effects from D1.2 — never both in one place.

**Trait-flag pattern**: Some traits carry a `Modifier` set (the common case) *and* an additional data-driven check on the trait itself. For example:

- `Subterranean` carries `Modifier.{Morale(1), GroundDefense(1)}` *and* the trait value is checked directly by D10.3 (`hasTrait(state, defender, Subterranean)`) to skip the surrender check.
- `Lithovore` carries `Modifier.{GrowthMod(1.0)}` *and* is checked directly by D5.1 (`hasTrait(state, race, Lithovore)`) to skip starvation when food is negative.

This pattern (Modifier for numeric tuning, `hasTrait` for on/off behavioral switches) is intentional. v1 keeps it; v2 could move flag-style effects into a `TraitFlag` enum if the pattern proliferates.

Each `Trait` value has a fixed list of modifiers, defined once in the spec:

```
traitModifiers(Militaristic)   = { GroundAttack(1), GroundDefense(1), ShipAttack(1) }
traitModifiers(Toltec)         = { Diplomacy(2) }
traitModifiers(Cybernetic)     = { ResearchEfficiency(1.20), ShipDefense(1) }
traitModifiers(Subterranean)   = { Morale(1), GroundDefense(1) }        // D10.3 reads `GroundDefense` AND skips surrender check (data-driven via the trait; see D10.3)
traitModifiers(Lithovore)      = { GrowthMod(1.0) }                     // D5.1 reads "no food dependence" via separate flag
traitModifiers(Skilled)        = { ResearchEfficiency(1.10), IndustryProduction(1.10) }
traitModifiers(Versatile)      = { FoodProduction(1.10), IndustryProduction(1.10), ResearchEfficiency(1.10) }
traitModifiers(Flight)         = { ShipSpeed(1), ShipDefense(1) }
traitModifiers(Warrior)        = { ShipAttack(1), GroundAttack(1) }
traitModifiers(Stealthy)       = { SpyDetection(0.85) }                 // D12.5 reads this as: Stealthy owner's own spy detection chance (i.e., detecting *enemy* spies on their own planets) is *reduced* by 15%. NOTE: this is the *opposite* of the natural reading of "easier to spot others' spies" — the original D3.2 wording was inverted; v1 keeps the inverted semantic for behavioral consistency with MoO. D12.5 documents the meaning.
traitModifiers(Repulsive)      = { Diplomacy(-2) }
traitModifiers(Industrial)     = { IndustryProduction(1.20) }
traitModifiers(Creative)       = { ResearchEfficiency(1.15) }
traitModifiers(Honorbound)     = { Morale(2), Pillage(-0.05) }          // Pillage less aggressively
traitModifiers(Telepathic)     = { CounterEspionage(2) }
traitModifiers(Erudite)        = { ResearchEfficiency(1.20) }
traitModifiers(Tolerant)       = { Diplomacy(1) }                        // gain positive relations 1.0× faster with non-aligned races; was missing from prior drafts
// (16 traits total — see D1.1 for the canonical catalog)
```

The D3.1 race catalog uses trait names that must each have an entry in this
table. New traits can be added by extending the ADT and this mapping;
existing modifiers don't need to change.

Consumers sum the modifiers from all of a race's traits:

```
totalModifiers(race) = fold(mergeModifier, traitModifiers(t) for t in race.traits)
```

And then apply them where the formula lives:

```
// in D5 (economy)
food = baseFood * totalModifiers(race).foodProduction

// in D12.5 (espionage detection) — applies to the *defending* player's modifier
detectionChance = baseDetectionChance * totalModifiers(defendingPlayer).spyDetection
```

Modeling decisions (locked in for v1):
- Modifiers are **summed, not multiplied**. Two `+1 ShipAttack` traits give `+2`, not `+1×1=+1`. Simpler reasoning; matches MoO's behavior.
- Each trait is a **named constant**, not a numerical code. Tests read `traitModifiers(Subterranean)` directly.
- Modifiers form an **open ADT**. New modifier kinds are added as we discover we need them; existing consumers don't need to change.
- The `Modifier` ADT is defined in D3.2 and imported by every consumer. **D3.2 is the single source of truth** for what modifiers exist.
- `ColonizationSpeed` and `MaxColonies` are *not* in the v1 ADT — v1 has no consumer (colonization is a one-time D8.6 event, not a repeated rate). Re-add in v2 if a colonization-rate mechanic is introduced. `HyperExpansion` is dropped for the same reason.
- v1 ties do not have a `Surrender` flag — `Subterranean`'s "no surrender until fully eliminated" is data-driven via D10.3 reading the `Subterranean` trait directly (alongside its `GroundDefense(1)` modifier). See D10.3.

### D3.3 → Race-specific tech affinities

Each race gets a per-tree research bonus. Implementation:

```
techAffinity(race: Race, tree: TechTree): int
```

Defaults to 0 for unlisted trees. Examples (from the catalog table):

```
techAffinity(Meklons,  Computer)    = +2
techAffinity(Psilons,  Weapons)     = +2
techAffinity(Psilons,  Shields)     = +1
techAffinity(Bulrathi, Construction) = +2
```

This is consumed by D6 (research) when computing per-turn research points.

A possible v2 simplification: collapse D3.3 into D3.2 by treating affinities as `ResearchProduction(tree, +N)` modifiers. We don't do that in v1 because affinities are race-level (one per race per tree), not trait-level (any race can theoretically pick the Cybernetic trait), and the consumption site (D6) is cleaner with a direct function.

### D3.4 → Diplomacy disposition

Two pieces:

1. **Per-race pair starting relation**. Each race has a `startingRelation(raceA, raceB): int` value in `[-20, +20]`. MoO defaults: Meklons and Darloks start unfriendly with everyone; Psilons start friendly; most pairs start neutral.

2. **Race-level diplomacy proclivity** (an enum). Used by the AI to weight its decisions in D13:
   - `Aggressive` — prefers war, demands tribute.
   - `Diplomatic` — prefers treaties, votes for council.
   - `Expansionist` — colonizes aggressively.
   - `Technologist` — trades for tech, less war.
   - `Honorable` — keeps treaties strictly; punishes betrayal heavily.
   - `Ruthless` — breaks treaties when convenient.

Each race has 1–2 proclivities. Combined with the per-player AI personality (chosen at game start), this gives a 2-axis personality system.

### D3.5 → Homeworld assignment

Given:
- A `Galaxy` from D2 with `candidates: PlanetId[]` tagged as candidate homeworlds.
- A list of `Race` to place (one per player).

Produce: a `Map<RaceId, PlanetId>` mapping each race to one candidate, where each candidate is used at most once.

Algorithm:
1. **Score each (race, candidate) pair** with a preference function:
   ```
   score(race, planet) =
       matchScore(race.homeworldPreference, planet.type)     // 0/1/2 (perfect/close/wrong)
     + sizeBonus(planet.size)                               // 0..2
     + richnessBonus(planet.richness)                       // 0..2
   ```
   `matchScore` is a small lookup table; e.g., Humans prefer Terrestrial (perfect), accept Oceanic or Arid (close), reject others (wrong).
2. **Greedy assignment**: sort races by "most picky" first (smallest candidate pool with positive scores), and assign each to its highest-scoring unused candidate.
3. **Fallback**: if a race has zero positive-scoring candidates (very unlucky galaxy), assign to the candidate with the highest non-negative score. This guarantees every race gets a homeworld.

Output: `Map<RaceId, PlanetId>` plus a list of "warnings" (e.g., `"Bulrathi placed on a Terrestrial planet (no Volcanic candidates)"`) that become `Event` entries in `GameState`.

D3.5 is the only D3 chunk that touches D2. D4 (Turn Cycle init) calls D3.5 at game start.

## Dependency graph (within D3)

```
D3.1 (catalog)
  ↓
D3.2 (traits)        ← reads traits from D3.1's race records
D3.3 (affinities)    ← reads from D3.1
D3.4 (dispositions)  ← reads from D3.1
  ↓
D3.5 (homeworld)     ← consumes D3.1 (preferences) and D2.6 (candidates)
```

D3.1 is the root of D3. D3.2/D3.3/D3.4 are independent leaves that all read from D3.1. D3.5 sits on top of D3.1 and D2.

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Race`, `Trait`, `PlanetType`, `Planet`, `PlanetId`, `RaceId`, `AiPersonalityTrait`, `Modifier` | D1.1, D1.2 |
| D2 Galaxy Generation | `Galaxy`, `Planet.candidateHomeworld`, `PlanetId` candidates list | D2.6 |

D3 has no other dependencies.

| Section | What it imports from D3 |
|---|---|
| D4 Turn Cycle | `Race`, `homeworldAssignment()` — used during game init |
| D5 Economy | `totalModifiers(race)` for production modifiers |
| D6 Research | `techAffinity(race, tree)` |
| D9 Space Combat | `totalModifiers(race)` for ship attack/defense/speed |
| D10 Ground Combat | `totalModifiers(race)` for ground combat |
| D11 Diplomacy | `startingRelation(raceA, raceB)` for initial state |
| D13 AI | `aiPersonalityHints` for default AI behavior |
| P2 Race selection | `Race` catalog (read-only) |
| P6 Ship design | `race.limits` (if any) — v1: none |
| P7 Diplomacy | `Race` portrait + relation |
| P12 Council | `Race` portrait + vote weight (optional) |

D3 is the second-most-imported section after D1.

## Quint-spec-sized leaves (the actual implementation units)

Four Quint files, plus the catalog data lives in one of them:

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/races/raceCatalog.qnt` | D3.1 + D3.3 + D3.4 | `src/domain/races/raceCatalog.ts` | ~180 (mostly catalog data) |
| `specs/races/traits.qnt` | D3.2 (modifier system) | `src/domain/races/traits.ts` | ~200 |
| `specs/races/homeworld.qnt` | D3.5 (homeworld assignment) | `src/domain/races/homeworld.ts` | ~120 |
| `specs/races/races.qnt` | top-level public API: `totalModifiers`, `techAffinity`, `startingRelation` re-exports | `src/domain/races/index.ts` | ~50 |

Total ~550 lines of Quint. The catalog is the only chunk that's "data-heavy" (~100 lines for the 10 races); the rest is logic.

The TypeScript catalog is a JSON file (`src/domain/races/raceData.json`) loaded at module init time. Trait modifiers are TypeScript constants exported from `traits.ts`.

## What "simple v1" looks like for D3

- **Fixed 10 races**, no custom race designer.
- **Fixed trait list** (~16 traits, each race has 2). No point-buy, no random traits.
- **Summed modifiers**, not multiplied. Easier to reason about; matches MoO behavior.
- **Greedy homeworld assignment**, no Hungarian algorithm. MoO's assignment was greedy too; nothing to gain from optimal matching when races are roughly equally matched.
- **No government types**. MoO had government types (Democracy, etc.) with their own bonuses; v1 omits them entirely. Add as a separate chunk in v2.
- **No race-specific ship hulls**. MoO had a few race-specific hulls (e.g., the Gnolams' unique frigate). v1 uses universal hulls from D7. Add race-specific hulls in v2.

## Resolved decisions for D3

- **Modifiers are summed, not multiplied.** Simpler; matches MoO.
- **D3.5 owns homeworld assignment.** Per the D2 split; D2 produces candidates, D3 assigns.
- **Greedy assignment, with fallback to "best available."** Guarantees every race gets a homeworld.
- **No government types in v1.** Deferred.
- **No custom races in v1.** Deferred.
- **No race-specific hulls in v1.** Deferred.
- **Modifier ADT is open.** Adding new modifier kinds is a one-place change in D3.2.

## Open questions for D3

- **Trait list scope**: how many traits for v1? Original MoO had ~16. Proposing 16 for v1, with the option to slim down if we want a faster first playable. **Default to 16.** Decide on the exact list when we get to the Quint spec.
- **AI personality: race hint vs. player choice.** Both. Race gives a default; player picks the actual AI personality at game start. Already locked in D3.4 above.
- **Diplomacy disposition values**: what are the exact per-pair starting relations for the 10 races? Propose using MoO defaults as a starting point and adjusting during playtesting. **Defer to spec phase.**

No open questions block starting the D3 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D7 Ship Design & Combat Stats** (D4 is skipped for now because it's small and central — better to do it after the sections it calls into are decomposed).

D3's Quint spec can be written **now** without waiting for D4/D5/etc. — D3's only dependencies are D1 (done) and D2 (done).
