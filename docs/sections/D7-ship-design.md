# Section D7 — Ship Design & Combat Stats

D7 defines the things you put *in* a ship (hulls, weapons, specials) and the function that turns those components into the combat stats D9 reads. This is where players spend most of their tactical time — designing fleets — so v1 needs enough variety to be interesting without becoming a cataloging slog.

## Section contract

> **Ship Design & Combat Stats**: given a `ShipDesign` (hull + slots + weapons + specials), validate it and compute the `CombatStats` used by D9. D7 owns ship component catalogs (hulls, weapons, specials) and the pure functions over them.

**Owns**:
- Hull, weapon, and special catalogs (data + prereq lookups).
- Structural validation of `ShipDesign` (slot count, hull size compatibility, component space).
- Pure combat stat computation from a `ShipDesign` (no race or tech state).
- The `CombatStats` record consumed by D9.

**Does not own**:
- Tech-prereq enforcement (deferred to D5/P6; see "Resolved decisions").
- Per-instance combat modifiers from race traits or tech levels (applied at consumer — D9 or D8 wrapper).
- Production / build queues (D5.7 owns that).
- Tactical combat itself (D9 reads `CombatStats` but does no design work).
- Ship UI / P6 display (presentation only reads from D7).

## Top-level chunks

Five top-level chunks, lifted directly from `DECOMPOSITION.md`:

- **D7.1 Hull catalog** — hull sizes, slot layouts, base stats.
- **D7.2 Weapon catalog** — beam, missile, torpedo, fighter, bomber with damage/range/shots.
- **D7.3 Specials catalog** — shields, armor, ECM, etc.
- **D7.4 Design validation** — rules of placement (e.g., small weapons can't go on huge hulls).
- **D7.5 Stat computation** — attack/defense/computer/speed/hp from components.

## Recursive decomposition

### D7.1 → Hull catalog

Pure data: a `Map<HullId, Hull>` where each hull is:

```
type Hull = {
  id: HullId,
  name: string,
  size: HullSize,                // Small, Medium, Large, Huge
  slotCount: int,
  baseHp: int,
  baseSpeed: int,
  baseSpace: int,                // total component space available
  baseCost: int,                 // production cost in industry
  prereqTech: Option<TechId>,    // hulls unlocked by tech
}
```

v1 catalog (~10 hulls, MoO-style):

| Hull | Size | Slots | Base HP | Base Speed | Notes |
|---|---|---|---|---|---|
| Fighter | Small | 1 | 5 | 30 | fast, fragile |
| Bomber | Small | 2 | 8 | 25 | carries bombs against big ships |
| Destroyer | Medium | 4 | 20 | 20 | versatile |
| Cruiser | Medium | 6 | 35 | 18 | workhorse |
| Battleship | Large | 8 | 60 | 14 | heavy combat |
| Dreadnought | Large | 10 | 100 | 12 | battleship+ |
| Titan | Huge | 14 | 180 | 10 | very rare |
| Transport | Medium | 0 | 25 | 18 | carries troops; no weapons |
| Colony Ship | Medium | 0 | 25 | 16 | has Colony Module slot |
| Doomship | Huge | 18 | 300 | 8 | ultra-endgame |

(Exact numbers are placeholders for now; v1 spec will pin them after a few playtests.)

### D7.2 → Weapon catalog

```
type Weapon = {
  id: WeaponId,
  kind: WeaponKind,              // Beam, Missile, Torpedo, Fighter, Bomber
  name: string,
  damage: int,
  range: int,                    // parsecs or tactical hexes; same unit throughout
  shots: int,                    // shots per combat round
  accuracy: int,                 // flat bonus to to-hit
  spaceCost: int,                // design slots consumed
  compatibleSizes: Set<HullSize>,// which hull sizes can mount this weapon
  prereqTech: Option<TechId>,
}
```

v1 catalog (~10 weapons), grouped by kind:

- **Beam** (short range, infinite shots-per-battle but limited per round by `shots`):
  Laser, Ion Cannon, Phaser, Disruptor, Death Ray.
- **Missile** (longer range, finite ammo):
  Missile, Heavy Missile, Multi-Warhead Missile.
- **Torpedo** (slow, very long range, high damage):
  Torpedo.
- **Fighter** (spawned ships; treated as a weapon for placement):
  Fighter.
- **Bomber** (spawned ships; bombs capital ships):
  Bomber.

### D7.3 → Specials catalog

```
type Special = {
  id: SpecialId,
  kind: SpecialKind,              // Shield, Armor, ECM, PointDefense, HyperDrive, ColonyModule, ...
  name: string,
  effect: SpecialEffect,         // ADT, value depends on kind
  spaceCost: int,                // design slots consumed
  maxInstances: int,             // 1 for most; some (e.g., Shield) stack
  prereqTech: Option<TechId>,
}
```

v1 has **four functional special kinds** plus placeholder entries for the rest:

| Kind | v1 effect |
|---|---|
| Shield | Absorbs up to N damage per round, then fails. Stackable up to N stacks. |
| Armor | Reduces all incoming damage by N%. |
| ECM | Flat +N to defense (reduces to-hit chance). |
| ColonyModule | Allows the ship to colonize an uninhabited planet. Required on Colony Ship hull. |

**Placeholder specials** (in catalog, but no v1 effect yet): PointDefense, HyperDrive, FuelTank, BattleScanner, DamageControl, AutomatedRepair. Their `effect` field is `NoEffect` in v1; v2 turns on real behavior without changing the catalog shape.

This is a deliberate choice — **the catalog is forward-compatible**. v2 enables more specials without touching the catalog or any consumer; only the stat / combat resolvers grow.

### D7.4 → Design validation

Pure function: `validateDesign(design, hull, weapons, specials) -> Result<Unit, List<DesignError>>`.

Runs **structural** checks only. Tech-prereq enforcement is *not* D7's job (see "Resolved decisions").

Validation rules:

1. **Hull exists** in the catalog.
2. **Slot count** matches hull's `slotCount` (no more, no less).
3. **Each component fits its slot** — `slot.component.spaceCost <= slot.space`.
4. **Hull-size compatibility** — each weapon's `compatibleSizes` includes the hull's size.
5. **Component existence** — every `WeaponId` and `SpecialId` referenced exists in the catalog.
6. **Single-instance specials** — special kinds with `maxInstances == 1` aren't installed more than once.
7. **Hull requires no weapons** — Transport and Colony Ship hulls forbid weapons (sanity check; the catalog's `slotCount=0` already implies this).

Errors form an ADT:

```
type DesignError =
  | UnknownHull(HullId)
  | SlotCountMismatch(expected: int, actual: int)
  | ComponentTooLarge(slot: int, available: int, required: int)
  | WeaponTooLargeForHull(weaponId: WeaponId, hullSize: HullSize)
  | UnknownComponent(componentId: ComponentId)
  | DuplicateSpecial(specialId: SpecialId)
  | WeaponsOnCivilianHull(hullId: HullId)
```

`validateDesign` returns the full error list (not just the first), so a UI can show all problems at once.

### D7.5 → Stat computation

Pure function: `computeCombatStats(design, hull, weapons, specials) -> CombatStats` (only callable on designs that pass `validateDesign`).

```
type CombatStats = {
  attack: int,        // sum of (weapon.damage * weapon.shots) + accuracy bonus
  defense: int,       // base + ECM + armor-pct-equivalent + computer-bonus
  computer: int,      // base 1 + scanner bonuses (v1: always 1)
  speed: int,         // hull.baseSpeed (v1; v2 may add drive specials)
  hp: int,            // hull.baseHp + armor HP bonus
  shieldCapacity: int,// 0 if no shields; sum of shield capacities
  space: int,         // hull.baseSpace - used
  cost: int,          // hull.baseCost + sum(weapon.cost) + sum(special.cost)
  range: int,         // base 1; HyperDrive in v2
}
```

Computation rules (v1):

- `attack = Σ weapon.damage × weapon.shots + Σ weapon.accuracy`
- `defense = 1 + Σ special.effect.ecmBonus` (no other sources in v1)
- `computer = 1` (constant in v1; v2 reads from BattleScanner)
- `speed = hull.baseSpeed` (v1; HyperDrive in v2)
- `hp = hull.baseHp + armor.hpBonus` (where each Armor special adds X HP)
- `shieldCapacity = Σ shield.capacity` (across all Shield specials)
- `space = hull.baseSpace - Σ component.spaceCost`
- `cost = hull.baseCost + Σ weapon.cost + Σ special.cost`
- `range = 1` (no HyperDrive in v1; v2 reads from installed HyperDrive specials)

Each modifier is small and explicit. The function is one fold over the components.

**Per-instance modifiers** (race traits, owner tech level) are NOT applied here. They are applied at the consumer (D9 reads `CombatStats` and the per-fleet state, then combines them). This keeps D7.5 pure per-design and trivially testable.

## Dependency graph (within D7)

```
D7.1 (hulls) ────────────────────────────────┐
D7.2 (weapons) ──────────────────────────────┤
D7.3 (specials) ─────────────────────────────┤
  ↓                                           │
D7.4 (validation) ── reads D7.1, D7.2, D7.3  │
  ↓                                           │
D7.5 (stat computation) ── reads D7.1-D7.4   ┘
```

Linear and acyclic. D7.4 and D7.5 both depend on D7.1-D7.3; D7.5 additionally depends on D7.4 (assumes the design passed validation).

## Cross-section dependencies

| Depends on | What we need | Where it lives |
|---|---|---|
| D1 Core Types | `Hull`, `Weapon`, `Special`, `ShipDesign`, `CombatStats`, `HullSize`, `WeaponKind`, `SpecialKind`, `HullId`, `WeaponId`, `SpecialId`, `ComponentId` | D1.1, D1.2 |
| D3 Races & Traits | (none directly) — D7 doesn't import from D3; per-race modifiers are applied at the consumer | D3 |

D7 has no other dependencies. Note that D7 **does not import from D6 (Research)** either: prereq fields exist on Hull/Weapon/Special records, but D7.4 doesn't enforce them.

| Section | What it imports from D7 |
|---|---|
| D5 Economy | `Hull.baseCost` for ship production cost; full `CombatStats` for fleet-strength display |
| D8 Fleet & Movement | `CombatStats.speed`, `CombatStats.range` for fleet movement |
| D9 Space Combat | `CombatStats` for every shot resolution |
| D10 Ground Combat | (indirectly) — troop transport uses Transport hull |
| D11 Diplomacy | (indirectly) — trade offers can include ships |
| D13 AI | Reads hull/weapon/special catalogs to design ships |
| P6 Ship Design | All of D7 — UI mirrors the catalogs and validation |
| P10 Tactical Combat | `CombatStats` for HUD display |

D7 is imported by every system that touches ships. It's the third-most-imported section (after D1 and D3).

## Quint-spec-sized leaves (the actual implementation units)

Five Quint files, no orchestrator (consumers import what they need):

| Quint file | Implements | TS module | Approx. lines |
|---|---|---|---|
| `specs/ships/hulls.qnt` | D7.1 | `src/domain/ships/hulls.ts` | ~120 (mostly catalog data) |
| `specs/ships/weapons.qnt` | D7.2 | `src/domain/ships/weapons.ts` | ~150 (mostly catalog data) |
| `specs/ships/specials.qnt` | D7.3 | `src/domain/ships/specials.ts` | ~150 (catalog + `SpecialEffect` ADT) |
| `specs/ships/designValidation.qnt` | D7.4 | `src/domain/ships/designValidation.ts` | ~150 |
| `specs/ships/combatStats.qnt` | D7.5 | `src/domain/ships/combatStats.ts` | ~120 |

Total ~690 lines of Quint. The catalogs are the data-heavy part (~10 hulls × ~10 lines, ~10 weapons × ~12 lines, ~10 specials × ~12 lines = ~340 lines of pure data). Logic is in the other ~350 lines.

No top-level `shipDesign.qnt` orchestrator is needed — D7.4 and D7.5 are independent entry points, and consumers import them directly.

## What "simple v1" looks like for D7

- **~10 hulls, ~10 weapons, ~10 specials** (MoO-scale). Enough variety to be interesting; small enough to balance.
- **4 functional special kinds** (Shield, Armor, ECM, ColonyModule). The rest are placeholder catalog entries with `NoEffect`. v2 turns them on.
- **No tech-prereq enforcement in D7.4.** Structural only. Prereq checks happen at build time (D5) and design UI (P6).
- **No per-race or per-tech modifiers in D7.5.** Pure per-design. Consumers apply modifiers.
- **No ship-XP / ship-upgrade mechanics in v1.** Each design is a fixed shape; experience from combat (D9.6) is tracked but doesn't change ship stats in v1. (v2 adds "veteran" tiers.)
- **No "fitting" complexity.** A design is `Hull + N Slots`, each slot filled or empty. No sub-slots, no modular internals.

## Resolved decisions for D7

- **Tech-prereq enforcement lives in D5/P6, not D7.4.** D7.4 is purely structural.
- **Per-instance modifiers (race traits, owner tech) are applied at the consumer, not in D7.5.** Keeps D7.5 pure per-design.
- **Specials catalog includes placeholder entries** for v2 specials. Forward-compatible.
- **Hulls are universal**, not race-specific. (Per D3.1, no race-specific hulls in v1.)
- **`CombatStats` is a fixed record shape.** New stats (e.g., `cloakDetection`) added as new fields when v2 needs them; old consumers ignore them.
- **Validation returns a full error list**, not just the first error.

## Open questions for D7

- **Exact stat numbers for v1 catalog** (hull HP/speed/slots, weapon damage/range/shots, etc.). Propose placeholders now; pin during Quint spec phase. **Default to MoO values** for v1.
- **Should `computeCombatStats` validate first, or assume valid input?** Propose: assumes valid input. Validation is a separate call; combining them couples concerns. Caller can do `if (validateDesign(d).isOk) computeCombatStats(d)`.
- **Transport hull weapons check**: MoO let transports mount a small number of weapons. v1: transports have `slotCount = 0` and forbid weapons. If we want armed transports in v1, the hull's `slotCount` would be 1–2 and rule #7 in D7.4 would change. **Default: unarmed transports** (matches v1 simplicity).
- **Colony Ship specialization**: is Colony Ship its own hull, or a regular hull with a required Colony Module? MoO: separate hull. v1: separate hull with `slotCount = 0` and a required Colony Module slot. Decision: separate hull, slot is implicit.

No open questions block starting the D7 Quint spec.

## Next step

Per [`ROADMAP.md`](../ROADMAP.md), the next commit is **D8 Fleet & Movement** (the last D-section D9 depends on).

D7's Quint spec can be written **now** without waiting for D8/D9 — D7's only dependency is D1 (done).
