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
keywords: string[]            // resolved to IKeyword at runtime; engine's live view (base + granted + aura-applied). Silence clears all.
grantedKeywords: string[]     // subset of keywords added by non-aura buffs after summon (base keywords from the definition are NOT here). Used by RetainEnchantments bounce; Silence clears.
isInverted: bool
canAttack: bool
attacksUsedThisTurn: int      // incremented on each attack
attacksAllowedThisTurn: int   // default 1; Windfury sets to 2 on summon
summonOrder: int              // monotonically increasing per session; used for trigger fire ordering
isFrozen, isDamaged: bool
rebornAvailable: bool         // the one-time Reborn charge — distinct from the "reborn" keyword tag, which persists. Initialized at summon to keywords.Contains("reborn"); the reborn-summon path summons with it false; consuming it (NOT the keyword) is what reborn does. See §4 ⑦.
```

### Card

```
id, name: string
type: CardType                // Minion | Spell | Weapon | HeroPower
rarity: CardRarity
baseManaCost: int
baseAttack?, baseHealth?: int // Minion / Weapon only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
grantedKeywords: string[]     // keywords carried over from a RetainEnchantments bounce/shuffle. On play, these migrate to the new minion's grantedKeywords (and keywords) — they are not part of the card's base definition.
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // has "normal" and "inverted" sections
handlerKey?: string
isInverted: bool              // intrinsic state; preserved across all minion↔card transitions
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

### Card and Minion Identity

Cards and minions are independent entities with independent ID allocators. They are linked only by a shared `definitionKey` (string — e.g. `"river_crocolisk"`, `"boar"`) into the card-definition library.

**Identity rules:**

- `Card.id` and `MinionOnBoard.minionId` are allocated from two distinct counters; **IDs are never reused** within a session.
- The `definitionKey` field on `Card` and on `MinionOnBoard` points into the static card-definition library. It is the only cross-entity link.
- Playing a Card to summon a Minion does **not** transfer the Card's `id` to the minion. The Card leaves play (to graveyard or fabricated-elsewhere); the Minion is a new entity with a fresh `minionId`.
- Tokens (minions summoned by effects with no originating card) have a `minionId` and a `definitionKey`. There is no `Card.id` associated with them while on the board.
- `GraveyardMinion.originalCard` carries the originating Card snapshot for lineage-aware effects ("resummon the last minion that died"). The resummon uses only the snapshot's `definitionKey` — a *new* `minionId` is allocated.

### Minion → Card Transitions (Bounce / Shuffle-In)

Any action that moves a minion off the board into a card-shaped zone (hand or deck) takes a `policy` parameter of type `MinionToCardPolicy`:

```
MinionToCardPolicy = Stripped | RetainEnchantments
```

The effect that triggers the transition decides which policy applies (e.g. a Sap-style spell uses `Stripped`; a Recall-style spell uses `RetainEnchantments`). The action itself encodes the policy; the handler implements both behaviours.

**Fabrication rule:** every minion→card transition fabricates a *new* Card with a freshly allocated `Card.id`. The minion's `minionId` is retired. Lineage to any earlier originating Card (if one existed) is not preserved on the fabricated Card.

**What's retained, by policy:**

| Field on the new Card | `Stripped` | `RetainEnchantments` |
|---|---|---|
| `definitionKey` | from minion | from minion |
| `isInverted` | from minion | from minion |
| `modifiers` | `[]` | copy of minion's `enchantments` (costDelta set to 0; attack/healthDelta preserved) — plus any cost-affecting modifiers the spec extends to in future |
| `grantedKeywords` | `[]` | copy of minion's `grantedKeywords` |

**Always lost (under both policies):**

| Lost | Why |
|---|---|
| `currentHealth` damage | A Card has no current-health concept — only base + modifiers. The minion returns to full health on replay. |
| `auraAttackBonus`, `auraHealthBonus` | Auras are recalculated continuously and never persisted on the minion; off-board they cease to apply. |
| Aura-granted keywords | Granted by a live aura against board minions. Absent from `grantedKeywords`; vanish on transition. |
| `attacksUsedThisTurn`, `canAttack`, `isFrozen`, `summonOrder` | Runtime board state with no Card-side equivalent. |
| Base keywords from the card definition | Re-derived on next play from the definition library, not carried on the Card. |

**Replay semantics:** when the fabricated Card is played again, the resulting minion inherits `grantedKeywords` from the Card into both `keywords` and `grantedKeywords` on the new minion, and inherits `modifiers` into the new minion's `enchantments`. `isInverted` is copied through unchanged. The new minion always has a fresh `minionId`.

**Scope note — keyword grant tracking:** `grantedKeywords` only captures non-aura buffs (effects that mutate the minion permanently). Aura-applied keywords are never added to `grantedKeywords`; they live only in `keywords` and are rewritten on every aura recalculation. Silence clears both `keywords` and `grantedKeywords`. This design avoids any per-source ledger in v1; if a future card requires per-source attribution (e.g. "remove only the Taunt that Sergeant gave," not the base Taunt), the model must extend `grantedKeywords` to `KeywordGrant { sourceId, keyword }`.

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
| `ReturnToHandAction` | minionId, policy: MinionToCardPolicy, sourceId? |
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

All events carry an `OccurredAt` timestamp and an `originEpoch: int` (the `currentActionEpoch` of the action that produced the event; used by `IEventBus` for creation-epoch visibility filtering — see §3 `IEventBus`). Events are append-only.

**Lifecycle:**

| Event | Key fields |
|---|---|
| `GameStartedEvent` | sessionId, player1Id, player2Id |
| `GameEndedEvent` | winnerId?, reason (Surrender/Defeat/Draw/Timeout/NoContest) |
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
| `DeathWaveStartedEvent` | waveIndex |
| `DeathWaveEndedEvent` | waveIndex, minionsResolved |
| `StabilizationAbortedEvent` | wavesReached, lastWaveMinionIds[] — fatal; always immediately followed by `GameEndedEvent { reason: NoContest }` |

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

**Publish-time snapshot + creation-epoch visibility.** On every `Publish(E)` the bus snapshots the relevant subscriber lists at that instant and iterates the copy — a listener registered while `E` is being dispatched does not receive `E`. Dispatch is then filtered by **creation epoch**, so a listener never receives an event produced by the same action that created it (most importantly, a minion never reacts to its own `MinionSummonedEvent`):

- `GameEngine` maintains a monotonic `currentActionEpoch`, incremented as each action begins processing (stage ④).
- Each subscriber is stamped `birthEpoch = currentActionEpoch` when `Subscribe` is called.
- Each event is stamped `originEpoch = currentActionEpoch` when its producing handler runs (alongside `OccurredAt`).
- `Publish(E)` dispatches to listener `L` **only if `L.birthEpoch < E.originEpoch`**.

Consequences: a listener receives only events from actions *strictly later* than the action that created it. Events already in flight from earlier actions are missed (the listener did not exist when they published); events from the *same* action — including the listener's own creation event and any sibling events — are excluded by the equal-epoch comparison; every subsequent action's events are received. This reproduces Hearthstone's rule (a freshly summoned or transformed minion does not trigger off the event that introduced it) **uniformly for every entity-introducing event**, with no per-event special-casing. Because the filter is a pointwise integer comparison against the *live* list, removed (unsubscribed) listeners are simply absent — there is no stale frozen list that could fire.

**Invariant — subscriptions happen only inside action handlers.** Every bus subscription occurs during stage ④ (`ICardHandler.OnSummon`, `IKeyword.OnApplied`), so every listener has a well-defined `birthEpoch`. Nothing subscribes during event publication (⑤) or aura recalculation (⑥). This invariant is what makes the epoch comparison total.

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
| Reborn | No listener — `DeathResolutionService` reads `keywords.Contains("reborn") && rebornAvailable` directly in Phase 3. Firing consumes `rebornAvailable` only; the keyword persists (still counts for auras, survives bounce/copy). |
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

Covers the full trigger catalog: Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous, via `IAura`), On Damage Taken, On Friendly Minion Death, On Spell Cast, Inspire, Combo, On Invert. New trigger types are added by implementing this interface. `ShouldFire` is typically delegated to an `ITriggerCondition` (below) rather than hand-written per card.

### `ITriggerCondition`

Reusable, composable predicates that answer one question: **"should this trigger fire?"** They are separated from `ITrigger` so the common firing filters are written once, tested once, and shared across every card. An `ITrigger.ShouldFire` implementation typically delegates to a single condition or a composed tree of them.

```csharp
interface ITriggerCondition {
    bool Matches(GameEvent evt, GameState state, string sourceId);
}
```

`sourceId` is the trigger's host entity (the minion or hero the trigger belongs to). Conditions evaluate the event *relative to* the host — e.g. "friendly" means "owned by the same player as `sourceId`".

**Single conditions — parameterless singletons** (referenced by reference; no per-check allocation):

| Condition | Matches when |
|---|---|
| `AlwaysFires` | always |
| `SelfIsSource` | the event's acting/source entity is `sourceId` |
| `SelfIsTarget` | the event's target entity is `sourceId` |
| `SelfIsRelated` | the event's primary subject (e.g. the dying minion in `MinionDiedEvent`) is `sourceId` |
| `FriendlyOnly` | the event's relevant entity is owned by the same player as `sourceId` |
| `EnemyOnly` | the event's relevant entity is owned by the opponent of `sourceId` |

**Single conditions — parameterized factories:**

| Condition | Matches when |
|---|---|
| `MinionTypeIs(definitionKey)` | the event's relevant minion has that `definitionKey`. (Tribe-based matching — "any Beast" — depends on Item 8 tribes, not yet in the model; this matches a *specific* minion definition.) |
| `CardTypeIs(CardType)` | the event's relevant card is a Minion / Spell / Weapon / HeroPower — e.g. "whenever you play a Spell **or** a Weapon" |
| `CostAtLeast(n)` / `CostAtMost(n)` | the event's relevant card's `effectiveCost` meets the threshold — e.g. "whenever you cast a 2-mana-or-higher spell" |

**Combinators** compose conditions into a tree:

| Combinator | Matches when |
|---|---|
| `All(c1, c2, …)` (AND) | every child matches; `All()` with no children matches |
| `Any(c1, c2, …)` (OR) | at least one child matches; `Any()` with no children does not match |

Combinators nest. Example — *"whenever a friendly Murloc or Beast dies"*:
`All(FriendlyOnly, Any(MinionTypeIs("murloc"), MinionTypeIs("beast")))`.

(`Not(c)` is the obvious third combinator — same shape — needed the first time a card reads "any *other* friendly minion died" = `All(FriendlyOnly, Not(SelfIsRelated))`. Left out of v1 until a card requires it.)

**One card, many triggers.** A card's `definition` carries a `triggers` array; `DefaultCardHandler` registers each entry as its own `ITrigger` subscription, all active simultaneously for the host's lifetime. The condition library applies *per trigger*. These are two orthogonal axes: **combinators compose conditions *within* a single trigger; separate effects that watch different events are separate triggers.** Example — a minion reading *"Whenever you play a 2-mana-or-higher spell, summon a Beast. Whenever a Beast dies, draw a card."* registers two triggers:

```jsonc
"triggers": [
  { "type": "OnSpellCast",
    "condition": { "all": ["FriendlyOnly", { "costAtLeast": 2 }] },
    "effect": { "summon": "beast_token" } },
  { "type": "OnFriendlyMinionDeath",
    "condition": { "minionType": "beast" },
    "effect": { "draw": 1 } }
]
```

**Conditions gate; trigger type routes.** `ITriggerCondition` answers only *whether* a trigger fires. *When and in what order* it resolves is governed by the trigger's `TriggerType`, not its condition — and several types carry engine-defined resolution semantics the condition layer does not touch:

- **Battlecry** resolves synchronously inside the PlayCard pipeline (may pause for targeting via `PendingChoice`), *before* the minion's persistent triggers go live.
- **Deathrattle** resolves in death-wave **Phase 2** (§4 ⑦), in active-player-L→R order, off the minion's death snapshot — *not* at `MinionDiedEvent` publish time. (Contrast `OnFriendlyMinionDeath`, a generic reactive trigger that *does* fire at publish time. Same event, different condition: Deathrattle = `SelfIsRelated`, OnFriendlyMinionDeath = `FriendlyOnly` excluding self.)

So Battlecry and Deathrattle use the same `{ type, condition, effect }` shape and the same condition library (a conditional Battlecry like "if you're holding a Dragon" is just a custom or parameterized condition), but their timing/ordering remains defined by §4. Conditions cover the common gates; bespoke predicates that inspect arbitrary game state still implement `ITriggerCondition` directly.

**JSON encoding** of the condition tree (the `{ "all": [...] }` / `{ "minionType": "…" }` shapes above) is a `DefaultCardHandler` implementation detail, *not* part of this spec — it settles when the first compound card is authored.

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
| Event visibility (new subscribers) | `IEventBus` — snapshot at publish + `birthEpoch < originEpoch` creation-epoch filter; a listener never receives events from the action that created it |
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

The registered handler for this action type is invoked. It mutates `GameState` and returns `events[]`. This is the sole point of state mutation in the pipeline. `GameEngine` increments `currentActionEpoch` as the action enters this stage; every event the handler returns is stamped with that `originEpoch`, and any subscription the handler registers (e.g. via `ICardHandler.OnSummon`) is stamped with the same value as its `birthEpoch` — see §3 `IEventBus`.

### ⑤ Publish Events → `IEventBus`

Events are published in the order returned by the handler. For each event, `IEventBus` snapshots the subscriber lists at publish time and fires them filtered by creation epoch (`birthEpoch < originEpoch`, see §3 `IEventBus`): current player's list first (sorted by current board index at publish time), then opponent's list (same). The epoch filter means a listener never receives an event from the action that created it. Subscribers enqueue new actions via `IActionQueue` — they do not process them inline.

### ⑥ Aura Recalculation

All registered `IAura.Calculate(state)` implementations are run. `auraAttackBonus` and `auraHealthBonus` on every minion are zeroed and fully rewritten. Any minion newly at ≤0 `maxHealth` is marked for death.

Recalculation is triggered after any action that changes board composition, minion stats, or keywords. It also runs after each action processed in Phase 2 of the death wave, and after Phase 3 reborn summons.

### ⑦ Death Resolution

`DeathResolutionService` runs the wave loop:

1. **Collect** all minions with `currentHealth ≤ 0` or marked for destruction. Sort: current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index. If none, exit loop. Otherwise publish `DeathWaveStartedEvent { waveIndex }` (0-based, incremented per wave).
2. **Phase 1 — Remove & grieve:** for each in sort order — remove from board, add `GraveyardEntry`, publish `MinionDiedEvent`.
3. **Phase 2 — Deathrattles:** for each in sort order — call `ICardHandler.OnDeath`, which enqueues this minion's deathrattle actions. Process the full action queue (steps ④–⑥ run for each). New deaths during this phase are deferred to the next wave. Minions summoned during this phase are registered with the `birthEpoch` of their own `SummonMinionAction`, so the creation-epoch filter (§3 `IEventBus`) keeps them from reacting to `MinionDiedEvent`s already published in this wave — they react only to events from subsequent actions.
4. **Phase 3 — Reborns:** for each minion in the current wave with the Reborn keyword **and** `rebornAvailable == true` (in sort order) — enqueue `SummonMinionAction` at 1 HP. Process fully. Run aura recalculation. The reborn copy **keeps the Reborn keyword** but is summoned with `rebornAvailable = false`, so it cannot reborn again (if it dies it goes to graveyard normally). The keyword still counts for auras and survives bounce. A *copy* made of that spent minion is a fresh summon → `rebornAvailable` re-initializes to `true` from the keyword, so the clone reborns if it dies.
5. Publish `DeathWaveEndedEvent { waveIndex, minionsResolved }`. Back to step 1 (next wave).

**Termination — iteration cap.** The wave loop is bounded by `GameConstants.MaxDeathWaves = 16`. Reborn cannot drive an unbounded loop (it is self-consuming, step 4), so the only runaway source is Phase 2 deathrattles/summons that keep producing new deaths — almost always a card-definition bug. If the loop is about to begin a wave whose `waveIndex` would reach `MaxDeathWaves`, the engine treats this as a fatal assertion failure, **not** a game event:

1. Publish `StabilizationAbortedEvent { wavesReached, lastWaveMinionIds[] }`.
2. Publish `GameEndedEvent { winnerId: null, reason: NoContest }` and set `phase = Ended`. The match is **voided** — not a draw (a draw would unfairly affect MMR).
3. Emit a `StabilizationAbortReport` to the telemetry sink (below).

The action is **not** rolled back and replayed — rollback would restore the exact state the runaway re-triggers from, so it cannot resolve the match. Aborting the whole match is the only safe escalation. There is no rollback/snapshot dependency: the engine simply tears the match down.

This is a last-line backstop. The primary mitigation is upstream content validation at card-load time (flag obvious self-feeding patterns such as a deathrattle that summons a 0-health token); general termination is undecidable, which is why the runtime cap remains.

**`StabilizationAbortReport` (telemetry — scenario-reproducible).** Formatted so a developer can paste it directly into a `CCG.GameLogic.Tests` scenario fixture (Build → Script → Assert), guaranteeing the runaway is reproduced and the abort behavior never regresses:

```
StabilizationAbortReport {
  rngSeed:            ulong          // GameState.rngSeed — replays all randomness (see Item 6 / IRandom)
  preActionState:     GameState      // full snapshot at the START of the triggering action — the Build step
  triggeringAction:   GameAction     // the action submitted that began the runaway — the Script step
  wavesReached:       int            // = MaxDeathWaves
  cascadeTrace:       GameEvent[]     // ordered events from action start to abort — the Exact-trace Assert
}
```

A regression scenario built from a report asserts: submitting `triggeringAction` against `preActionState` (seeded with `rngSeed`) terminates at wave `MaxDeathWaves` with `StabilizationAbortedEvent` then `GameEndedEvent { NoContest }` — i.e. it aborts rather than hangs.

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
