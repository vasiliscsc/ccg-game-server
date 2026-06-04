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
rngSeed: ulong                        // fixed at match creation; drives ALL randomness via IRandom; server-side, never sent to client
currentActionEpoch: int               // monotonic; ++ per action at stage ④; used for event-visibility filter AND per-action RNG derivation (see §3 IEventBus, IRandom)
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
currentHealth: int            // takes damage; healed up to maxHealth. ≤0 (or destroy-marked) ⇒ "mortally wounded" (pending death): the minion stays ON the board — still targetable, countable, and keyword-active — until the next death-wave settle (§4 ⑦), not removed on the spot (amendment B)
keywords: string[]            // maintained EFFECTIVE view = intrinsicKeywords ∪ grantedKeywords ∪ auraKeywords; resolved to IKeyword at runtime; what the engine queries. (Keyword analog of effectiveTribes.)
intrinsicKeywords: string[]   // from the card definition, seeded at summon; Silence clears
grantedKeywords: string[]     // PERMANENT non-aura grants after summon; retained across a RetainEnchantments bounce; Silence clears. Aura grants are NOT here.
auraKeywords: string[]        // aura-granted, recomputed each aura pass; never persisted, never bounce-retained; Silence does not touch (they recompute). v1: DECLARATIVE keywords only — see "Unaddressed Features".
intrinsicTribes: Tribe        // copied from the card's tribes at summon; immutable; survives Silence
grantedTribes: Tribe          // permanent tribe grants ("becomes a Beast"); Silence clears to Tribe.None
auraTribes: Tribe             // recomputed each aura pass; never persisted (parallels auraAttackBonus); drops when the aura leaves
effectiveTribes: Tribe        // computed = intrinsicTribes | grantedTribes | auraTribes; all engine tribe checks use this
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
tribes: Tribe                 // [Flags] intrinsic taxonomy from the definition; Tribe.None = tribeless (valid). Gameplay tag; NOT cleared by Silence. See "Tribes" below.
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
PendingIntervention — respondingPlayerId: string, heldAction: GameAction?, timeoutSeconds: int  // heldAction null = post-reaction window (nothing held — e.g. a dying-window save)
MulliganState       — player1Completed: bool, player2Completed: bool
CardPlayContext     — card: Card, playerId: string, targetId: string?, state: GameState (read-only)
```

### Tribes

Tribes are **gameplay tags** (tribal synergies — "your Beasts get +1/+1", "Discover a Murloc"), queried by the engine during aura recalc and effect resolution. Unlike keywords, a tribe is **inert** — it carries no behavior of its own; all behavior lives in the cards that reference it. It is therefore a closed, designer-curated taxonomy rather than a string-keyed extension point, and is modeled as a `[Flags]` enum so multi-tribe membership, granting, and removal are bitwise operations.

```csharp
[Flags]
enum Tribe {
    None   = 0,        // tribeless — a valid state
    Beast  = 1 << 0,
    Demon  = 1 << 1,
    Murloc = 1 << 2,
    // … new tribes ship with the content release that adds their cards (new cards are a library version bump anyway)
}
```

A minion's effective tribes come from three sources, kept in separate fields so aura contributions never pollute persistent state (the same discipline as `auraAttackBonus`):

- **`intrinsicTribes`** — from the card definition, copied to the minion at summon; immutable. **Survives Silence** (a silenced Beast is still a Beast).
- **`grantedTribes`** — permanent grants from effects ("becomes a Beast in addition to its other types"). **Silence clears these to `Tribe.None`.**
- **`auraTribes`** — granted by an active `IAura` ("your other minions are also Beasts"); fully recomputed each aura pass, never persisted, drops automatically when the aura leaves.

`effectiveTribes = intrinsicTribes | grantedTribes | auraTribes`; all engine tribe checks read it, e.g. `(minion.effectiveTribes & Tribe.Beast) != 0`. Cards expose only `tribes: Tribe` (their intrinsic taxonomy, possibly `Tribe.None`); the three-way split exists only on `MinionOnBoard`. `Tribe.None` participates correctly in all the bitwise operations, so tribeless cards/minions need no special-casing.

### Presentation data (out of library scope)

Art, display description, and sound cues are **not** part of `CCG.GameLogic`. They never affect gameplay, and the client already carries `definitionKey` on every entity it renders, so it resolves these from the presentation catalog (owned by the Platform API) by `definitionKey` — exactly as it resolves card art. Keeping them out preserves the library's "standalone, no presentation dependencies" property: the engine emits `definitionKey`, the client joins on it. (`rarity` stays on `Card` because it is plausibly gameplay-relevant — e.g. "Discover a Legendary".)

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
| `SubmitInterventionAction` | playerId, cardId? (null = skip), targetIds[]? (the response card's chosen targets — for a batched post-reaction window, picked from the window's candidate set = the accumulated `matched` set; validated by the §4 ③ membership check) |
| `SurrenderAction` | playerId |

**System/effect-issued** — emitted by `IEffect`/`ITrigger` implementations; processed through the same pipeline as player actions:

| Action | Key fields |
|---|---|
| `DrawCardAction` | playerId, sourceId? |
| `DealDamageAction` | target: `entityId` **or** `ITargetSelector` (AoE — evaluated at ④ over the current board), amount, sourceId — **the one channel for all damage, combat included** (see §3 "Reactive…"). A selector-carrying instance is a single declared action → a single ③′ interception window |
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
| `StartInterventionAction` | respondingPlayerId, heldAction? (null = post-reaction window — nothing held, e.g. a dying-window save), timeoutSeconds |

### 2B. GameEvent types

All events carry an `OccurredAt` timestamp and an `originEpoch: int` (the `currentActionEpoch` of the action that produced the event; used by `IEventBus` for creation-epoch visibility filtering — see §3 `IEventBus`). Events are append-only.

**Bus ⊋ log.** Almost every event below is a **handler-produced state delta** (emitted at ⑤) and populates the append-only **event log** — the client wire format and replay archive. The bus, however, carries a *superset*: the lone **`ActionDeclaredEvent`** is a transient **pre-execution interception signal** (emitted at ③′, before any state change) that rides the bus only so `ITrigger`s can subscribe — it is **excluded from the persisted event log and the client wire format**, because it represents intent, not a delta. So "the event log is exactly the state-delta stream" stays true even though the bus delivers one extra signal type.

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
| `MinionMortallyWoundedEvent` | minionId, sourceId, cause — fires the instant a minion enters pending-death (currentHealth ≤0 from damage, maxHealth ≤0 from aura loss, or a destroy-mark), **before** the death-wave settle removes it; distinct from `MinionDiedEvent` (Phase-1 removal). The hook for dying-window reactions (e.g. Intervention × lethal, amendment B ref R1). |
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
| `HeroMortallyWoundedEvent` | playerId, sourceId, cause — fires the instant a hero crosses to ≤0 health (combat / spell / fatigue / self-damage), **before** the ⑧ win-check finalizes the loss; the hero analogue of `MinionMortallyWoundedEvent`, distinct from `GameEndedEvent` (the ⑧ finalization). The hook for save-the-hero reactions (heal-above-0 / immune in response to lethal — e.g. an Ice-Block-style effect realised as a post-reaction rather than damage prevention; for prevention see "Interception", §3). |
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
| `ActionDeclaredEvent` | action — the universal **pre-execution** signal: published at stage **③′** (§4) for every dispatched action *after* it validates and *before* it executes (④), carrying the action itself. The single hook for **interception** (declaration-phase) reactions; subscribers filter on the carried action's runtime type/params via the condition library, so **no per-action-type `*Declared` taxonomy is required**. Fires **once** per action — a held action resumed after interception is re-validated but **not** re-declared (prevents redirect/secret loops). **Bus-only: not a state delta, so excluded from the persisted event log and client wire format** (see the "Bus ⊋ log" note above). (The pre-existing typed `AttackDeclaredEvent` is the combat-specific declaration, retained for renderer convenience at the same timing; it is not the general hook.) |

**Choice / Intervention:**

| Event | Key fields |
|---|---|
| `ChoiceStartedEvent` | waitingPlayerId, choiceType, options[] |
| `ChoiceResolvedEvent` | waitingPlayerId, selectedOptionId |
| `InterventionWindowOpenedEvent` | respondingPlayerId, timeoutSeconds |
| `InterventionWindowClosedEvent` | respondingPlayerId, wasSkipped |
| `InterventionPlayedEvent` | respondingPlayerId, cardId |

### 2C. Action Rejection

When validation (§4 ②③) fails, the action is rejected with a **structured code** — never an English string baked into the engine. A rejection is the negative arm of `Submit`'s return; it is **not** a `GameEvent`, never touches the event bus, and never enters the event log. (Rejected actions cause no state change, so on replay they simply re-reject — consistent with Item 6's "log = real state-changing player actions only".)

```csharp
enum ActionRejectionCode {
    WrongPhase, NotActivePlayer, CardNotInHand, NotEnoughMana,
    InvalidTarget, TargetStealthed, MustTargetTaunt,
    AttackerNotAlive, AttackerCannotAttack, AttackerFrozen, AttackerExhausted, BoardFull,
    // … closed set; grows in lockstep with the §4 ②③ validation rules
}

record ActionRejection(ActionRejectionCode Code, string? Detail = null);
```

`Code` is **the contract**: tests assert on it, and the Game Server maps it to a localized client string. **`Detail` is optional, free-form, and explicitly non-contractual** — a human-readable diagnostic for logs only (e.g. `"needs 5, has 3"`). It is never asserted in tests and never shown to users (clients localize from `Code`), and may be `null`. Anything richer — typed per-code payloads — is deliberately out of scope for v1: the data is reconstructable by the caller from `(action, state)`, so a *specific* code can be upgraded to carry structured context later if one is ever found that genuinely cannot. The whole-mechanism version is not built up front.

`Submit` returns a discriminated result rather than a bare event list:

```csharp
SubmitResult = Accepted(IReadOnlyList<GameEvent> events)
             | Rejected(ActionRejection rejection);
```

The pure validation predicate (§4 ③) returns `Ok | Rejection` using this same `ActionRejection`; `Submit` wraps a non-`Ok` predicate result into `Rejected` without ever entering stage ④. The Game Server relays the `Rejected` result to the originating client as the response to its request.

---

## Section 3: Engine Architecture

A small set of interfaces forms the engine: the **extensibility layer** (`IKeyword`, `IEffect`, `ITrigger` with its `ITriggerCondition`, `ITargetSelector`, `IAura`, `ICardHandler`) plus the **infrastructure backbone** (`IActionHandler`, `IEventBus`, `IActionQueue`, `IRandom`).

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

**Invariant — subscriptions happen only inside action handlers.** Every bus subscription occurs during stage ④ (`ICardHandler.OnSummon` registering this card's `ITrigger`s), so every listener has a well-defined `birthEpoch`. Nothing subscribes during event publication (⑤) or aura recalculation (⑥). This invariant is what makes the epoch comparison total. (Keywords do **not** subscribe — their behaviour is pulled at stage ④, see §3 `IKeyword` — so the bus carries only `ITrigger` subscriptions.)

### `IActionQueue`

Effects and triggers enqueue actions here — they never mutate state directly. Ensures every state change goes through an `IActionHandler` and produces auditable events.

```csharp
interface IActionQueue {
    void Enqueue(GameAction action);      // normal priority (back of queue)
    void EnqueueFront(GameAction action); // high priority (front of queue — death resolution only)
}
```

### `IKeyword`

Keywords are **declarative markers** on a minion's effective `keywords` view (intrinsic | granted | aura — see `MinionOnBoard`). A keyword **never subscribes to the event bus**; its behaviour is *pulled* at the minion's own action moments. This is the deliberate cut between the two extensibility points:

> **Keyword** — behaviour at the minion's *own* action moments (deal / take damage, attack, summon, stat recompute), read from declarative state and pulled by the relevant stage-④ handler.
> **Trigger** (`ITrigger`) — reaction to *board-wide* events, via a bus subscription with the creation-epoch filter.

Because keyword behaviour is always pulled from the **effective** view at the moment of use, **aura-granted keywords behave identically to intrinsic ones** (the aura contributes to `auraKeywords`, which is in the effective view), and a keyword on a minion that has left the board is simply never read. There is no subscription lifecycle and no registration-during-recalc problem — which is why aura grants are unrestricted (any keyword, not just "declarative" ones) and the epoch filter (§3 `IEventBus`) is needed only by `ITrigger`.

Behaviour attaches two ways.

**1. Inline reads** — most keywords are queried from the effective view at one decision point:

| Keyword | Read at |
|---|---|
| Taunt | attack validation (§4 ③ — opponent must attack a Taunt minion) |
| Stealth | targeting validation (cannot be targeted by the opponent) |
| Charge / Rush | summon-turn attack eligibility (`canAttack`) |
| Windfury | `attacksAllowedThisTurn = keywords has windfury ? 2 : 1`, read at attack eligibility |
| Divine Shield | consumed inside `DealDamageAction`: if target has it → remove it, publish `DivineShieldBrokenEvent`, apply no damage |
| Spell Damage +X | *pulled* and aggregated at spell-resolution start (`EffectContext.SpellDamageBonus`); magnitude in the keyword string (`spell_damage:1`) |
| Reborn | read by `DeathResolutionService` Phase 3 (`keywords has reborn && rebornAvailable`); firing consumes `rebornAvailable`, the keyword persists (counts for auras, survives bounce/copy) |
| Enrage | a conditional buff recomputed in the stage-⑥ aura/stat pass from `isDamaged` (not an event reaction) |

**2. Role-interface hooks** — a keyword whose behaviour *enqueues a follow-on action* at a specific moment implements a small hook interface. The owning action handler iterates the acting minion's **effective** keywords implementing that interface and invokes each — no subscription, all inside stage ④:

```csharp
interface IKeyword { string KeywordId { get; } }

// Invoked by DealDamageAction over the SOURCE's effective keywords, after damage applies.
interface IOnDealtDamage : IKeyword {
    void OnDealtDamage(string sourceId, string targetId, int amount,
                       GameState state, IActionQueue queue);
}
```

v1 hooks (`IOnDealtDamage`): **Lifesteal** → enqueue `HealAction` for the source's owner; **Poisonous** → enqueue `DestroyMinionAction` on a damaged enemy minion. The handler:

```csharp
ApplyDamage(target, amount);
foreach (var kw in EffectiveKeywords(source).OfType<IOnDealtDamage>())
    kw.OnDealtDamage(source, target, amount, state, queue);
```

Further "own-moment" hook points (on-take-damage, on-attack, …) are added as new role interfaces when a card first needs one; each is invoked the same way by its owning handler. **Freeze** is not a keyword hook — it is a status (`isFrozen`) set by `FreezeTargetAction` and cleared by the turn-lifecycle unfreeze sweep (§4 Turn Lifecycle step 4).

### `IEffect`

Atomic unit of "do something to game state." Used by card definitions and trigger implementations. Effects enqueue actions — they never mutate state.

```csharp
interface IEffect {
    void Execute(EffectContext context, IActionQueue queue);
}
```

`EffectContext` carries: `sourceId`, `sourcePlayerId`, `targetId?`, a read-only `GameState` reference, and `SpellDamageBonus: int`. (`sourceId` is the unified entity reference — formerly `sourceCardId`, generalized because a triggered effect's source is often a *minion*, not a card. See "Source Attribution" below.)

#### Source Attribution

Every action carries a `sourceId`, and `EffectContext` is built from the **action's own** source fields. Attribution follows one rule:

> **`sourceId` is the entity whose effect this is — set by whoever *enqueues* the action — and is never inherited from the upstream cause.**

- A played card's effects (including Battlecry) → `sourceId` = the card.
- A minion's triggered effects (Deathrattle, On-Damage, …) → `sourceId` = that minion. A trigger's `OnFire` stamps `sourceId = its host entity`; it does **not** pass along the source of the action that fired the trigger.
- Hero power / weapon effects → that hero power / weapon.
- `sourcePlayerId` = the source's controller, captured at enqueue time (from the death snapshot if the source is already dead).

So if Yeti's Deathrattle damages a minion, the damage's `sourceId` is **Yeti**, not the spell that killed Yeti. This determines friendly/enemy evaluation (Item 2 conditions), the Lifesteal heal target, and which entity "dealt" the damage for any watching trigger.

**Attribution ≠ keyword application.** `sourceId` is *data* on the event, so a watcher correctly credits Yeti even though Yeti is dead. But Lifesteal/Poisonous are **keyword hooks pulled from the source's effective keywords** (§3 `IKeyword`), and a dead Yeti is off the board — so `EffectiveKeywords(source)` is empty and the hooks do **not** fire for Yeti's own post-death Deathrattle damage. No listener lifecycle is involved; this falls out of "read effective keywords from live board state" with no special-casing. See "Unaddressed Features" for the case where a card would *want* dead-source keyword effects (enabled by a death-snapshot keyword fallback).

**`SpellDamageBonus`** is a read-only value computed **once at the start of spell resolution** and held on the context for the whole cast. Every `DealDamageEffect` in that spell reads the *same* snapshot and bakes the final number into its enqueued `DealDamageAction` (the action handler stays dumb — it applies the number it's given). Because the bonus is spell-scoped, not hit-scoped: a spell that deals damage twice still applies the bonus to both hits, and a spell that kills its own Spell-Power minion mid-cascade does not lose the bonus for its second half. It is `0` for any non-spell source, so it cannot leak into combat or hero-power damage.

The value is the aggregate, at snapshot time, of the caster's friendly **`spell_damage:X` keywords** in the effective `keywords` view — intrinsic *or* aura-granted (the latter via `auraKeywords`). **Player-scoped spell-damage sources** (e.g. "+1 to all spells this turn", "+3 to your next spell") are deliberately **out of scope here** — they belong to the future Modifier System (see `docs/superpowers/notes/2026-05-31-modifier-system.md`), because their scoped/consumable lifecycle is shared machinery with mana-cost reduction and should be designed once, for both, rather than baked into spell damage.

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

### `ITargetSelector`

The **dual of `ITriggerCondition`**: a condition filters *events*, a selector produces *entities*. It is the single targeting primitive — one library answers "which entities does this effect act on?", "which targets may the player legally pick?", and "what are the options in a Discover/mid-effect choice?" There is no separate target-requirement taxonomy; legality is membership in a selector's output (below).

```csharp
interface ITargetSelector {
    EntityId[] Select(EffectContext context);  // pure; ordered; 0..N entities
}
```

`Select` is a **pure function of `GameState`** (read via the context) — no mutation, no RNG, no enqueueing. Like the condition library it evaluates *relative to* the source: "enemy" means "opponent of `context.sourcePlayerId`".

**Singletons — parameterless** (referenced by reference, no per-call allocation): `AllEnemyMinions`, `AllFriendlyMinions`, `AllNeutralMinions`, `AllMinions`, `AliveEnemyMinions`, `AliveFriendlyMinions`, `AliveNeutralMinions`, `AliveMinions`, `DeadEnemyMinions`, `DeadFriendlyMinions`, `DeadNeutralMinions`, `DeadMinions`, `EnemyHero`, `FriendlyHero`, `AllEnemyCharacters`, `AllFriendlyCharacters`, `Self`.

**Factories — parameterized:** `AdjacentTo(id)`, `MinionsWithTribe(Tribe)`, `MinionsWithKeyword(string)`, `OtherThan(selector, id)`, `Union(s1, s2, …)`, `Filter(selector, ITriggerCondition)` (reuse the condition library as the entity predicate — e.g. *"all damaged friendly minions"* = `Filter(AllFriendlyMinions, IsDamaged)`).

**`All…` / `Alive…` / `Dead…` (mortally-wounded inclusion).** A minion at `currentHealth ≤ 0` that is **still on the board** — not yet removed by death resolution — is **"mortally wounded"** (a.k.a. pending death). (This window is transient under any model but becomes a designer-meaningful state under the cascade-settle death model — Fireplace amendment B, §4 ⑦ — where such a minion can persist across same-cascade reactions.) The three on-board minion families partition exactly on this distinction — **`All… = Alive… ∪ Dead…`**, disjoint:

| Family | Returns | Use when |
|---|---|---|
| `AllEnemyMinions` / `AllFriendlyMinions` / `AllNeutralMinions` / `AllMinions` | **every on-board minion** of that side, **mortally-wounded included** | the effect should treat the doomed-but-present body as real — HS-faithful AoE and board-count effects ("deal 1 to all enemies", "damage equal to enemy minions") |
| `AliveEnemyMinions` / `AliveFriendlyMinions` / `AliveNeutralMinions` / `AliveMinions` | only minions with **`currentHealth > 0`** | the effect must **not** act on a corpse-in-waiting — e.g. "return a random alive enemy minion to hand", "buff your lowest-health alive minion" |
| `DeadEnemyMinions` / `DeadFriendlyMinions` / `DeadNeutralMinions` / `DeadMinions` | only the **mortally-wounded** (`currentHealth ≤ 0`, still on board, pending removal) | the effect targets the dying specifically — an **Intervention × lethal** save/finisher (amendment B, ref R1), "destroy all mortally-wounded enemies", counting the doomed |

Each `Alive…` singleton is exactly `Filter(<corresponding All…>, currentHealth > 0)` and each `Dead…` is `Filter(<corresponding All…>, currentHealth ≤ 0)`, surfaced as named singletons for ergonomics. **Heroes have no mortally-wounded state** — a 0-health hero ends the game at §4 ⑧ rather than lingering — so there is no `Alive`/`Dead` hero/character variant. **Graveyard minions are *not* covered here:** an already-removed minion (resurrection / "died this turn" pools) is not a board target — selecting from the graveyard to *summon* is a separate primitive, not an `ITargetSelector`.

**Ordering.** The returned list is **ordered by the canonical board order** already locked for death/trigger resolution (§4 ⑦): current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index, with heroes slotted by the selector's own definition. Ordered, not unordered, because sequential within-action application (per-target damage, `DamageTakenEvent` emission order, keyword pulls) is order-sensitive and determinism demands a fixed order. (Deaths are no longer interleaved per hit — they settle at ⑦, amendment B — but the per-target application order still must be fixed.) Selectors never invent a second ordering.

**Three consumption modes** — a selector's candidate set covers every targeting feature with no new mechanism:

| Mode | How the effect consumes `Select(context)` |
|---|---|
| **Auto-hit** (AoE) | Enqueue **one** action carrying the **`ITargetSelector` reference** (the same one-action-carries-selector pattern as Random-K below); the stage-④ handler evaluates it over the current board and applies the effect to every result, in returned order — e.g. Flamestrike is a *single* `DealDamageAction` carrying `AllEnemyMinions`. Pure: no RNG. **One action ⇒ one ③′ declaration ⇒ one interception/pre-damage window** for the whole AoE (with the targets as that window's selector), rather than N per-entity actions raising N windows (see §3 "Reactive…", "Batching"). The handler still emits one per-target event (e.g. a `DamageTakenEvent` each). |
| **Random-K** | Enqueue **one** action carrying the **`ITargetSelector` reference** (not a pre-computed pool) + `k`; the **stage-④ handler evaluates the selector against the *current* board, then draws K** using that action's epoch-derived `IRandom`. Carrying the selector rather than a frozen `EntityId[]` keeps selection both pure *and* current — if a candidate is mortally wounded or removed between enqueue and the draw, the ④ evaluation reflects it (and the card chooses the right `All…/Alive…/Dead…` family to decide whether the dying still count). Randomness is consumed only at ④, preserving the Item-6 invariant verbatim and keeping each draw attributable to one action's epoch (clean for the `StabilizationAbortReport`). |
| **Player-choice** (Discover, mid-effect targeting) | Issue `StartChoiceAction` with `options =` the candidate set → existing `PendingChoice` (§4). The selector *feeds* the choice; it does not subsume it — "compute targets" and "interrupt for player input" stay separate concerns. |

**Player-target legality is unified into the same primitive.** A card/effect requiring a player-supplied target declares a `(selector, cardinality)` pair. The §4 ③ validator's *"is a legal target type for this card"* check is then exactly: **`chosen ∈ selector.Select(context)`** (plus the cardinality/optionality rule) → `InvalidTarget` otherwise. The *same* `AllEnemyMinions` selector that an AoE uses to hit every enemy is what a single-target spell uses to define its legal targets — so previewed legality (client highlighting), validation, and effect resolution can never disagree. This replaces the spec's earlier hand-wave with one source of truth (parallel to how §4 ③ made the validator the single source of truth for action legality).

**JSON encoding** of selector references in card definitions is a `DefaultCardHandler` detail, *not* part of this spec — same treatment as the condition tree.

### Reactive Triggers, Interventions & Interception

This subsection unifies Secrets, the locked Intervention system, and the dying-window (amendment B ref R1) into **one** mechanism over `ITrigger`. Decided 2026-06-04 (Fireplace menu point D — the secrets/armed-reactive question); see the borrow-list note for the full derivation. Three ideas: **zone-scoped hosting**, **two hook phases**, and **interception as ordinary effects + re-resolution**.

**1 — A trigger is live by virtue of its host's zone (aura-like).** An `ITrigger` is registered by `ICardHandler` when its host enters a zone and unregistered when it leaves — exactly as a board minion's triggers register in `OnSummon` and drop on death. The same lifecycle simply runs on **other zones**:

| Host zone | Registered on | What a fired trigger does | Status |
|---|---|---|---|
| **board** (minion) | `OnSummon` (entering board) | ordinary reaction | existing |
| **hand** (card) | the `DrawCard`/`GiveCard` handler (entering hand), dropped on play/discard/return | opens a single-response **intervention** window for the owner (play it / skip) | **this amendment** |
| **secret zone** (card) | the play-to-secret handler | **auto-resolves** without a prompt (a Secret) | **deferred** (auto flavour + zone + visibility/cost — see end) |

All of this obeys the existing invariants unchanged: registration happens inside an action handler (④), so every reactive trigger has a `birthEpoch` and the creation-epoch filter (§3 `IEventBus`) stops a freshly-drawn card from reacting to the very event that drew it; the bus still carries only `ITrigger`. The hand is "armed" simply by holding the card — there is no separate arming step, and **what opens a window is precise and inspectable**: a card in hand whose live reactive trigger matched. This realises the locked Intervention scope ("play one card / skip") literally — the window only ever offers a card *designed* with a reactive trigger.

**2 — Two hook phases; that is the whole taxonomy.** Every action has a **pre** phase and **post** events:

- **Pre — the declaration phase (`ActionDeclaredEvent`, stage ③′).** A trigger here fires *before* the action resolves and may **intercept** it (prevent / alter). This is the held-action / suspend-resume kind.
- **Post — the regular events.** A trigger here reacts *after* the fact and cannot prevent — it just acts. This **includes `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`**: a "save" reaction on those is an ordinary post-reaction whose *before-finalization* timing is a gift of the cadence deferral (⑦ death / ⑧ win are settle stages — §4), **not** special machinery. "Consequence-deferral" is therefore not a third category; it is a post-reaction that exploits the deferral. **Post-reaction player windows batch**: matches accumulate through a cascade and open **one** window per (card, settle) before ⑦, with the matched set as the card's selector — so an AoE hitting N minions never opens N windows (see §4 "Batching"). Deterministic, no-choice reactions belong in the auto/secret flavour and open no window at all.

**3 — Interception is ordinary effects + re-resolution — there is no "disposition vocabulary."** An interception response runs as **normal effects** (summon, grant a keyword, gain armor, return-to-hand, deal damage — anything a card can do). The held action is then **re-resolved through full validation (②③)**, so almost every "alteration" emerges for free:

- grant Divine Shield to the defender → on resume the shield (a keyword pull at the damage moment) absorbs the hit;
- grant Poisonous to the defender → its resumed counter-damage destroys the attacker;
- gain armor / a standing damage modifier → read at the damage moment, no action edit;
- destroy / return / silence the attacker, or summon a **Taunt** token → on resume the held action **re-validates and either fizzles or is forced to the Taunt** by the ordinary rules.

Only **two** manipulations cannot be expressed as board-state mutation, and they are the *entire* irreducible surface — not a vocabulary:

- **cancel** — drop the held action (Counterspell: nothing on the board invalidates a legally-cast spell);
- **retarget** — substitute the held action's target *in place*, without re-declaring it (Misdirection's random redirect, Spellbender's spell redirect — not derivable from Taunt).

**Completeness rests on one invariant:** *anything a card can intercept is a discrete action* (so it has a declaration phase). Intrinsic, deterministic modifiers (the Divine Shield keyword, armor) are **pulled inline at resolution**, never declared — consistent with the `IKeyword` pull model (§3 `IKeyword`). So one interception point has **two realisations**: intrinsic *pulls* (hot-path, no window) and extrinsic *subscriptions* (secrets / interventions, possibly windowed).

**This subsumes damage prevention (menu point A — now closed).** Ice-Block-style "prevent the fatal hit, no side-effects" is an **interception on `DealDamageAction`** (`cancel`, or grant the target immunity, at its declaration) — *not* a `HeroMortallyWoundedEvent` post-reaction, which would let the damage (and its lifesteal/reflect side-effects) register before undoing the death. Three things make this universal and deterministic:

- **All damage flows through `DealDamageAction`, combat included.** `AttackAction` *enqueues* `DealDamageAction`s for the two combatants rather than applying damage inline, so the predamage declaration covers **every** source — Ice Block / redirect / prevention stops combat damage too, not just spells.
- **AoE damage is one declaration.** An auto-hit AoE is a *single* selector-carrying `DealDamageAction` (§3 `ITargetSelector`), so it raises **one** ③′ window with the targets as its selector — the pre-damage twin of post-reaction batching (one window for 7 minions), differing only in that it fires **immediately, before the hits** (pre-damage cannot defer to the settle). Per-target protection inside it is a grant (immune / Divine Shield) read at ④, not a new op.
- **④ damage-modifier precedence (pinned).** After ③′ interception, the handler computes each target's effective amount in a fixed order: **(1)** base `amount` (spell-damage bonus already baked at enqueue, §3 `IEffect`); **(2)** multiplicative modifiers (double-damage); **(3)** flat reductions, floored at 0; **(4)** caps; **(5)** **immune** → 0, stop (no shield break, no health loss); **(6)** **Divine Shield** → if still > 0, break it (`DivineShieldBrokenEvent`) and set 0; **(7)** apply to `currentHealth`, compute overkill, emit `DamageTakenEvent`, and if `≤ 0` emit `MinionMortallyWoundedEvent` (hero → `HeroMortallyWoundedEvent`). Only the baked spell-damage bonus (1) and Divine Shield / events (6–7) exist in v1; steps 2–4 are the slots future modifiers drop into, in this order, so stacking stays deterministic and replayable.

**Still deferred (game-feel, do not block this lock):** the **Secret** (auto-resolve) flavour and its zone + one-per-name/max/own-turn rules; **visibility** (a hand-live reactive trigger is *hidden* by default — MTG-instant "gotcha" vs. HS-secret telegraph); **cost timing** (free vs. pay-on-response); and the **marked-for-destruction scope** of dying-window saves (a destroy-marked minion cannot be healed/inverted back above 0). *(Batching is now resolved — see §4 "Batching".)* See the borrow-list note.

### `IAura`

Continuous effects recalculated after every board-affecting action. Aura bonuses are **never** stored in `enchantments` — they live in `auraAttackBonus`/`auraHealthBonus` and are fully rewritten on every recalculation pass.

```csharp
interface IAura {
    string AuraId { get; }
    string SourceMinionId { get; }
    IEnumerable<AuraEffect> Calculate(GameState state);
}

record AuraEffect(
    string TargetMinionId,
    int AttackBonus,
    int HealthBonus,
    Tribe GrantedTribes = Tribe.None,
    IReadOnlyList<string>? GrantedKeywords = null);  // DECLARATIVE keywords only — see below
```

A minion that reaches ≤0 `maxHealth` due to an aura loss is treated identically to a minion killed by damage.

Aura-granted tribes and keywords flow through `AuraEffect.GrantedTribes` / `GrantedKeywords`: each recalc pass, a minion's `auraTribes` is rewritten to the OR of all `GrantedTribes` targeting it, and its `auraKeywords` to the union of all `GrantedKeywords` targeting it (parallel to `auraAttackBonus`/`auraHealthBonus`) — so both appear/disappear with the aura and never persist.

**Aura recalc grants DECLARATIVE keywords only.** An *active* keyword (one whose `IKeyword.OnApplied`/`OnRemoved` register/unregister event-bus listeners — Lifesteal, Poisonous, Enrage, Windfury, Freeze) cannot be aura-granted, because subscribing/unsubscribing during recalc (stage ⑥) would violate the invariant that *all* subscriptions occur inside action handlers (stage ④ — see §3 `IEventBus` and Item 3). Recalc therefore only rewrites lists; it never touches the bus. Active keywords reach a minion only via `intrinsicKeywords` (subscribed in `OnSummon`, ④) or `grantedKeywords` (subscribed by the granting effect's handler, ④; unsubscribed by the Silence handler, ④). See "Unaddressed Features" for the deferred aura-granted-active-keyword case.

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

### `IRandom`

All in-match randomness flows through a single injected `IRandom`. No implementation may call `Random.Shared`, a clock-seeded RNG, or any other ambient source. This is the determinism contract that makes telemetry (`StabilizationAbortReport`, §4 ⑦) and replay possible.

```csharp
interface IRandom {
    int Next(int maxExclusive);                   // [0, maxExclusive)
    int Next(int minInclusive, int maxExclusive); // [minInclusive, maxExclusive)
}
```

**Seed.** `GameState.rngSeed: ulong` is fixed at match creation and never mutates. It is server-side state — the client is a pure event renderer and never receives `GameState`, so the seed is structurally hidden from both players and cannot be used to predict draws. Tests construct the engine with a chosen seed for repeatable scenarios.

**Per-action derivation (counter-based).** There is **no single long-lived PRNG** advanced across the match. Instead, at the start of each action's stage ④ the engine derives a *fresh* `IRandom` for that action from `mix(rngSeed, currentActionEpoch)` via a strong mixing function (e.g. splitmix64, so adjacent epochs yield uncorrelated streams). The handler draws from that instance as many times as it needs; draw order within the action is fixed by the deterministic pipeline ordering (below). Consequence: **any single action is independently reproducible from `(rngSeed, currentActionEpoch, preActionState)` alone** — no PRNG internal state is ever serialized. That is what makes `StabilizationAbortReport` self-contained (its `preActionState` snapshot already carries `currentActionEpoch`), and it reuses the very `currentActionEpoch` introduced for event visibility (§3 `IEventBus`). This is the counter-based / splittable-RNG model (cf. JAX key-split, NumPy `SeedSequence.spawn`, Slay the Spire's per-domain streams), chosen over a single advancing stream precisely so a mid-game slice can be reproduced without serializing RNG state.

**Where randomness is consumed.** Only inside action handlers (stage ④), the sole state-mutation point. An effect or trigger that needs a random outcome enqueues an action whose handler does the roll — e.g. `DiscardCardAction { cardId = null }` picks the discard via `IRandom`; Discover generates its options; a "random enemy minion" action resolves its target. (Target *selectors* are **pure and do not receive `IRandom`** — see §3 `ITargetSelector`. A random-target effect enqueues a single action carrying the selector's *pool*; the stage-④ handler draws K from it with that action's `IRandom`. So random target resolution is, like every other roll, a stage-④ concern.)

**Deck shuffle (at init).** The opening shuffle runs inside the engine at match setup as a Fisher-Yates over `IRandom`, seeded from the match seed like any other action. **Decklists are the stored input; the shuffled order regenerates from `rngSeed`.** The coin flip (first player) and opening-hand deal are seeded the same way.

**Replay.** The engine is a **deterministic function of `(rngSeed, ruleset version, decklists, ordered input log)`**. System actions, random rolls, and the entire event cascade regenerate; `currentActionEpoch` reconstructs itself by counting actions, so it need not be stored. (A mid-game *slice* repro such as the abort report does not replay from the start, so it relies on `currentActionEpoch` being carried in the captured `GameState`.) Four points make this precise:

- **An *input* is anything submitted via `Submit`** — not only deliberate player moves but also **timeout / forfeit / disconnect-injected actions** (auto-`EndTurn`, auto-skip, auto-resolve a `PendingChoice`). These are *external nondeterminism* — they depend on wall-clock time, not game state, so they are **not** regenerable and **must** appear in the input log like any other action. (The timer that injects them is a Game Server concern; GameLogic only requires that they enter through the same `Submit` seam and are logged.) "Player actions only" would be subtly wrong the first time a turn times out.
- **Single, total-order input stream** — one sequence per game, not per-player. The engine processes inputs in a single total order; per-player streams would reintroduce interleaving ambiguity.
- **Ruleset-version pinning** — a command-log replay re-runs engine logic + card `definition` JSON, so it is only valid against the **same engine + card-definition version** it was recorded under (the classic command-log-replay fragility across patches). The replay package carries a ruleset/content version stamp.
- **Command log vs. event log — two artifacts, two jobs.** The **command/input log is canonical**: compact, re-simulatable, the source of truth (but version-fragile per above). The **event log is the client wire format**, spectator/late-join stream, and version-robust archive: self-contained state deltas the client renders with **no game logic**, but *not* re-simulatable. (The event log is exactly the **handler-produced deltas**; transient bus signals — `ActionDeclaredEvent` — are excluded, as they carry no state change. See §2B "Bus ⊋ log".) The asymmetry: the **event log is derivable from the command log + seed, never the reverse** (events record outcomes, not the inputs/rolls that produced them) — so the command log is the richer artifact and the event log the safer/self-contained one.

### Deterministic Ordering

| Concern | Owner |
|---|---|
| Trigger fire order | `IEventBus` — current player first, then opponent; within each, sorted by current board index at publish time |
| Event visibility (new subscribers) | `IEventBus` — snapshot at publish + `birthEpoch < originEpoch` creation-epoch filter; a listener never receives events from the action that created it |
| Death wave phases | `DeathResolutionService` — Phase 1 (remove) → Phase 2 (deathrattles) → Phase 3 (reborns) |
| Death sort order | `DeathResolutionService` — current player board[0..n] by index, opponent board[0..n] by index, neutral zone by index |
| Deathrattle before Reborn | `DeathResolutionService` — Phase 3 runs only after all Phase 2 actions complete |
| New deaths during deathrattles | `DeathResolutionService` — deferred to next wave, not resolved mid-wave |
| Resolution cadence | `GameEngine` — one action at a time; ①–⑥ run per action, draining the queue; ⑦ death + ⑧ win fire **only when the queue empties** (cascade settled), then loop. Deaths are batched at the settle point, never mid-cascade — so triggered reactions resolve before removals (amendment B) |
| Interception (declaration window) | `GameEngine` — stage ③′: an action is suspended before ④ **at most once** (depth-1 cap); the response resolves fully (FIFO), then the held action re-validates and executes-or-fizzles. A single suspend/resume of one action — the only deviation from FIFO, never a general LIFO stack (point D) |
| Randomness | `IRandom` — per-action stream derived from `mix(rngSeed, currentActionEpoch)`; consumed only in stage ④ handlers; draw order fixed by the rows above |

---

## Section 4: Action Processing Pipeline

The pipeline is a **loop**, not a single linear pass (cadence amended 2026-06-03 — Fireplace amendment B). Stages **①–⑥ run per action** as each is dequeued; triggered reactions enqueue further actions, and the engine keeps dequeuing and running ①–⑥ until the **action queue is empty** — the point at which the cascade started by a top-level action has fully *settled*. Only then do the **settle stages** run: **⑦ Death Resolution** and **⑧ Win Check**. **⑨** drives the loop (dequeue next, or trigger the settle when the queue is empty). The engine never advances a stage until the current one completes.

Because deaths are processed only at the settle point, **every triggered reaction from the settling cascade resolves *before* any minion is removed**: a minion reduced to ≤0 HP (or destroy-marked) is **mortally wounded** — still on the board, still targetable and countable, still keyword-active — until the settle, not removed on the spot. This is the Hearthstone-faithful cadence; it replaced an earlier per-action model where ⑦ ran at the tail of *every* action (so a dying minion vanished before same-event reactions ran). See amendment B in the borrow-list note for the design rationale, the mechanics it unlocks (overkill, retaliation, simultaneous-death, count-the-doomed, Intervention × lethal, Inversion × lethal), and the worked example.

### ① Submit

Player action arrives via WebSocket, or system action is dequeued from `IActionQueue`. `Submit` returns a `SubmitResult` — `Accepted(events)` or `Rejected(ActionRejection)` (see §2C).

### ② Phase Guard

Rejects the action if it is not valid in the current `GamePhase`, returning `Rejected(WrongPhase)` with no state mutation (see §2C).

| Phase | Accepted actions |
|---|---|
| `Mulligan` | `SubmitMulliganAction` only |
| `InProgress` | All player-initiated actions |
| `PendingChoice` | `SubmitChoiceAction` from the waiting player only |
| `PendingIntervention` | `SubmitInterventionAction` from the responding player only |
| `Ended` | None |

System actions bypass Phase Guard — they are trusted internal commands.

### ③ Action Validator

Precondition checks. Returns `Rejected(ActionRejection)` with the matching code and no state mutation on failure (codes per §2C):

- Turn ownership: is this the active player? → `NotActivePlayer`
- Mana affordability: `effectiveCost ≤ player.mana` → `NotEnoughMana`
- Target validity: target is alive, is not stealthed (for opponent spells/attacks), and is a **member of the card/effect's declared target selector's output** (`chosen ∈ selector.Select(context)`, plus the cardinality/optionality rule — see §3 `ITargetSelector`) → `InvalidTarget` / `TargetStealthed`. The same selector defines the client's legal-target highlighting, so preview and validation cannot drift.
- Taunt enforcement: if opponent has Taunt minions on board, attacks must target one of them → `MustTargetTaunt`
- Attack eligibility: attacker is **alive** — `currentHealth > 0` and not destroy-marked (else `AttackerNotAlive`), `canAttack = true` (else `AttackerCannotAttack`), `isFrozen = false` (else `AttackerFrozen`), `attacksUsedThisTurn < attacksAllowedThisTurn` (else `AttackerExhausted`). The aliveness check is what makes a **mortally-wounded attacker fizzle on re-validation**: a "destroy / return the attacker" interception (§3) leaves the attacker ≤0 / destroy-marked but on board (death is deferred to ⑦), and the held attack, re-entering ③ on resume, is rejected here before it can swing.
- Board space: player board has fewer than 7 minions before summoning → `BoardFull`

**Validation is a pure, standalone-invokable predicate.** Stages ② and ③ together form a side-effect-free function `(action, state) → Ok | Rejection` (where `Rejection` is the `ActionRejection` of §2C) — no mutation, no dispatch (④), no events. It is the **single source of truth for legality**: `Submit` runs it before ④, and any "is this action legal?" / "what actions are legal?" query — a future `GetLegalActions(playerId)`, bot move generation, or client pre-flight (greying out unaffordable cards / illegal targets) — **must call this same predicate** rather than re-deriving legality. This guarantees previewed/enumerated legality can never drift from what `Submit` actually accepts. (The positive-enumeration API itself — `GetLegalActions`, via brute-force-then-filter over candidate actions — is deferred to the future bot-support epic; only the standalone-predicate seam is part of the core.)

### ③′ Declaration & Interception Window

A pre-execution checkpoint inserted between validation (③) and dispatch (④) — it does **not** renumber ④–⑨. (Decided 2026-06-04, Fireplace point D; the model lives in §3 "Reactive Triggers, Interventions & Interception".) For every action that reaches dispatch:

1. Publish **`ActionDeclaredEvent { action }`** — the universal pre-execution signal (one event type for all actions; subscribers filter on the carried action's type/params). Most actions have no matching reactive trigger and pass straight to ④.
2. If a reactive trigger fired and its host **auto-resolves** (a Secret — deferred flavour), its response resolves inline here (no halt). If the host is a **hand card** offering a player response, the engine **suspends** the action — stores it as `pendingIntervention.heldAction`, sets `phase = PendingIntervention`, and opens the single-response window (§4 "PendingIntervention Interruption").
3. **Depth-1 cap:** while an interception response is resolving, no further declaration windows open — a response cannot itself be intervened. (Auto-resolving reactions still chain in FIFO order; only *player windows* are capped.) This bounds the otherwise-MTG-style stack to a single hold.

A held action is **declared once**: when it resumes (after the response) it re-enters ③ for re-validation but is **not** re-published to ③′. The declaration phase is the only deviation from the otherwise-FIFO cadence — a single suspend/resume of one action, never a general LIFO stack. See §3 for what a response may do (ordinary effects + re-resolution; only `cancel`/`retarget` touch the held action directly).

### ④ Dispatch → `IActionHandler`

The registered handler for this action type is invoked. It mutates `GameState` and returns `events[]`. This is the sole point of state mutation in the pipeline. `GameEngine` increments `GameState.currentActionEpoch` as the action enters this stage; every event the handler returns is stamped with that `originEpoch`, and any subscription the handler registers (e.g. via `ICardHandler.OnSummon`) is stamped with the same value as its `birthEpoch` — see §3 `IEventBus`. The engine also derives this action's `IRandom` from `mix(GameState.rngSeed, currentActionEpoch)`; every random outcome the handler produces is drawn from that instance (see §3 `IRandom`).

### ⑤ Publish Events → `IEventBus`

Events are published in the order returned by the handler. For each event, `IEventBus` snapshots the subscriber lists at publish time and fires them filtered by creation epoch (`birthEpoch < originEpoch`, see §3 `IEventBus`): current player's list first (sorted by current board index at publish time), then opponent's list (same). The epoch filter means a listener never receives an event from the action that created it. Subscribers enqueue new actions via `IActionQueue` — they do not process them inline.

### ⑥ Aura Recalculation

All registered `IAura.Calculate(state)` implementations are run. `auraAttackBonus` and `auraHealthBonus` on every minion are zeroed and fully rewritten. A minion newly at ≤0 `maxHealth` **enters pending-death** (publishing `MinionMortallyWoundedEvent`, cause = aura loss) — but it is **not removed here**; like a damage-killed minion it lingers, mortally wounded, until the next settle (⑦).

Recalculation runs **per action** (⑥, while draining) — after any action that changes board composition, minion stats, or keywords — and also after each action processed in Phase 2 of the death wave and after Phase 3 reborn summons. Keeping aura recalc per-action (cheaper than deferring it) means mid-cascade reactions always read fresh stats; only *death* is deferred to the settle, not aura math.

### ⑦ Death Resolution

**Settle stage — runs only when the action queue has drained to empty** (the cascade started by a top-level action has fully settled), never mid-cascade. While the queue is non-empty the engine stays in ①–⑥, so every triggered reaction resolves first and a mortally-wounded minion remains on the board (targetable, countable, keyword-active — and savable by a dying-window intervention, amendment B ref R1) right up to this point. Then `DeathResolutionService` runs the wave loop:

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
  rngSeed:            ulong          // GameState.rngSeed; with preActionState.currentActionEpoch reproduces this action's RNG via mix(rngSeed, epoch) (see §3 IRandom)
  preActionState:     GameState      // full snapshot at the START of the triggering top-level action whose settle entered the runaway wave (carries currentActionEpoch) — the Build step
  triggeringAction:   GameAction     // the action submitted that began the runaway — the Script step
  wavesReached:       int            // = MaxDeathWaves
  cascadeTrace:       GameEvent[]     // ordered events from action start to abort — the Exact-trace Assert
}
```

A regression scenario built from a report asserts: submitting `triggeringAction` against `preActionState` (seeded with `rngSeed`) terminates at wave `MaxDeathWaves` with `StabilizationAbortedEvent` then `GameEndedEvent { NoContest }` — i.e. it aborts rather than hangs.

### ⑧ Win Condition Check

**Settle stage — runs immediately after ⑦**, at the same queue-empty settle point. If any hero has health ≤ 0: if both ≤ 0 the game is a draw, otherwise the other player wins. `GameEndedEvent` is published and `phase` is set to `Ended`.

Mirroring ⑦'s pending-death window: a hero that crosses to ≤0 **mid-cascade** publishes **`HeroMortallyWoundedEvent`** at that moment (from the damaging action's ④), but the loss is **not** finalized until this settle — so a post-reaction save (heal-above-0 / immune, §3) enqueued onto the still-draining queue resolves first, and ⑧ then re-reads actual hero health and finds it survived. This gives heroes the same dying-window the death wave gives minions; simultaneous mutual lethality still settles here as a draw.

### ⑨ Dequeue / Drive Loop

The loop driver. **If the action queue is non-empty**, pull the next action and return to ① (continue draining the cascade through ①–⑥). **If the queue is empty**, first open any **pending batched post-reaction windows** (one per reactive card with accumulated matches — §4 PendingIntervention "Batching"); a response enqueues actions, so the queue may refill and drain again before this settle completes. With no windows left to open, run the settle stages ⑦ (death wave) then ⑧ (win check); the death wave may enqueue actions (deathrattles, reborns), so if the queue is non-empty again, resume draining — otherwise the cascade is fully settled and the engine awaits the next player action. (A player-submitted action is just the next thing dequeued, starting a fresh cascade.) **Ordering at the settle: batched windows → ⑦ death → ⑧ win** — so a post-reaction save resolves before the death/win it guards against.

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
- **Pre-halt death rule (amendment B):** before halting, Death Resolution (⑦) runs to settle any pending deaths, so the player never chooses against a board where mortally-wounded minions still linger. (A choice is not keyed to the dying window, so there is no exception here — contrast `PendingIntervention` below.)
- `GamePhase` → `PendingChoice`
- Remaining effect actions (the continuation) are serialised into `PendingChoice.context`
- Pipeline halts — no further actions are processed until the choice resolves
- `SubmitChoiceAction` arrives → selected option is filled into the continuation → continuation actions re-enqueued → `GamePhase` → `InProgress`

### PendingIntervention Interruption

A single-response window, **opened by a live reactive trigger on a hand card** that matched (§3 "Reactive Triggers, Interventions & Interception") — never a general engine rule. It resolves one of two ways, by which phase the trigger hooked. (Throughout: `phase → PendingIntervention`; timeout timer starts; only `SubmitInterventionAction` from the responding player is accepted; timeout = an injected skip, logged as an input per Item 13; `phase → InProgress` at the end.)

**Declaration-hold (pre-phase — `heldAction` present).** Opened at ③′ when a hand trigger fires on `ActionDeclaredEvent`. The triggering action is suspended into `pendingIntervention.heldAction`.
- On **play**: the response card resolves first (full pipeline). It runs ordinary effects (summon, grant keyword, gain armor, …) and may `cancel` or `retarget` the held action. Then the held action **resumes, re-validated (②③)** against the post-response board: still legal → executes (④); now illegal (attacker dead/returned, target gone, forced to a Taunt) or `cancel`led → it **fizzles** — dropped, not executed (no `ActionRejection`; no client awaits it). It is **not** re-declared (③′ fires once — no redirect/secret loops).
- On **skip / timeout**: the held action resumes and re-validates directly.

**Post-reaction (post-phase — `heldAction` null).** A hand trigger that fires on a regular event (including the cadence-deferred `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`) does **not** halt per-event. Instead it **accumulates** into a per-card pending-response record `{ cardId, respondingPlayerId, matched: [...] }` — `matched` appending the event's subject each time the trigger fires during the cascade. **Windows open batched, at the settle** (see "Batching" below): the response resolves there, **before** ⑦/⑧, so a save (invert a mortally-wounded minion to `currentHealth > 0`, heal a hero above 0) lands before the death/win finalizes. On **skip / timeout** nothing happens and the settle proceeds. There is no held action to resume.
- **Pre-halt death rule + R1 exception (amendment B):** before halting for a choice/intervention, Death Resolution (⑦) settles pending deaths so the responder never acts against a board where mortally-wounded minions still linger — **except** for a post-reaction save keyed to that very window (R1), where the mortally-wounded minion must remain on board for the response to act on; deaths are **not** pre-resolved, and the wave simply does not collect a minion the response lifted back above 0.

**Batching — one window per (reactive card, settle), with the matched set as the selector.** A single AoE produces many sub-events (one selector-carrying `DealDamageAction` → many `DamageTakenEvent`s, one per target), so a per-event post-reaction window would halt N times. It does not. Post-reaction matches **accumulate** through the cascade; when the queue drains, the engine opens **at most one** `PendingIntervention` window per pending reactive card, **before ⑦**. The window's candidate set is that card's `matched` list **∩ its declared `(selector, cardinality)` family** (the family's `All…/Alive…/Dead…` choice decides whether a mortally-wounded match is offered — inclusive for a "save" card). The player picks `cardinality` targets from that set, or skips; `SubmitInterventionAction` carries the choice and is validated by the ordinary §4 ③ membership check (`chosen ∈ matched`). Example: an AoE damaging **7** friendly minions while the responder holds a single-use "heal one minion +2 on `DamageTakenEvent`" card → **one** window offering all 7 (the dying ones included), pick one to heal/save. *(Declaration-hold windows do not need accumulate-at-settle: each declared action is one `ActionDeclaredEvent`. A "counter/redirect the spell" hooks the one `PlayCardAction`; a **pre-damage** card hooks the AoE's **single selector-carrying `DealDamageAction`** (§3 `ITargetSelector` auto-hit) — one window for the whole AoE either way, fired immediately rather than deferred to the settle.)* If multiple reactive cards are pending at the same settle, their windows open in a deterministic order (responder — current player first — then the card's hand order). Only **player-choice windows** batch this way; **auto reactions** (secrets, board triggers) still fire inline per-event in FIFO. A deterministic reaction with no decision (e.g. "heal *it* for 2", no target choice) therefore belongs in the **auto/secret flavour**, where it simply enqueues one heal per match and opens no window at all.

---

## Unaddressed Features

Features the current architecture **deliberately does not support** and that are deferred **indefinitely** (no planned epic/ticket, unlike the "deferred to a future epic" items tracked in the borrow-list/plan). Each entry records the limitation, why it exists, and what enabling it would cost — so a future card requirement can reopen it with full context rather than rediscovering the constraint.

> **Note (keyword model collapse):** an earlier entry here — *"aura-granted active keywords"* — has been **removed**, not deferred. It existed because keywords were split into declarative (queried) and active (event-bus listeners), and an aura granting a listener-backed keyword would have had to subscribe during stage-⑥ recalc, violating the Item 3 invariant. The active/listener category was eliminated (§3 `IKeyword`): all keywords are declarative, pulled from the effective view at the minion's own action moments, with follow-on actions expressed as role-interface hooks. Aura-granted keywords now behave identically to intrinsic ones with no special mechanism, so the limitation no longer exists.

### Keyword effects on a dead source

A minion's **keyword hooks (Lifesteal, Poisonous, …) do not apply to damage its own Deathrattle deals after it has died.** The damage is still *attributed* to the dead minion (`sourceId` on the event, so watching triggers and friendly/enemy checks resolve correctly — see §3 "Source Attribution"), but the keyword *effects* don't fire.

- **Why:** keyword hooks read the acting minion's **effective** keywords from live board state (§3 `IKeyword`). A dead source has left the board (death-wave Phase 1) before its Deathrattle resolves (Phase 2), so `EffectiveKeywords(source)` is empty. No special-casing.
- **"Dead" here means *removed*, not *mortally wounded* (amendment B):** a minion at ≤0 HP that is still on the board (pending death, before the settle) is *fully keyword-active* — if it deals damage in that window (e.g. a dying-swing retaliation), its Lifesteal/Poisonous etc. *do* fire, because `EffectiveKeywords(source)` reads it from the live board. The limitation applies only once the death wave's Phase 1 has actually removed the source.
- **Cost to enable:** a death-snapshot fallback — when the source is dead, the relevant handler reads keywords off the source's `GraveyardMinion.snapshot` instead of live board state. A localized fallback in the affected handlers, not a parallel mechanism.
- **Status:** no card in scope needs it. Revisit only when one does.
