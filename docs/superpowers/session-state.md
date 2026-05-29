# CCG Session State

> Full context for resuming work. Kept in the project so it travels with the repo.

---

## What we're building

An online 1v1 collectible card game (Hearthstone-clone for initial scope). Features: ranked PvP matchmaking (MMR-based), spectatable game sessions, full accounts + card collection system, authoritative server (client is a pure event renderer). Target scale: 1k–10k concurrent players.

---

## Architecture (locked)

**Core Split** — two services + one client:

| Component | Stack | Role |
|---|---|---|
| Game Server | C# / .NET (ASP.NET Core) | Authoritative game state, WebSocket connections, action processing, event broadcasting |
| Platform API | TypeScript / Node.js | Auth, card collection, deck management, matchmaking, MMR |
| Client | Unity or Godot (TBD) | Renders events, sends player actions |

**Shared logic:** `CCG.GameLogic` — standalone C# class library (no engine/framework deps). Referenced by Game Server; optionally by Unity client.

**Storage:** PostgreSQL (persistent) + Redis (ephemeral queue/session state).

---

## Subsystem build order

| # | Subsystem | Service |
|---|---|---|
| 1 | Game Server | C# / .NET |
| 2 | Auth & Accounts | Platform API |
| 3 | Collection & Decks | Platform API |
| 4 | Matchmaking | Platform API |
| 5 | Client | Unity / Godot |

---

## Where we stopped

**Game Mechanics spec is COMPLETE and approved** — all 4 sections written to `docs/superpowers/specs/2026-05-26-game-mechanics.md` (data model, actions/events, engine architecture, action pipeline).

**Implementation plan is COMPLETE** — broken into 16 epics with ~80 tickets under `docs/superpowers/plans/2026-05-27-ccg-game-logic/`. Start at the README there; it has the full epic index, ticket outline, and a writing-progress tracker. Each ticket is a small, testable increment using the scenario-test methodology (build state → submit action(s) → assert ordered events + state).

**Implementation is NOT yet started** — we deferred kickoff in favour of a spec-refinement pass driven by the older Unity proof-of-concept at `C:\Users\vasil\UnityProjects\ccg`.

### Spec refinements applied in session 2026-05-28

1. **Card and Minion Identity subsection** added to spec §1 — establishes `definitionKey` as the only cross-entity link; `Card.id` and `MinionOnBoard.minionId` are independent never-reused allocators; tokens, graveyard lineage, and replay semantics specified.
2. **Minion → Card Transitions subsection** added to spec §1 — defines `MinionToCardPolicy { Stripped, RetainEnchantments }`, retain/strip table, and the scope note that v1 doesn't track per-source keyword grants (future extension if needed).
3. **`grantedKeywords: string[]`** field added to both `MinionOnBoard` and `Card` — non-aura keyword grants survive `RetainEnchantments` bounces; aura grants do not. `isInverted` preserved across all minion↔card transitions.
4. **`ReturnToHandAction`** signature updated to include `policy: MinionToCardPolicy`. Epic 16 T16.3 rewritten with six scenario tests (Stripped, RetainEnchantments preserves buffs, aura grants don't survive, isInverted preservation, damage always discarded, full-hand destroy).

### Next session resume point

**Discuss the borrow list from the old Unity project** — `docs/superpowers/notes/2026-05-28-old-project-borrow-list.md` contains 13 discrete spec-refinement candidates harvested from the old proof-of-concept. Each has: what's there, why it matters for our spec, recommendation, and open questions. The user picks one item at a time; each decision either amends the spec/plan or gets rejected, then the notes file records the outcome.

Items, ordered by impact (✅ = resolved this pass):
1. ✅ Stabilization loop iteration cap & wave events — ADOPTED (cap=16, abort-as-NoContest, wave markers, scenario-repro telemetry). Also produced a related amendment: Reborn keyword/charge split (`MinionOnBoard.rebornAvailable`). See borrow-list note for full decision.
2. ⏳ **NEXT** — Pre-built `ITriggerCondition` singletons
3. Snapshot triggers before processing (registration safety)
4. Effect-op result chaining + Spell Damage +X hook
5. `GetLegalActions(playerId)` API
6. Deterministic `IRandom` interface
7. Structured error codes
8. Card definition hooks (art / description / sound / tribe)
9. Nested `EffectContext` source-attribution rules
10. Op `Results` ledger (deferred)
11. Game-engine builder pattern (implementation detail, not spec)
12. Targeting strategy registry
13. Command-log vs event-log replay

When all 13 are walked through (adopted, adapted, deferred, or rejected), and the spec + plan are reconciled with the outcomes, implementation can begin at **Epic 01 / T1.1**.

**Do NOT start implementation** while the borrow-list pass is in progress. The user is driving the refinement.

**Key decisions locked during planning:**
- .NET 10 (current LTS), xUnit + FluentAssertions; records for actions/events, classes for state.
- `GameEngine.Submit(action)` returns the full event list; `EventBus` is independently inspectable.
- Trigger fire order = current player first, then by **board index at publish time** (not summonOrder, though summonOrder is kept for disambiguation).
- Death wave: Phase 1 remove → Phase 2 deathrattles → Phase 3 reborn; new deaths deferred to next wave.
- `Card.id` and `MinionOnBoard.minionId` are independent allocators; `definitionKey` is the only cross-entity link.
- `MinionToCardPolicy` (Stripped / RetainEnchantments) parameterises minion→card transitions; `isInverted` always survives both.

---

## Decided feature scope (locked in brainstorming)

### Card types
All four: **Minion, Spell, Weapon, Hero Power**. Hero/class is tied to the deck (like Hearthstone).

### Keywords (all selected, system must be extensible)
Taunt, Divine Shield, Charge, Rush, Lifesteal, Windfury, Poisonous, Stealth, Spell Damage +X, Reborn, Enrage, Freeze — plus any future keywords added via new `IKeyword` implementations.

### Trigger types (all selected, system must be extensible)
Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous), On Damage Taken, On Friendly Minion Death, On Spell Cast, Inspire, Combo, On Invert — plus any future triggers added via new `ITrigger` implementations.

### Inversion mechanic
- A card or minion can be in **Normal** or **Inverted** state
- Stats flip: Attack ↔ Health when inverted
- Trigger type can change: e.g. Battlecry → Deathrattle when inverted
- Effects are individually defined per card (normal and inverted sections in definition JSON)
- Reversible (can be inverted and un-inverted multiple times)
- Applies to cards in hand AND minions on board
- Triggered by: card effect/spell, player-initiated action, on draw/zone entry, opponent action

### Choice / interactive mechanics
- **Mulligan** — pre-game hand selection
- **Discover** — present 3 options, player picks 1
- **Mid-effect targeting** — pause mid-resolution for player to select a target
- **Card crafting** — multi-step interactive sequence producing a custom card
All require a `PendingChoice` game phase.

### Intervention system
When an opponent declares an action, the other player gets a **single-response window** (play one card / skip) before the action resolves. Exact trigger conditions TBD in Game Server spec. Requires `PendingIntervention` game phase.

### Neutral zone
- Third board zone between the two player boards — neither player owns it
- Neutral minions have no turn of their own
- Players can: attack neutral minions, command them to attack, mind control them
- Taunt keyword is **ignored** for neutral minions
- Player auras do **not** affect neutral minions
- Neutral minions enter via: card effect, pre-populated at game start, event trigger
- Zone has a **max capacity** and may **repopulate on turn start** (configured per game mode)

### Architecture pattern
**Event Bus + Component System** — Actions (Commands) → Events → State updates.
- New keyword = new `IKeyword` class
- New trigger = new `ITrigger` class
- New effect = new `IEffect` class
- Custom card logic = new `ICardHandler` that registers triggers into the bus
- Auras = `IAura` recalculated after every board change

---

## Approved data model (Section 1 — approved)

### GameState
```
sessionId: string
player1: PlayerState
player2: PlayerState
neutralZone: MinionOnBoard[]          // ownerId = null; Taunt ignored; auras don't apply
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone this game mode
turn: TurnState
timer: TimerState
phase: GamePhase                      // Mulligan | WaitingForPlayers | InProgress | PendingChoice | PendingIntervention | Ended
winnerId?: string
pendingChoice?: PendingChoice
pendingIntervention?: PendingIntervention
mulliganState?: MulliganState
```

### NeutralZoneConfig
```
maxCapacity: int
spawnPool: Card[]
repopulateOnTurnStart: bool
```

### PlayerState
```
playerId, heroClass: string
health, mana, maxMana, heroAttack: int
hand: Card[]
deck: Card[]
board: MinionOnBoard[]        // player's own side only
graveyard: GraveyardEntry[]   // unified — minions + spells + weapons
weapon?: WeaponOnHero
heroPower: HeroPower
heroPowerUsedThisTurn: bool
cardsPlayedThisTurn: int      // Combo tracking
```

### MinionOnBoard
```
minionId, cardId: string
ownerId: string?              // null = neutral zone
baseAttack, baseHealth: int   // immutable, from card definition
enchantments: StatModifier[]  // permanent buffs — Silence clears
auraAttackBonus, auraHealthBonus: int  // recalculated, never in enchantments
attack: int                   // = baseAttack + Σenchantments.attackDelta + auraAttackBonus
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus
currentHealth: int            // takes damage; healed up to maxHealth
keywords: string[]            // resolved to IKeyword at runtime; Silence clears
isInverted: bool
canAttack: bool
attacksUsedThisTurn: int      // Windfury allows 2
isFrozen, isDamaged: bool
```

### Card
```
id, name: string
type: CardType                // Minion | Spell | Weapon | HeroPower
rarity: CardRarity
baseManaCost: int
baseAttack?, baseHealth?: int // Minion / Weapon only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // has "normal" and "inverted" sections
handlerKey?: string
isInverted: bool
```

### StatModifier (unified — used on both Card and MinionOnBoard)
```
sourceId: string
costDelta: int      // meaningful on Card only; ignored on board
attackDelta: int
healthDelta: int
```

### GraveyardEntry hierarchy
```
GraveyardEntry (abstract)
  originalCard: Card
  turnPlayed: int

GraveyardMinion : GraveyardEntry
  snapshot: MinionOnBoard   // full state at death
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  // base fields sufficient

GraveyardWeapon : GraveyardEntry
  weaponState: WeaponOnHero
  destroyedOnTurn: int
```

### Supporting types
```
WeaponOnHero    — cardId: string, attack: int, durability: int
HeroPower       — cardId: string, manaCost: int, definition: JsonElement, handlerKey?: string
TurnState       — activePlayerId: string, number: int
TimerState      — secondsRemaining: int
PendingChoice   — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[], context: JsonElement
PendingIntervention — respondingPlayerId: string, heldAction: GameAction, timeoutSeconds: int
```

---

## Key files

| File | Purpose |
|---|---|
| `docs/superpowers/specs/2026-05-25-ccg-system-design.md` | System design doc (approved) |
| `docs/superpowers/plans/2026-05-26-game-server.md` | Game Server plan — **SUPERSEDED, do not execute** until new mechanics spec is written and a new plan is created |
| `docs/brainstorm/` | Brainstorm HTML artifacts |
| `.superpowers/brainstorm/` | Visual companion session files |
