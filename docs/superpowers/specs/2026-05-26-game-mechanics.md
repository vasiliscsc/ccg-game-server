# Game Mechanics Spec

**Date:** 2026-05-26  
**Status:** Approved  
**Scope:** `CCG.GameLogic` — the standalone C# class library that implements authoritative game rules. No framework dependencies. Referenced by the Game Server; optionally by the Unity/Godot client.

---

## Section 1: Data Model

### GameState

```
sessionId: string
player1: PlayerState
player2: PlayerState
neutralZone: MinionOnBoard[]          // ownerId = null; Taunt ignored; player auras do not apply
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone in this game mode
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
board: MinionOnBoard[]        // player's own side only; ordered left to right
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
enchantments: StatModifier[]  // permanent buffs; Silence clears all
auraAttackBonus, auraHealthBonus: int  // recalculated each aura pass; never stored in enchantments
attack: int                   // = baseAttack + Σenchantments.attackDelta + auraAttackBonus
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus
currentHealth: int            // takes damage; healed up to maxHealth
keywords: string[]            // resolved to IKeyword at runtime; Silence clears all
isInverted: bool
canAttack: bool
attacksUsedThisTurn: int      // incremented on each attack
attacksAllowedThisTurn: int   // default 1; Windfury sets to 2 on summon
summonOrder: int              // monotonically increasing per session; used for trigger fire ordering
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
WeaponOnHero        — cardId: string, attack: int, durability: int
HeroPower           — cardId: string, manaCost: int, definition: JsonElement, handlerKey?: string
TurnState           — activePlayerId: string, number: int
TimerState          — secondsRemaining: int
PendingChoice       — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[],
                       context: JsonElement  // stores effect continuation
PendingIntervention — respondingPlayerId: string, heldAction: GameAction, timeoutSeconds: int
MulliganState       — player1Completed: bool, player2Completed: bool
CardPlayContext     — card: Card, playerId: string, targetId: string?, state: GameState (read-only)
```

---

## Section 2: Actions + Events

Actions are immutable commands (input layer). Events are immutable facts (output layer). Every state change is produced by an action and announced via one or more events. The client receives events only — never raw state diffs. Both are C# records.

### 2A. GameAction types

All actions carry a nullable `SourcePlayerId` (null = system-issued) and a `RequestedAt` timestamp. The engine infers entity type (minion vs hero vs card) from the ID at processing time — actions do not carry type discriminators.

**Player-initiated:**

| Action | Key fields |
|---|---|
| `PlayCardAction` | playerId, cardId, targetId? |
| `AttackAction` | attackerId, targetId |
| `UseHeroPowerAction` | playerId, targetId? |
| `EndTurnAction` | playerId |
| `SubmitMulliganAction` | playerId, cardIdsToKeep[] |
| `SubmitChoiceAction` | playerId, choiceId, selectedOptionId |
| `SubmitInterventionAction` | playerId, cardId? (null = skip) |
| `SurrenderAction` | playerId |

**System/effect-issued** — emitted by `IEffect`/`ITrigger` implementations; processed through the same pipeline as player actions:

| Action | Key fields |
|---|---|
| `DrawCardAction` | playerId, sourceId? |
| `DealDamageAction` | targetId, amount, sourceId |
| `HealAction` | targetId, amount, sourceId |
| `DestroyMinionAction` | minionId, sourceId? |
| `SummonMinionAction` | cardId, ownerId?, boardPosition, sourceId? |
| `EquipWeaponAction` | playerId, cardId, sourceId? |
| `DestroyWeaponAction` | playerId, sourceId? |
| `GiveCardAction` | playerId, cardId, sourceId? |
| `ReturnToHandAction` | minionId, sourceId? |
| `TransformMinionAction` | minionId, newCardId, sourceId? |
| `InvertTargetAction` | targetId, sourceId? |
| `UnInvertTargetAction` | targetId, sourceId? |
| `BuffMinionAction` | minionId, attackDelta, healthDelta, sourceId |
| `SilenceMinionAction` | minionId, sourceId? |
| `FreezeTargetAction` | targetId, sourceId? |
| `ModifyManaAction` | playerId, delta, sourceId? |
| `ShuffleDeckAction` | playerId, sourceId? |
| `DiscardCardAction` | playerId, cardId? (null = random), sourceId? |
| `SpawnNeutralMinionAction` | cardId, position?, sourceId? |
| `StartChoiceAction` | waitingPlayerId, choiceType, options[], context |
| `StartInterventionAction` | respondingPlayerId, heldAction, timeoutSeconds |

### 2B. GameEvent types

All events carry an `OccurredAt` timestamp and are append-only.

**Lifecycle:**

| Event | Key fields |
|---|---|
| `GameStartedEvent` | sessionId, player1Id, player2Id |
| `GameEndedEvent` | winnerId?, reason (Surrender/Defeat/Draw/Timeout) |
| `TurnStartedEvent` | activePlayerId, turnNumber |
| `TurnEndedEvent` | activePlayerId, turnNumber |
| `MulliganStartedEvent` | playerId, options[] — fired once per player, each with only their own options |
| `MulliganCompletedEvent` | playerId |

**Cards / Hand:**

| Event | Key fields |
|---|---|
| `CardDrawnEvent` | playerId, card |
| `CardAddedToHandEvent` | playerId, card (given by effect, not drawn) |
| `HandOverflowEvent` | playerId, burnedCard |
| `CardPlayedEvent` | playerId, card, targetId? |
| `CardDiscardedEvent` | playerId, card |
| `CardReturnedToHandEvent` | playerId, card (bounced from board) |
| `CardShuffledIntoDeckEvent` | playerId, card |

**Board:**

| Event | Key fields |
|---|---|
| `MinionSummonedEvent` | minion, ownerId? (null = neutral zone) |
| `MinionDiedEvent` | minionId, snapshot, sourceId, diedOnTurn |
| `MinionTransformedEvent` | minionId, newCard |
| `NeutralMinionSpawnedEvent` | minion |
| `NeutralZoneRepopulatedEvent` | spawnedMinions[] |

**Combat:**

| Event | Key fields |
|---|---|
| `AttackDeclaredEvent` | attackerId, targetId |
| `AttackResolvedEvent` | attackerId, targetId, damageToTarget, damageToAttacker |
| `DamageTakenEvent` | targetId, amount, overkill, sourceId |
| `HealedEvent` | targetId, amount, sourceId |

**Status / Keywords:**

| Event | Key fields |
|---|---|
| `MinionSilencedEvent` | minionId |
| `MinionFrozenEvent` | minionId |
| `MinionInvertedEvent` | minionId, isInverted |
| `CardInvertedEvent` | cardId, playerId, isInverted |
| `DivineShieldBrokenEvent` | minionId |
| `EnrageStateChangedEvent` | minionId, isEnraged |
| `MinionStatsChangedEvent` | minionId (signals aura recalc needed) |

**Mana / Hero:**

| Event | Key fields |
|---|---|
| `ManaChangedEvent` | playerId, mana, maxMana |
| `HeroPowerUsedEvent` | playerId, targetId? |
| `WeaponEquippedEvent` | playerId, weapon |
| `WeaponDestroyedEvent` | playerId, weapon |
| `WeaponDurabilityLostEvent` | playerId, remainingDurability |

**Effects / Triggers:**

| Event | Key fields |
|---|---|
| `SpellCastEvent` | playerId, cardId, targetId? |
| `BattlecryTriggeredEvent` | minionId |
| `DeathrattleTriggeredEvent` | minionId |
| `ComboTriggeredEvent` | cardId, playerId |
| `InspireTriggeredEvent` | playerId |

**Choice / Intervention:**

| Event | Key fields |
|---|---|
| `ChoiceStartedEvent` | waitingPlayerId, choiceType, options[] |
| `ChoiceResolvedEvent` | waitingPlayerId, selectedOptionId |
| `InterventionWindowOpenedEvent` | respondingPlayerId, timeoutSeconds |
| `InterventionWindowClosedEvent` | respondingPlayerId, wasSkipped |
| `InterventionPlayedEvent` | respondingPlayerId, cardId |

---

## Section 3: Engine Architecture

Six interfaces form the engine. The first five are the extensibility layer. The sixth is the infrastructure backbone.

### `IActionHandler<TAction>`

One handler per action type. Receives an action and the current `GameState`, mutates state, and returns the events produced. All state mutation in the engine happens inside action handlers — nowhere else.

```csharp
interface IActionHandler<TAction> where TAction : GameAction {
    IEnumerable<GameEvent> Handle(TAction action, GameState state);
}
```

### `IEventBus`

Broadcast backbone. Maintains two subscriber lists per event type: one for the current player's triggers and one for the opponent's. The current player's list always fires first. Within each list, subscribers are sorted by current board index at publish time — not at subscription time. This means a minion that fills a vacated slot fires in the position it currently occupies.

```csharp
interface IEventBus {
    void Subscribe<TEvent>(string listenerId, Action<TEvent, GameState> handler);
    void Unsubscribe(string listenerId);
    void Publish<TEvent>(TEvent evt, GameState state);
}
```

### `IActionQueue`

Effects and triggers enqueue actions here — they never mutate state directly. Ensures every state change goes through an `IActionHandler` and produces auditable events.

```csharp
interface IActionQueue {
    void Enqueue(GameAction action);      // normal priority (back of queue)
    void EnqueueFront(GameAction action); // high priority (front of queue — death resolution only)
}
```

### `IKeyword`

One implementation per keyword. Applied/removed as strings in `MinionOnBoard.keywords`; resolved to the corresponding `IKeyword` instance at runtime.

```csharp
interface IKeyword {
    string KeywordId { get; }
    void OnApplied(string minionId, IEventBus bus, IActionQueue queue);
    void OnRemoved(string minionId, IEventBus bus);
}
```

**Declarative keywords** (Taunt, Divine Shield, Stealth, Charge, Rush) have no-op `OnApplied`/`OnRemoved` — the pipeline queries their presence by string. Divine Shield is handled inside the `DealDamageAction` handler: if target has `divine_shield`, the keyword is removed, `DivineShieldBrokenEvent` is published, and no damage is applied.

**Active keywords** register event bus listeners:

| Keyword | Listener behaviour |
|---|---|
| Lifesteal | `DamageTakenEvent` on self → enqueues `HealAction` for owner |
| Poisonous | `DamageTakenEvent` on target caused by this minion → enqueues `DestroyMinionAction` |
| Windfury | `MinionSummonedEvent` on self → sets `attacksAllowedThisTurn = 2` |
| Spell Damage +X | `SpellCastEvent` by owner → bumps spell damage amount in context |
| Reborn | No listener — `DeathResolutionService` reads `keywords.Contains("reborn")` directly in Phase 3 |
| Enrage | `DamageTakenEvent` + `HealedEvent` on self → enqueues stat update if `isDamaged` changed |
| Freeze | `FreezeTargetAction` handler sets `isFrozen = true`; registers end-of-turn unfreeze listener |

### `IEffect`

Atomic unit of "do something to game state." Used by card definitions and trigger implementations. Effects enqueue actions — they never mutate state.

```csharp
interface IEffect {
    void Execute(EffectContext context, IActionQueue queue);
}
```

`EffectContext` carries: sourceCardId, sourcePlayerId, targetId?, and a read-only `GameState` reference.

### `ITrigger`

Encapsulates "fire when event X occurs." Subscribes to a specific event type; may filter by game state condition.

```csharp
interface ITrigger {
    string TriggerType { get; }
    Type ListensTo { get; }
    bool ShouldFire(GameEvent evt, GameState state);
    void OnFire(GameEvent evt, GameState state, IActionQueue queue);
}
```

Covers the full trigger catalog: Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous, via `IAura`), On Damage Taken, On Friendly Minion Death, On Spell Cast, Inspire, Combo, On Invert. New trigger types are added by implementing this interface.

### `IAura`

Continuous effects recalculated after every board-affecting action. Aura bonuses are **never** stored in `enchantments` — they live in `auraAttackBonus`/`auraHealthBonus` and are fully rewritten on every recalculation pass.

```csharp
interface IAura {
    string AuraId { get; }
    string SourceMinionId { get; }
    IEnumerable<AuraEffect> Calculate(GameState state);
}

record AuraEffect(string TargetMinionId, int AttackBonus, int HealthBonus);
```

A minion that reaches ≤0 `maxHealth` due to an aura loss is treated identically to a minion killed by damage.

### `ICardHandler`

Top-level extensibility point. Cards with non-trivial behaviour have a `handlerKey` in their definition that maps to an `ICardHandler` implementation. Cards without custom logic use `DefaultCardHandler`, which reads the `definition` JSON and dispatches to standard `IEffect` implementations.

```csharp
interface ICardHandler {
    string HandlerKey { get; }
    void OnPlay(CardPlayContext context, IEventBus bus, IActionQueue queue);
    void OnSummon(MinionOnBoard minion, IEventBus bus, IActionQueue queue);
    void OnDeath(MinionOnBoard minion, IEventBus bus);
}
```

`OnDeath` is responsible for enqueuing this minion's Deathrattle actions only. Reborn is always handled by `DeathResolutionService` in Phase 3 — `ICardHandler.OnDeath` must not enqueue its own Reborn.

### Deterministic Ordering

| Concern | Owner |
|---|---|
| Trigger fire order | `IEventBus` — current player first, then opponent; within each, sorted by current board index at publish time |
| Death wave phases | `DeathResolutionService` — Phase 1 (remove) → Phase 2 (deathrattles) → Phase 3 (reborns) |
| Death sort order | `DeathResolutionService` — current player board[0..n] by index, opponent board[0..n] by index, neutral zone by index |
| Deathrattle before Reborn | `DeathResolutionService` — Phase 3 runs only after all Phase 2 actions complete |
| New deaths during deathrattles | `DeathResolutionService` — deferred to next wave, not resolved mid-wave |
| Action isolation | `GameEngine` — one action at a time; next action not started until death check runs to stability |

---

## Section 4: Action Processing Pipeline

Nine stages in strict sequence. The engine does not advance to the next stage until the current one completes.

### ① Submit

Player action arrives via WebSocket, or system action is dequeued from `IActionQueue`.

### ② Phase Guard

Rejects the action if it is not valid in the current `GamePhase`. Returns an error to the client with no state mutation.

| Phase | Accepted actions |
|---|---|
| `Mulligan` | `SubmitMulliganAction` only |
| `InProgress` | All player-initiated actions |
| `PendingChoice` | `SubmitChoiceAction` from the waiting player only |
| `PendingIntervention` | `SubmitInterventionAction` from the responding player only |
| `Ended` | None |

System actions bypass Phase Guard — they are trusted internal commands.

### ③ Action Validator

Precondition checks. Returns an error with no state mutation on failure.

- Turn ownership: is this the active player?
- Mana affordability: `effectiveCost ≤ player.mana`
- Target validity: target is alive, is not stealthed (for opponent spells/attacks), is a legal target type for this card
- Taunt enforcement: if opponent has Taunt minions on board, attacks must target one of them
- Attack eligibility: `canAttack = true`, `isFrozen = false`, `attacksUsedThisTurn < attacksAllowedThisTurn`
- Board space: player board has fewer than 7 minions before summoning

### ④ Dispatch → `IActionHandler`

The registered handler for this action type is invoked. It mutates `GameState` and returns `events[]`. This is the sole point of state mutation in the pipeline.

### ⑤ Publish Events → `IEventBus`

Events are published in the order returned by the handler. For each event, `IEventBus` fires subscribers: current player's list first (sorted by current board index at publish time), then opponent's list (same). Subscribers enqueue new actions via `IActionQueue` — they do not process them inline.

### ⑥ Aura Recalculation

All registered `IAura.Calculate(state)` implementations are run. `auraAttackBonus` and `auraHealthBonus` on every minion are zeroed and fully rewritten. Any minion newly at ≤0 `maxHealth` is marked for death.

Recalculation is triggered after any action that changes board composition, minion stats, or keywords. It also runs after each action processed in Phase 2 of the death wave, and after Phase 3 reborn summons.

### ⑦ Death Resolution

`DeathResolutionService` runs the wave loop:

1. **Collect** all minions with `currentHealth ≤ 0` or marked for destruction. Sort: current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index. If none, exit loop.
2. **Phase 1 — Remove & grieve:** for each in sort order — remove from board, add `GraveyardEntry`, publish `MinionDiedEvent`.
3. **Phase 2 — Deathrattles:** for each in sort order — call `ICardHandler.OnDeath`, which enqueues this minion's deathrattle actions. Process the full action queue (steps ④–⑥ run for each). New deaths during this phase are deferred to the next wave.
4. **Phase 3 — Reborns:** for each minion in the current wave that had the Reborn keyword (in sort order) — enqueue `SummonMinionAction` at 1 HP. Process fully. Run aura recalculation.
5. Back to step 1 (next wave).

### ⑧ Win Condition Check

If any hero has health ≤ 0: if both ≤ 0 the game is a draw, otherwise the other player wins. `GameEndedEvent` is published and `phase` is set to `Ended`.

### ⑨ Dequeue Next Action

Pull the next action from `IActionQueue` and return to ①. If the queue is empty, the engine awaits the next player action.

---

### Turn Lifecycle

The `EndTurnAction` handler and subsequent system actions implement the full turn transition:

1. Publish `TurnEndedEvent`
2. Fire End-of-Turn triggers (current player L→R, then opponent L→R)
3. Swap active player
4. Unfreeze minions frozen during the previous turn
5. Increment `maxMana` (cap 10), restore `mana` to `maxMana`
6. Reset `canAttack`, `attacksUsedThisTurn` on all of the new active player's minions
7. Reset `heroPowerUsedThisTurn`, `cardsPlayedThisTurn`
8. Enqueue `DrawCardAction`
9. Publish `TurnStartedEvent`
10. Fire Start-of-Turn triggers (new active player L→R, then opponent L→R)
11. Start turn timer

Each of these produces actions that go through the full pipeline.

---

### PendingChoice Interruption

When an effect issues `StartChoiceAction`:
- `GamePhase` → `PendingChoice`
- Remaining effect actions (the continuation) are serialised into `PendingChoice.context`
- Pipeline halts — no further actions are processed until the choice resolves
- `SubmitChoiceAction` arrives → selected option is filled into the continuation → continuation actions re-enqueued → `GamePhase` → `InProgress`

### PendingIntervention Interruption

When the engine issues `StartInterventionAction`:
- `GamePhase` → `PendingIntervention`
- `heldAction` stored in `GameState.pendingIntervention` — not yet processed
- Timeout timer starts
- Only `SubmitInterventionAction` from the responding player is accepted
- On response: if a card was played, it is processed first (full pipeline), then `heldAction` is processed
- On skip or timeout: `heldAction` is processed directly
- `GamePhase` → `InProgress`
