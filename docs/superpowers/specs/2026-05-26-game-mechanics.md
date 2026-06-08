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
neutralZone: MinionOnBoard[]          // ownerId = null; Taunt is per-LANE (a neutral Taunt forces attacks aimed INTO the neutral lane, but not attacks into the opponent lane — and vice versa; §4 ③); player auras do not apply
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone in this game mode
neutralGraveyard: GraveyardEntry[]    // single shared graveyard, owned by GameState (NOT per-player). A dead minion lands here iff bornNeutral && ownerId == null at death (system-spawned neutral that died in the neutral lane); see §4 ⑦ Phase 1 routing
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
minionId: string
definitionKey: string         // the SOLE link into the card-definition library — and to the ICardHandler that registers this minion's triggers/deathrattle/keyword-hooks (see note at end of block). NOT a Card.id: playing a card never transfers its id; tokens have a definitionKey but no Card.id (Identity section).
ownerId: string?              // null = neutral zone. A control effect (TakeControlAction) sets this to a player, moving the minion out of the neutral lane onto that player's board; it then dies to that player's graveyard.
bornNeutral: bool             // ORIGIN flag, NOT a zone read: true iff this minion was spawned-neutral by the system (SpawnNeutralMinionAction). Immutable; survives a control change. With ownerId, the SOLE input to the §4 ⑦ graveyard-routing rule. A player-card minion summoned INTO the neutral lane is NOT bornNeutral (deferred — see Unaddressed Features).
baseAttack, baseHealth: int   // immutable, from card definition
enchantments: StatModifier[]  // permanent buffs; Silence clears all
auraAttackBonus, auraHealthBonus: int  // recalculated each aura pass; never stored in enchantments
attack: int                   // = baseAttack + Σenchantments.attackDelta + auraAttackBonus
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus
currentHealth: int            // takes damage; healed up to maxHealth. ≤0 (or destroy-marked) ⇒ "mortally wounded" (pending death): the minion stays ON the board — still targetable, countable, and keyword-active — until the next death-wave settle (§4 ⑦), not removed on the spot (amendment B)
keywords: string[]            // maintained EFFECTIVE view = intrinsicKeywords ∪ grantedKeywords ∪ auraKeywords; resolved to IKeyword at runtime; what the engine queries. (Keyword analog of effectiveTribes.)
intrinsicKeywords: string[]   // from the card definition, seeded at summon; Silence clears
grantedKeywords: string[]     // PERMANENT non-aura grants after summon; retained across a RetainEnchantments bounce; Silence clears. Aura grants are NOT here.
auraKeywords: string[]        // aura-granted, recomputed each aura pass; never persisted, never bounce-retained; Silence does not touch (they recompute). ANY keyword may be aura-granted — keywords are pulled from the effective view, never a bus subscription (§3 IKeyword; keyword-collapse 2026-06-02).
intrinsicTribes: Tribe        // copied from the card's tribes at summon; immutable; survives Silence
grantedTribes: Tribe          // permanent tribe grants ("becomes a Beast"); Silence clears to Tribe.None
auraTribes: Tribe             // recomputed each aura pass; never persisted (parallels auraAttackBonus); drops when the aura leaves
effectiveTribes: Tribe        // computed = intrinsicTribes | grantedTribes | auraTribes; all engine tribe checks use this
isInverted: bool
summoningSick: bool           // raw fact (replaces the old derived `canAttack` bool): true while this minion entered a board THIS turn and has not yet woken at its controller's turn-start. Set on board-entry by BOTH summon and control; cleared by the turn-start sweep (Turn Lifecycle step 6). Attack eligibility — including the Charge-any-target / Rush-minions-only distinction — is PULLED from effective keywords at §4 ③, never baked into a stored verdict here. Neutrals are never summoningSick (no turn of their own; command bypasses eligibility and retaliation is ungated).
attacksUsedThisTurn: int      // raw counter, incremented on each attack; reset at the controller's turn-start sweep. The ONLY attack-state stored on the minion — the per-turn attack BUDGET is deliberately NOT a field: it is PULLED from effective keywords at §4 ③ (Windfury → 2, else 1; extensible to future budget-granting keywords), so it can never desync from the keyword (silencing Windfury drops the budget to 1 on the spot). NOT consulted for a neutral commanded via CommandAttackAction — command is a card-granted activation, not the minion's own budget (§2A / §4 ③).
summonOrder: int              // monotonically increasing per session; used for trigger fire ordering
isFrozen: bool                // frozen → cannot attack (§4 ③ AttackerFrozen)
frozenOnTurn: int             // turn.number when the freeze landed (meaningful only while isFrozen); drives the end-of-turn thaw rule (Turn Lifecycle step 3 / §3 IKeyword Freeze)
isDamaged: bool
rebornAvailable: bool         // the one-time Reborn charge — distinct from the "reborn" keyword tag, which persists. Initialized at summon to keywords.Contains("reborn"); the reborn-summon path summons with it false; consuming it (NOT the keyword) is what reborn does. See §4 ⑦.
// TRIGGERS — not stored as fields. This minion's ITrigger(s), Deathrattle, and keyword-hooks are registered into IEventBus by its ICardHandler (resolved via definitionKey) at summon (OnSummon, ④) and dropped when it leaves the board. Live by virtue of the board zone (§3 "Reactive Triggers, Interventions & Interception"). The bus carries only ITrigger; keywords are pulled from the effective `keywords` view (§3 IKeyword), not subscribed.
```

### Card

```
id, name: string
definitionKey: string         // the only cross-entity link into the card-definition library (Identity section); distinct from `id`, which is the per-instance allocator
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
// REACTIVE TRIGGERS — while this card sits in a trigger-hosting zone, its ICardHandler registers its reactive ITrigger(s) into IEventBus, live by zone and dropped on play/discard/return: in HAND → a player-choice intervention window; in the SECRET zone → auto-resolve. (Most cards have none and just sit inert in hand.) See §3 "Reactive Triggers, Interventions & Interception".
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
  turnPlayed: int
  // NO stored Card. The card form is FABRICATED on demand (the §1 Fabrication rule) at the
  // point of use — draw-from-graveyard, resurrect-as-card, re-equip, recast — minting a fresh
  // Card.id then, as a stage-④ action. (A pre-stored "originalCard" would be a pure function of
  // the subtype snapshot below — derived data that can only drift; eager minting would also burn
  // a Card.id per corpse that is never pulled.) Each subtype keeps its own ENTITY snapshot, the
  // single source of truth the fabrication reads.

GraveyardMinion : GraveyardEntry
  snapshot: MinionOnBoard   // full state at death; carries definitionKey + isInverted → card fully derivable
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  definitionKey: string     // a spell has no board snapshot, so it stores its own identity here
  isInverted: bool          // (its analogue of snapshot/weaponState) → card fabricated from these on recast

GraveyardWeapon : GraveyardEntry
  weaponState: WeaponOnHero // carries definitionKey → card derivable
  destroyedOnTurn: int
```

### Supporting types

```
WeaponOnHero        — definitionKey: string, attack: int, durability: int
HeroPower           — definitionKey: string, manaCost: int, definition: JsonElement, handlerKey?: string
TurnState           — activePlayerId: string, number: int
TimerState          — secondsRemaining: int
PendingChoice       — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[],
                       context: JsonElement  // stores effect continuation
PendingIntervention — respondingPlayerId: string, heldAction: GameAction?, candidateCardIds: string[],
                       timeoutSeconds: int
                       // SET-VALUED window (2026-06-09): candidateCardIds = the responder's matching + affordable
                       // reactive hand cards offered this window (§4). The player plays ONE (card + its targets) or
                       // skips. heldAction null = post-reaction (re-offer loop); non-null = declaration-hold
                       // (re-declare loop). The window re-opens after each PLAY (the offer shrinks), never after a skip.
MulliganState       — player1Completed: bool, player2Completed: bool
EffectContext       — sourceId: string, sourcePlayerId: string, targetId: string?,
                       state: GameState (read-only), SpellDamageBonus: int
                       // BASE resolution context — passed to IEffect.Execute, ITargetSelector.Select, and keyword pulls (§3). `sourceId` is the unified entity ref (minion OR card) and `sourcePlayerId` its controller, set by the enqueuer from the action's own source fields (§3 "Source Attribution"). Id-based precisely because the source may be a minion, not a card.
CardPlayContext : EffectContext — + card: Card
                       // SUPERSET for ICardHandler.OnPlay only: adds the resolved Card the handler reads its `definition` from. Here card.id == sourceId and playerId == sourcePlayerId; it inherits targetId/state/SpellDamageBonus, so OnPlay can pass itself to the IEffects it dispatches and bake spell-damage at enqueue.
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
- A `GraveyardMinion` stores **no card** — only its `snapshot` (full death state, including `definitionKey` + `isInverted`). Lineage-aware effects derive everything from it: "resummon the last minion that died" uses `snapshot.definitionKey` (a *new* `minionId` is allocated); "add the dead minion's *card* to hand" / resurrect-as-card **fabricates** a Card on demand from `snapshot` via the Fabrication rule above (minting a fresh `Card.id` at that moment). This holds uniformly for tokens and played-from-hand minions alike — `MinionOnBoard` never retains its originating Card, so the card form is always a fabrication, never a stored reference.

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
| `attacksUsedThisTurn`, `summoningSick`, `isFrozen`, `summonOrder` | Runtime board state with no Card-side equivalent. |
| Base keywords from the card definition | Re-derived on next play from the definition library, not carried on the Card. |

**Replay semantics:** when the fabricated Card is played again, the resulting minion inherits `grantedKeywords` from the Card into both `keywords` and `grantedKeywords` on the new minion, and inherits `modifiers` into the new minion's `enchantments`. `isInverted` is copied through unchanged. The new minion always has a fresh `minionId`.

**Scope note — keyword grant tracking:** `grantedKeywords` only captures non-aura buffs (effects that mutate the minion permanently). Aura-applied keywords are never added to `grantedKeywords`; they live only in `keywords` and are rewritten on every aura recalculation. Silence clears both `keywords` and `grantedKeywords`. This design avoids any per-source ledger in v1; if a future card requires per-source attribution (e.g. "remove only the Taunt that Sergeant gave," not the base Taunt), the model must extend `grantedKeywords` to `KeywordGrant { sourceId, keyword }`.

---

## Section 2: Actions + Events

Actions are immutable commands (input layer). Events are immutable facts (output layer). Every state change is produced by an action and announced via one or more events. The client receives events only — never raw state diffs. Both are C# records.

### 2A. GameAction types

All actions carry a nullable `SourcePlayerId` (null = system-issued) and a `RequestedAt` timestamp. The engine infers entity type (minion vs hero vs card) from the ID at processing time — actions do not carry type discriminators.

**ID convention:** a `cardId` / `minionId` / `targetId` field names an **existing instance** in a zone (a `Card.id`, a `MinionOnBoard.minionId`, …); a `definitionKey` field names a **library key to fabricate or summon from**, where no instance exists yet (summoning a token, generating a card into hand, equipping a weapon, transforming). The two are never interchangeable — see "Card and Minion Identity" (§1).

**Player-initiated:**

| Action | Key fields |
|---|---|
| `PlayCardAction` | playerId, cardId, targetId? |
| `AttackAction` | attackerId, targetId. **Handler (④), after ③ re-validation:** (1) snapshot the attacker's effective **base attack** (HS-style attack-lock; damage *modifiers* are pulled later, per `DealDamageAction` ④); (2) **consume the activation** — `attacksUsedThisTurn++`; (3) **break Stealth** if the attacker has it — an own-moment of *attacking*: clear the Stealth keyword and emit `StealthBrokenEvent` (attacking is what drops Stealth; merely being targeted/dealing non-attack damage does not — closes the Stealth half of hole #4); (4) emit `AttackPerformedEvent { attackerId, targetId }` (the post-interception **actual** target); (5) **enqueue two independent `DealDamageAction`s** — the strike (attacker → target, amount = the snapshotted attack) and, iff the defender's attack > 0, the retaliation (defender → attacker) — processed **sequentially, not atomically** (§3). **Freeze interaction:** a frozen attacker never reaches ④ (rejected at §4 ③ `AttackerFrozen`); conversely the `attacksUsedThisTurn` consumed in step 2 feeds the end-of-turn unfreeze sweep (Turn Lifecycle step 3) — a character frozen **this turn while exhausted** (it had already swung) **stays** frozen through its next turn, one frozen while it **still had an attack thaws** at the end of this turn, so Freeze costs exactly one attack. |
| `UseHeroPowerAction` | playerId, targetId? |
| `EndTurnAction` | playerId |
| `SubmitMulliganAction` | playerId, cardIdsToKeep[] |
| `SubmitChoiceAction` | playerId, choiceId, selectedOptionId |
| `SubmitInterventionAction` | playerId, chosenCardId? (null = **skip** — ends the window loop), targetIds[]? — plays **one** card from the window's offer (`pendingIntervention.candidateCardIds`) plus that card's chosen targets (from its matched-subject set, cardinality-many; validated by the §4 ③ membership check `chosenCardId ∈ candidateCardIds`, `targetIds ⊆ matched`). Accepted only from the responding player during `PendingIntervention`; **exempt** from the ③ turn-ownership check, and its cost is paid from the responder's **current `mana`** (the ordinary `effectiveCost ≤ mana` check — that pool is their next-turn mana, pre-loaded at the previous turn-end; see Turn Lifecycle). A **play** re-opens the window over the remaining offer; a **skip** ends it (§4 "PendingIntervention Interruption"). |
| `SurrenderAction` | playerId |

**System/effect-issued** — emitted by `IEffect`/`ITrigger` implementations; processed through the same pipeline as player actions:

| Action | Key fields |
|---|---|
| `DrawCardAction` | playerId, sourceId? |
| `DealDamageAction` | target: `entityId` **or** `ITargetSelector` (AoE — evaluated at ④ over the current board), amount, sourceId — **the one channel for all damage, combat included** (see §3 "Reactive…"). A selector-carrying instance is a single declared action → a single ③′ interception window. **Combat = two independent instances** (attacker strike + defender retaliation), each separately declared and processed sequentially — *not* one atomic unit (§3, 2026-06-06) |
| `HealAction` | targetId, amount, sourceId |
| `DestroyMinionAction` | minionId, sourceId? |
| `SummonMinionAction` | definitionKey, ownerId?, boardPosition, sourceId? |
| `EquipWeaponAction` | playerId, definitionKey, sourceId? |
| `DestroyWeaponAction` | playerId, sourceId? |
| `GiveCardAction` | playerId, definitionKey, sourceId? |
| `ReturnToHandAction` | minionId, policy: MinionToCardPolicy, sourceId? |
| `TransformMinionAction` | minionId, newDefinitionKey, sourceId? |
| `TakeControlAction` | minionId, newOwnerId, boardPosition?, sourceId? — **CONTROL** (permanent): re-homes the minion onto `newOwnerId`'s board and sets `ownerId`. The card's `ITargetSelector` defines legal `minionId` (neutral-only = `AllNeutralMinions`, enemy-only = `AllEnemyMinions`, either = `Union(...)`; no new selectors). Handler: reject `BoardFull` if the destination is full; else set `ownerId`, move zones, mark the minion **summoning-sick** (`summoningSick = true`) — whether it may act this turn is the ordinary pulled eligibility (Charge → any target, Rush → minions only) at §4 ③, *not* decided here — **re-bucket** its existing trigger subscriptions under the new owner (the bus orders firing by each host's owner vs. the active player, so however it tracks ownership it must now reflect `newOwnerId`) — this is **not** a re-run of `OnSummon`: triggers are neither re-created nor re-fired, and their `birthEpoch` is **preserved** (a moved entity, not a new one) — aura-recalc, emit `MinionControlChangedEvent`. The minion is now non-neutral and dies to `newOwnerId`'s graveyard. |
| `CommandAttackAction` | attackerId (a neutral), targetId, sourceId — **COMMAND** (one-shot, no zone change): a card-granted single attack *activation* by a neutral minion. Effect-issued. Re-validates at §4 ③ with the **relaxed attacker rule** (a neutral may attack only via this action; never via player `AttackAction`); legal `targetId` = anything a controlled minion could attack (opponent characters incl. hero, or another neutral) under the same **per-lane Taunt** rule. Honors the attacker's keywords: a **Windfury** neutral makes **two** strikes at the one target, **sequenced independently** — the second re-validates, so a mortal wound from the first retaliation fizzles it (§4 ③ aliveness). Ignores `attacksUsedThisTurn` and bypasses summoning-sickness entirely (neutrals are never `summoningSick`) — command is a card-granted activation, not the minion's own budget. Desugars into the same combat `DealDamageAction` pair(s) as `AttackAction`, so retaliation lands normally. |
| `InvertTargetAction` | targetId, sourceId? |
| `UnInvertTargetAction` | targetId, sourceId? |
| `BuffMinionAction` | minionId, attackDelta, healthDelta, sourceId |
| `SilenceMinionAction` | minionId, sourceId? |
| `FreezeTargetAction` | targetId, sourceId? — sets `isFrozen = true` and stamps `frozenOnTurn = turn.number` on the target; emits `MinionFrozenEvent`. Thawing is the end-of-turn unfreeze sweep (Turn Lifecycle step 3), never here. (Freezing a **hero** needs the same `isFrozen`/`frozenOnTurn` pair on `PlayerState` — deferred with the hero-combat pass; see Unaddressed Features.) |
| `ModifyManaAction` | playerId, delta, sourceId? |
| `ShuffleDeckAction` | playerId, sourceId? |
| `DiscardCardAction` | playerId, cardId? (null = random), sourceId? |
| `SpawnNeutralMinionAction` | definitionKey, position?, sourceId? |
| `StartChoiceAction` | waitingPlayerId, choiceType, options[], context |
| `StartInterventionAction` | respondingPlayerId, heldAction? (null = post-reaction window — nothing held, e.g. a dying-window save), candidateCardIds[] (the responder's matching + affordable hand cards offered this window), timeoutSeconds |

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
| `MinionMortallyWoundedEvent` | minionId, sourceId, cause — fires the instant a minion enters pending-death (currentHealth ≤0 from damage, maxHealth ≤0 from aura loss, or a destroy-mark), **before** the death-wave settle removes it; distinct from `MinionDiedEvent` (Phase-1 removal). The hook for dying-window reactions — e.g. a **save-from-lethal** intervention, where a player responds to lethal damage to rescue the minion before the settle removes it (the "R1" design avenue in the borrow-list note's amendment B). |
| `MinionDiedEvent` | minionId, snapshot, sourceId, diedOnTurn |
| `MinionTransformedEvent` | minionId, newCard |
| `MinionControlChangedEvent` | minionId, fromOwnerId? (null = was neutral), toOwnerId — emitted by `TakeControlAction` after the minion is re-homed onto its new owner's board (summoning-sick on arrival; it may still act this turn if it has Charge/Rush — pulled at §4 ③). A commanded attack (`CommandAttackAction`) does **not** emit this — it changes no owner — it rides the ordinary combat events. |
| `NeutralMinionSpawnedEvent` | minion |
| `NeutralZoneRepopulatedEvent` | spawnedMinions[] |
| `DeathWaveStartedEvent` | waveIndex |
| `DeathWaveEndedEvent` | waveIndex, minionsResolved |
| `StabilizationAbortedEvent` | wavesReached, lastWaveMinionIds[] — fatal; always immediately followed by `GameEndedEvent { reason: NoContest }` |

**Combat:**

| Event | Key fields |
|---|---|
| `AttackDeclaredEvent` | attackerId, targetId — the combat-specific **declaration**, published at `AttackAction` **③′** (the *intended*, pre-interception target): renderer telegraph + the typed hook for "when this attacks" interception (e.g. a redirect). May still fizzle/retarget before ④. |
| `AttackPerformedEvent` | attackerId, targetId — the **committed** swing, emitted at `AttackAction` **④** (the post-interception *actual* target). Announces the activation-consumed delta (`attacksUsedThisTurn++`) and is the natural anchor for a future "after this attacks" trigger. The swing's **damage is not here** — it lands via the two enqueued `DealDamageAction`s' own `DamageTakenEvent`s. **Replaces the removed `AttackResolvedEvent`**, which bundled both hit amounts into a single delta — impossible under two-action combat, where neither hit has landed at ④ and a card may interleave between them. |
| `DamageTakenEvent` | targetId, amount, overkill, sourceId |
| `HealedEvent` | targetId, amount, sourceId |

**Status / Keywords:**

| Event | Key fields |
|---|---|
| `MinionSilencedEvent` | minionId |
| `MinionFrozenEvent` | minionId |
| `MinionThawedEvent` | minionId — frozen status cleared by the end-of-turn unfreeze sweep (Turn Lifecycle step 3). |
| `MinionInvertedEvent` | minionId, isInverted |
| `CardInvertedEvent` | cardId, playerId, isInverted |
| `DivineShieldBrokenEvent` | minionId |
| `StealthBrokenEvent` | minionId — Stealth keyword cleared because the minion attacked (`AttackAction` ④). Parallels `DivineShieldBrokenEvent` (a one-time keyword consumption). |
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
    AttackerNotAlive, AttackerNotControlled, AttackerCannotAttack, RushCannotTargetHero, AttackerFrozen, AttackerExhausted, BoardFull,
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
| Stealth | two reads: (1) targeting validation — cannot be targeted by the opponent (`TargetStealthed`); (2) **broken when the minion attacks** — an own-moment cleared at `AttackAction` ④ (`StealthBrokenEvent`), like Divine Shield's one-time consumption. Being targeted or dealing non-attack damage does not break it. |
| Charge / Rush | pulled at attack eligibility (§4 ③) for a `summoningSick` minion: **Charge** → may attack any target (minion or hero); **Rush** → may attack a **minion only** (enemy or neutral) on the entry turn, never a hero. Not baked into a stored bool — pulling at the moment makes aura-granted or silenced haste correct with no recompute. |
| Windfury | pulled at attack eligibility (§4 ③): the per-turn attack **budget** = `effectiveKeywords has windfury ? 2 : 1` (extensible — e.g. a future Mega-Windfury → 4). Never stored on the minion, so it can't drift from the keyword; the eligibility check is `attacksUsedThisTurn < budget`. |
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

Further "own-moment" hook points (on-take-damage, on-attack, …) are added as new role interfaces when a card first needs one; each is invoked the same way by its owning handler. **Freeze** is not a keyword hook — it is a status: `FreezeTargetAction` sets `isFrozen = true` and stamps `frozenOnTurn = turn.number` (`MinionFrozenEvent`), and a frozen character cannot attack (§4 ③ `AttackerFrozen`). It thaws in the **end-of-turn unfreeze sweep** (§4 Turn Lifecycle step 3), which thaws the ending player's frozen characters **unless** the freeze landed *this* turn while the character was already exhausted — so Freeze costs **exactly one attack**: a character frozen with an attack still available misses that turn and thaws at its end, while one frozen *after* it has swung (e.g. into a retaliation-freezer) stays frozen through its next turn.

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

**Control / command targeting needs no new selectors.** A "take control of any neutral or enemy minion" card declares `Union(AllNeutralMinions, AllEnemyMinions)`; a neutral-only or enemy-only variant uses the corresponding singleton; "command a neutral to attack another neutral" uses `OtherThan(AllNeutralMinions, attackerId)`, and "command into the opponent lane" uses `Union(AllEnemyMinions, EnemyHero)`. Per-lane Taunt is **not** encoded in these selectors — it is enforced uniformly by the §4 ③ validator on top of the card's declared set, exactly as for a normal `AttackAction`.

**`All…` / `Alive…` / `Dead…` (mortally-wounded inclusion).** A minion at `currentHealth ≤ 0` that is **still on the board** — not yet removed by death resolution — is **"mortally wounded"** (a.k.a. pending death). (This window is transient under any model but becomes a designer-meaningful state under the cascade-settle death model — Fireplace amendment B, §4 ⑦ — where such a minion can persist across same-cascade reactions.) The three on-board minion families partition exactly on this distinction — **`All… = Alive… ∪ Dead…`**, disjoint:

| Family | Returns | Use when |
|---|---|---|
| `AllEnemyMinions` / `AllFriendlyMinions` / `AllNeutralMinions` / `AllMinions` | **every on-board minion** of that side, **mortally-wounded included** | the effect should treat the doomed-but-present body as real — HS-faithful AoE and board-count effects ("deal 1 to all enemies", "damage equal to enemy minions") |
| `AliveEnemyMinions` / `AliveFriendlyMinions` / `AliveNeutralMinions` / `AliveMinions` | only minions with **`currentHealth > 0`** | the effect must **not** act on a corpse-in-waiting — e.g. "return a random alive enemy minion to hand", "buff your lowest-health alive minion" |
| `DeadEnemyMinions` / `DeadFriendlyMinions` / `DeadNeutralMinions` / `DeadMinions` | only the **mortally-wounded** (`currentHealth ≤ 0`, still on board, pending removal) | the effect targets the dying specifically — a **save-from-lethal** intervention or finisher, responding to lethal damage to rescue (or pick off) a dying minion (the "R1" avenue, amendment B), "destroy all mortally-wounded enemies", counting the doomed |

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

This subsection unifies Secrets, the locked Intervention system, and the dying-window — the **save-from-lethal** beat, where a player responds to lethal damage to rescue a minion before it dies (the "R1" avenue in amendment B) — into **one** mechanism over `ITrigger`. Decided 2026-06-04 (Fireplace menu point D — the secrets/armed-reactive question); see the borrow-list note for the full derivation. Three ideas: **zone-scoped hosting**, **two hook phases**, and **interception as ordinary effects + re-resolution**.

**1 — A trigger is live by virtue of its host's zone (aura-like).** An `ITrigger` is registered by `ICardHandler` when its host enters a zone and unregistered when it leaves — exactly as a board minion's triggers register in `OnSummon` and drop on death. The same lifecycle simply runs on **other zones**:

| Host zone | Registered on | What a fired trigger does | Status |
|---|---|---|---|
| **board** (minion) | `OnSummon` (entering board) | ordinary reaction | existing |
| **hand** (card) | the `DrawCard`/`GiveCard` handler (entering hand), dropped on play/discard/return | opens a **set-valued, looping intervention** window for the owner (play one / skip; re-opens on each play) | **this amendment** |
| **secret zone** (card) | the play-to-secret handler | **auto-resolves** without a prompt (a Secret) | **deferred** (auto flavour + zone + visibility/cost — see end) |

All of this obeys the existing invariants unchanged: registration happens inside an action handler (④), so every reactive trigger has a `birthEpoch` and the creation-epoch filter (§3 `IEventBus`) stops a freshly-drawn card from reacting to the very event that drew it; the bus still carries only `ITrigger`. The hand is "armed" simply by holding the card — there is no separate arming step, and **what opens a window is precise and inspectable**: a card in hand whose live reactive trigger matched. This realises the locked Intervention scope ("play one card / skip") literally — the window only ever offers a card *designed* with a reactive trigger.

**2 — Two hook phases; that is the whole taxonomy.** Every action has a **pre** phase and **post** events:

- **Pre — the declaration phase (`ActionDeclaredEvent`, stage ③′).** A trigger here fires *before* the action resolves and may **intercept** it (prevent / alter). This is the held-action / suspend-resume kind. It fires **immediately, per action, mid-cascade** — there is no other coherent timing, since interception must precede that action's ④. A response does not end the matter: if it **plays** a card, the held action re-validates (②③) and — if still legal — **re-declares (③′)**, re-opening the window over the responder's *remaining* matching + affordable cards; the loop runs until the responder **skips** (the held action proceeds to ④) or holds nothing affordable, or the action **fizzles** (the **re-declare loop**, 2026-06-09). Multiple held interventions on one declaration are thus the responder chaining their *own* cards, in an order they set by which they pick each pass — **not** a nested stack (depth-1 still bars a further *player window* on a response; an auto/secret host may still fire — and even intercept — inline; see §4 ③′).
- **Post — the regular events.** A trigger here reacts *after* the fact and cannot prevent — it just acts. This **includes `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`**: a "save" reaction on those is an ordinary post-reaction whose *before-finalization* timing is a gift of the cadence deferral (⑦ death / ⑧ win are settle stages — §4), **not** special machinery. "Consequence-deferral" is therefore not a third category; it is a post-reaction that exploits the deferral. **Post-reaction player windows are set-valued and re-offer on play (stage ⑥′)**: after a *single* action's ⑤/⑥, the engine opens **one** window presenting *all* the responder's matching + affordable hand cards; the player plays **one** (choosing it and its targets) or **skips**. A played card's targets come from its *own* matched-subject set, so an AoE (one selector-carrying action hitting N minions) offers that one card over all N subjects — **one** window, not N. Each **play** re-offers the *remaining* matching set against the same (frozen) events (the **re-offer loop**, 2026-06-09); a **skip** ends the window (only a play changes the offer). Two *separate* damage actions remain two distinct instances → two windows. The triggering unit is one action's events — **never per-event, never per-cascade** (see §4 "Batching"). Deterministic, no-choice reactions belong in the auto/secret flavour and open no window at all.

**3 — Interception is ordinary effects + re-resolution — there is no "disposition vocabulary."** An interception response runs as **normal effects** (summon, grant a keyword, gain armor, return-to-hand, deal damage — anything a card can do). The held action is then **re-resolved through full validation (②③)**, so almost every "alteration" emerges for free:

- grant Divine Shield to the defender → on resume the shield (a keyword pull at the damage moment) absorbs the hit;
- grant Poisonous to the defender → its resumed counter-damage destroys the attacker;
- gain armor / a standing damage modifier → read at the damage moment, no action edit (note: hero `armor` itself is not yet in the §1 model — deferred to the hero-combat pass; see Unaddressed Features, "Hero armor");
- destroy / return / silence the attacker, or summon a **Taunt** token → on resume the held action **re-validates and either fizzles or is forced to the Taunt** by the ordinary rules.

Only **two** manipulations cannot be expressed as board-state mutation, and they are the *entire* irreducible surface — not a vocabulary:

- **cancel** — drop the held action (Counterspell: nothing on the board invalidates a legally-cast spell);
- **retarget** — substitute the held action's target *in place*, without re-declaring it (Misdirection's random redirect, Spellbender's spell redirect — not derivable from Taunt).

**Completeness rests on one invariant:** *anything a card can intercept is a discrete action* (so it has a declaration phase). Intrinsic, deterministic modifiers (the Divine Shield keyword, armor) are **pulled inline at resolution**, never declared — consistent with the `IKeyword` pull model (§3 `IKeyword`). So one interception point has **two realisations**: intrinsic *pulls* (hot-path, no window) and extrinsic *subscriptions* (secrets / interventions, possibly windowed).

**This subsumes damage prevention (menu point A — now closed).** Ice-Block-style "prevent the fatal hit, no side-effects" is an **interception on `DealDamageAction`** (`cancel`, or grant the target immunity, at its declaration) — *not* a `HeroMortallyWoundedEvent` post-reaction, which would let the damage (and its lifesteal/reflect side-effects) register before undoing the death. Three things make this universal and deterministic:

- **All damage flows through `DealDamageAction`, combat included.** `AttackAction` *enqueues* `DealDamageAction`s for the two combatants rather than applying damage inline, so the predamage declaration covers **every** source — Ice Block / redirect / prevention stops combat damage too, not just spells. **Combat is two independent damage actions, not one atomic unit (decided 2026-06-06).** The attacker's strike and the defender's retaliation are each first-class actions — each with its own ③′ declaration and its own ⑥′ window — processed **strictly sequentially** through the queue. This is the §4 ⑥′ rule applied verbatim ("one window per action; two instances = two windows"); an atomic single-window combat would be an *exception* to that rule, so it was rejected. Consequences: **(a)** interception/reaction windows are **per recipient** — the natural granularity, since Ice Block / Divine Shield protect one entity, not "a combat" — and a held card may interleave **between** the two hits (e.g. bounce the defender to dodge its retaliation); **(b)** a same-combat reaction sees **per-hit** board state (the defender's "when damaged" reaction fires *before* the retaliation lands) — a deliberate, narrow divergence from HS's apply-both-then-react; **(c)** **base attack is snapshotted at `AttackAction` ④** (HS-style attack lock), while damage *modifiers* are pulled at each damage action's ④ (precedence below). Because death is deferred to ⑦, **both combatants are mortally-wounded-but-on-board until the queue drains, so they die in one shared wave**, and a lethal-but-not-removed defender **still deals its retaliation with full effect** (keywords + source-side modifiers pulled live). This makes **dying-swing retaliation** — Poisonous/Lifesteal on a dying defender, or "retaliation doubled while mortally wounded" — a supported mechanic *by construction*, not special machinery.
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
    IReadOnlyList<string>? GrantedKeywords = null);  // any keyword — pulled from the effective view, never a bus subscription (§3 IKeyword); see below
```

A minion that reaches ≤0 `maxHealth` due to an aura loss is treated identically to a minion killed by damage.

Aura-granted tribes and keywords flow through `AuraEffect.GrantedTribes` / `GrantedKeywords`: each recalc pass, a minion's `auraTribes` is rewritten to the OR of all `GrantedTribes` targeting it, and its `auraKeywords` to the union of all `GrantedKeywords` targeting it (parallel to `auraAttackBonus`/`auraHealthBonus`) — so both appear/disappear with the aura and never persist.

**Aura recalc only rewrites lists; it never touches the bus.** Keywords are declarative markers pulled from the effective view at the minion's own action moments (§3 `IKeyword`) — no keyword subscribes to the bus — so an aura may grant **any** keyword, with no registration-during-recalc problem, and the grant disappears when the aura leaves. (Keyword-model collapse, 2026-06-02: the earlier active/declarative split — and its restriction that an *active* keyword could not be aura-granted because it would have to subscribe during stage ⑥ — was removed; see §3 `IKeyword` and the removal note under "Unaddressed Features".)

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
| Interception (declaration window) | `GameEngine` — stage ③′: an action suspends before ④; the response resolves **immediately** and fully (board/auto triggers fire inline; the **depth-1 cap** bars only a further *player window* on a response — *nesting* — while auto/secret hosts still fire, and may intercept, inline), then the held action re-validates and, if still legal, **re-declares (③′)** — re-opening the **set-valued** window over the responder's remaining matching+affordable cards until they **skip** or the action fizzles/executes. **Iteration** (one responder chaining their own cards on one action) is bounded by next-turn mana, not capped; **nesting** is capped at depth-1. The only deviation from FIFO (point D) |
| Post-reaction window | `GameEngine` — stage ⑥′: after each action's ⑤/⑥, **one set-valued window** offering all the responder's matching + affordable reactive hand cards; play one (card + targets from its matched set) or **skip**; a **play** re-offers the remaining set, a **skip** ends it. Response resolves immediately before the queue resumes; deaths **not** pre-settled. Per action — never per-event, never per-cascade |
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
- Attacker control: the submitter may attack only with a minion **it controls** — `attacker.ownerId == submitter` → else `AttackerNotControlled`. This is an explicit check (it closes the gap where the old "is this the active player?" line let a player swing with an opponent-owned minion), and it means **neutrals are never default attackers** (`ownerId == null` controls for no one). The **sole exception** is `CommandAttackAction`, a card-granted effect whose validation permits a *neutral* `attackerId` (the relaxed attacker rule); it still runs every other check below.
- Taunt enforcement — **per lane** (amended 2026-06-07): the constraint is scoped to the **target's lane**. Attacking into the **opponent lane** → if that opponent has Taunt minions *on their own board*, the attack must target one (else `MustTargetTaunt`); face is legal when none. Attacking into the **neutral lane** → if a neutral Taunt is present, the attack must target it. **Cross-lane Taunt never applies** — a neutral Taunt does not lock you out of the opponent's hero, nor an opponent Taunt out of a neutral. (Supersedes the earlier "Taunt is ignored for neutral minions" scope; neutral Taunt is now honored, but only within the neutral lane.) The same rule governs `CommandAttackAction`.
- Attack eligibility: the attacker is **alive** — `currentHealth > 0` and not destroy-marked (else `AttackerNotAlive`) — not frozen (`isFrozen = false`, else `AttackerFrozen`), and has an activation left (`attacksUsedThisTurn < budget`, else `AttackerExhausted`, where the per-turn **budget** is *pulled* from effective keywords — Windfury → 2, else 1 — never a stored field, so silencing Windfury immediately drops the budget and a once-swung minion is correctly exhausted). **Summoning-sickness + haste is one *pulled*, target-aware check, not a stored verdict:** if the attacker is `summoningSick` (it entered a board this turn and has not woken at a controller turn-start — set on entry by **both** summon and control), the engine pulls its effective keywords —
  - **Charge** → may attack any legal target;
  - **Rush** → may attack a **minion only** (enemy or neutral; per-lane Taunt still applies), and a hero/face target is rejected `RushCannotTargetHero`;
  - **neither** → it cannot attack at all → `AttackerCannotAttack`.

  A non-sick attacker may hit any legal target. Pulling Charge/Rush here (rather than baking a `canAttack` bool at entry) is the §3 `IKeyword` philosophy: aura-granted haste works mid-turn and silenced haste is correctly lost, with no recompute — and the shared board-entry routine (summon **and** control) does nothing but set `summoningSick`, never a per-handler charge/rush rule. (Neutrals are never `summoningSick`; a `CommandAttackAction` bypasses this eligibility entirely, and retaliation is ungated — see the fizzle note next.)

  The aliveness check is what makes a **mortally-wounded attacker fizzle on re-validation**: a "destroy / return the attacker" interception (§3) leaves the attacker ≤0 / destroy-marked but on board (death is deferred to ⑦), and the held attack, re-entering ③ on resume, is rejected here before it can swing — and likewise the second strike of a **Windfury command** fizzles if the first retaliation mortally wounded the neutral. **This fizzle is the *attacker's* concern only — never the defender's retaliation.** Eligibility (③) gates the *active swing* of an `AttackAction` / `CommandAttackAction`; a defender's **retaliation** is a separate `DealDamageAction` whose source is the defender, and it is **not** subject to this check — so a **mortally-wounded defender still retaliates with full effect** (live keywords + stats, because death is deferred to ⑦). A minion killed-but-not-removed swings back; it just cannot *initiate* a fresh swing.
- Board space: player board has fewer than 7 minions before summoning → `BoardFull`

**Validation is a pure, standalone-invokable predicate.** Stages ② and ③ together form a side-effect-free function `(action, state) → Ok | Rejection` (where `Rejection` is the `ActionRejection` of §2C) — no mutation, no dispatch (④), no events. It is the **single source of truth for legality**: `Submit` runs it before ④, and any "is this action legal?" / "what actions are legal?" query — a future `GetLegalActions(playerId)`, bot move generation, or client pre-flight (greying out unaffordable cards / illegal targets) — **must call this same predicate** rather than re-deriving legality. This guarantees previewed/enumerated legality can never drift from what `Submit` actually accepts. (The positive-enumeration API itself — `GetLegalActions`, via brute-force-then-filter over candidate actions — is deferred to the future bot-support epic; only the standalone-predicate seam is part of the core.)

### ③′ Declaration & Interception Window

A pre-execution checkpoint inserted between validation (③) and dispatch (④) — it does **not** renumber ④–⑨. (Decided 2026-06-04, Fireplace point D; the model lives in §3 "Reactive Triggers, Interventions & Interception".) For every action that reaches dispatch:

1. Publish **`ActionDeclaredEvent { action }`** — the universal pre-execution signal (one event type for all actions; subscribers filter on the carried action's type/params). Most actions have no matching reactive trigger and pass straight to ④.
2. If reactive triggers fired and their host **auto-resolves** (a Secret — deferred flavour), the response resolves inline here (no halt). If matching **hand cards** offer a player response, the engine **suspends** the action — stores it as `pendingIntervention.heldAction`, records the responder's matching + affordable cards in `candidateCardIds`, sets `phase = PendingIntervention`, and opens the **set-valued** window (§4 "PendingIntervention Interruption").
3. **Depth-1 cap (nesting only):** while an interception response is resolving, **no further *player window*** opens to halt it — but an **auto/secret host may still fire, and even intercept, inline** (e.g. a pre-damage hero secret firing on a `DealDamageAction` that a response enqueues), since the cap bounds only *player windows*, not reactions. (Auto-resolving reactions chain in FIFO order.) This bars a nested LIFO stack *of player windows*. It does **not** bar **iteration**: the responder may chain several of *their own* held cards against the one action — see the re-declare loop below.

**Re-declare loop (2026-06-09 — reverses the former "declared once" rule).** When a held action resumes after a **played** response, it re-enters ③ for re-validation; if still legal it is **re-published to ③′**, re-evaluating the responder's *remaining* matching + affordable cards and re-opening the window if any remain (else it proceeds to ④). The loop ends when the responder **skips** (the held action proceeds to ④), when no affordable match remains, or when re-validation **fizzles** the action (cancelled / retargeted off its subject / attacker dead — §3, §4 ③). A **skip never re-opens** the window — only a play can change the offer. The loop is bounded by the responder's next-turn mana (spent from `mana`, pre-loaded at the previous turn-end — see Turn Lifecycle), not a hard cap. See §3 for what a response may do (ordinary effects + re-resolution; only `cancel`/`retarget` touch the held action directly).

### ④ Dispatch → `IActionHandler`

The registered handler for this action type is invoked. It mutates `GameState` and returns `events[]`. This is the sole point of state mutation in the pipeline. `GameEngine` increments `GameState.currentActionEpoch` as the action enters this stage; every event the handler returns is stamped with that `originEpoch`, and any subscription the handler registers (e.g. via `ICardHandler.OnSummon`) is stamped with the same value as its `birthEpoch` — see §3 `IEventBus`. The engine also derives this action's `IRandom` from `mix(GameState.rngSeed, currentActionEpoch)`; every random outcome the handler produces is drawn from that instance (see §3 `IRandom`).

### ⑤ Publish Events → `IEventBus`

Events are published in the order returned by the handler. For each event, `IEventBus` snapshots the subscriber lists at publish time and fires them filtered by creation epoch (`birthEpoch < originEpoch`, see §3 `IEventBus`): current player's list first (sorted by current board index at publish time), then opponent's list (same). The epoch filter means a listener never receives an event from the action that created it. Subscribers enqueue new actions via `IActionQueue` — they do not process them inline.

### ⑥ Aura Recalculation

All registered `IAura.Calculate(state)` implementations are run. `auraAttackBonus` and `auraHealthBonus` on every minion are zeroed and fully rewritten. A minion newly at ≤0 `maxHealth` **enters pending-death** (publishing `MinionMortallyWoundedEvent`, cause = aura loss) — but it is **not removed here**; like a damage-killed minion it lingers, mortally wounded, until the next settle (⑦).

Recalculation runs **per action** (⑥, while draining) — after any action that changes board composition, minion stats, or keywords — and also after each action processed in Phase 2 of the death wave and after Phase 3 reborn summons. Keeping aura recalc per-action (cheaper than deferring it) means mid-cascade reactions always read fresh stats; only *death* is deferred to the settle, not aura math.

### ⑥′ Post-reaction Window

A checkpoint inserted after aura recalc (⑥), before the loop driver (⑨) considers the next action — the post-phase twin of ③′, and like ③′ it does **not** renumber the stages. After an action's events are published (⑤) and stats refreshed (⑥), the engine checks for **hand-card post-reactions** whose triggers matched **this action's** events:

1. The engine gathers the responder's reactive hand cards whose triggers matched **this action's** events and that are **affordable** (`effectiveCost ≤ mana`), records them in `candidateCardIds`, and opens **one set-valued** `PendingIntervention` window (`phase → PendingIntervention`, `heldAction = null`). Each card's offered targets are its matched subjects from this action ∩ its declared `(selector, cardinality)` family.
2. **Deaths are not pre-settled here** (contrast the type-A pre-halt rule under `PendingChoice` / declaration-hold): a post-reaction is *about* what just happened, so a mortally-wounded subject must stay on the board to be acted on. ⑦ still runs only at the eventual full-drain settle.
3. The player plays **one** card — choosing it and its `cardinality` targets — or **skips**. The response **resolves immediately** (see "Response resolution"). A **play** then **re-offers** the *remaining* matching + affordable cards against the same (frozen) events with the now-current board (a card whose condition lapsed — its subject left — drops out); a **skip** ends the window. Only a play re-opens it (the **re-offer loop**, 2026-06-09).

Auto reactions (secrets, board triggers) do **not** open ⑥′ windows — they fired inline during ⑤ in FIFO order and enqueued their actions. Only player-choice hand cards open a ⑥′ window.

### ⑦ Death Resolution

**Settle stage — runs only when the action queue has drained to empty** (the cascade started by a top-level action has fully settled), never mid-cascade. While the queue is non-empty the engine stays in ①–⑥, so every triggered reaction resolves first and a mortally-wounded minion remains on the board (targetable, countable, keyword-active — and savable by a dying-window **save-from-lethal** intervention — a player responding to lethal damage to rescue it before removal, the "R1" avenue in amendment B) right up to this point. Then `DeathResolutionService` runs the wave loop:

1. **Collect** all minions with `currentHealth ≤ 0` or marked for destruction. Sort: current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index. If none, exit loop. Otherwise publish `DeathWaveStartedEvent { waveIndex }` (0-based, incremented per wave).
2. **Phase 1 — Remove & grieve:** for each in sort order — remove from board, build a `GraveyardMinion` from its death state (`snapshot`; **no card is fabricated here** — the card form is produced lazily at point of use, §1), **route it to a graveyard**, then publish `MinionDiedEvent`. Routing reads exactly two inputs:
   - `ownerId != null` → **that player's** `graveyard` (a controlled minion — including a once-`bornNeutral` body taken over via `TakeControlAction` — grieves to its controller).
   - `bornNeutral && ownerId == null` → the shared `GameState.neutralGraveyard` (a system-spawned neutral that died in the neutral lane).
   - `!bornNeutral && ownerId == null` → **undefined, asserts** — no in-scope path creates a player-originated body in the neutral lane (`TakeControlAction` only ever moves bodies *out*). See Unaddressed Features, "A non-`bornNeutral` body dying in the neutral lane."
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

The loop driver. **If the action queue is non-empty**, pull the next action and return to ① (continue draining the cascade through ①–⑥, including ⑥′). **If the queue is empty**, run the settle stages ⑦ (death wave) then ⑧ (win check); the death wave may enqueue actions (deathrattles, reborns), so if the queue is non-empty again, resume draining — otherwise the cascade is fully settled and the engine awaits the next player action. (A player-submitted action is just the next thing dequeued, starting a fresh cascade.)

Post-reaction windows are **not** opened here at the settle; they open **per action at ⑥′**, immediately after each triggering action. A save therefore resolves right after its cause — well before the eventual ⑦/⑧ — so it can lift a mortally-wounded minion back above 0 before any death wave collects it, without waiting for or racing against the full-cascade settle.

---

### Turn Lifecycle

The `EndTurnAction` handler and subsequent system actions implement the full turn transition:

1. Publish `TurnEndedEvent`
2. Fire End-of-Turn triggers (current player L→R, then opponent L→R)
3. **Unfreeze sweep — for the player whose turn is ENDING** (before the swap, so the freeze actually costs an attack rather than thawing at the start of the controller's turn): thaw each frozen character they control **unless** it was frozen *this* turn while exhausted — i.e. keep frozen iff `frozenOnTurn == turn.number && attacksUsedThisTurn >= budget` (the same exhaustion test as `AttackerExhausted`, §4 ③); otherwise set `isFrozen = false`, clear `frozenOnTurn`, emit `MinionThawedEvent`. This makes Freeze cost exactly one attack (a character frozen after swinging stays frozen through its next turn; one frozen on the opponent's turn, or this turn before swinging, thaws at the end of the turn it was unable to act).
4. **Mana refresh — for the player whose turn is ENDING** (moved from turn-start to turn-**end**, 2026-06-09): increment *their* `maxMana` (cap 10) and restore *their* `mana` to `maxMana`. Refreshing here pre-loads the full pool the player then carries **through the opponent's turn**, so an off-turn **intervention** spends straight from `mana` (the ordinary `effectiveCost ≤ mana` check) and the player's own next turn simply begins with whatever interventions didn't consume — **no `reservedMana` field, no separate ceiling**; the affordability ceiling is naturally their *full ramped* next-turn pool. (The game's first turn has no preceding turn-end, so game setup seeds each player's turn-1 `mana`/`maxMana` directly. A future overload-style "less mana next turn" effect would apply its reduction here.)
5. Swap active player
6. Clear `summoningSick` (wake) and reset `attacksUsedThisTurn` on all of the new active player's minions
7. Reset `heroPowerUsedThisTurn`, `cardsPlayedThisTurn`
8. Enqueue `DrawCardAction`
9. Publish `TurnStartedEvent`
10. Fire Start-of-Turn triggers (new active player L→R, then opponent L→R)
11. Start turn timer

Each of these produces actions that go through the full pipeline.

---

### Response resolution (both window kinds)

Whenever a window's response is submitted, the response's effects **resolve to completion immediately** — the engine processes the response's enqueued actions (and any inline board/auto reactions they chain, FIFO) **fully, before returning to drain the pre-existing queue.** Then the window **loops**: a declaration-hold resumes its `heldAction` (re-validated ②③) and, if still legal, **re-declares (③′)**; a post-reaction **re-offers** its remaining matched set at ⑥′. The loop re-opens **iff the response played a card** — which shrinks the offer and may change the board; a **skip** changes nothing and so **terminates** the loop (a declaration-hold's `heldAction` proceeds to ④; a post-reaction's cascade resumes). Each played card pays `effectiveCost` from the responder's `mana` (their next-turn pool, pre-loaded at turn-end — Turn Lifecycle); when nothing affordable remains, the loop ends.

Immediate resolution is **load-bearing, not an optimization**: were the response merely FIFO-appended behind the rest of the cascade, an already-queued `PendingChoice` or declaration-hold could pre-settle deaths (⑦) and remove the very minion a save just targeted, before the heal lands. The **depth-1 cap** applies **within** a response — while it resolves, its own sub-actions open no further *player* windows (③′ and ⑥′ are suppressed); **board/auto triggers still fire inline**. The cap bars only **nesting** (a further *player window* on a response), never **iteration** (the loop re-opening the *same* responder's window after the response completes) and never inline reactions — an auto/secret host may even *intercept* a response's own sub-action inline (e.g. a pre-damage secret on damage the response deals).

### PendingChoice Interruption

When an effect issues `StartChoiceAction`:
- **Pre-halt death rule (amendment B) — type-A halts only:** before halting, Death Resolution (⑦) runs to settle any pending deaths, so the player never acts against a board where mortally-wounded minions still linger. This applies to **`PendingChoice` and declaration-hold interventions** alike — both are halts about a player's own *forward* action, where lingering pending-death minions are noise. It does **not** apply to post-reaction windows (⑥′), which are *about* the just-happened events and must keep their mortally-wounded subjects on board.
- `GamePhase` → `PendingChoice`
- Remaining effect actions (the continuation) are serialised into `PendingChoice.context`
- Pipeline halts — no further actions are processed until the choice resolves
- `SubmitChoiceAction` arrives → selected option is filled into the continuation → continuation actions re-enqueued → `GamePhase` → `InProgress`

### PendingIntervention Interruption

A window **opened by live reactive triggers on hand cards** that matched (§3 "Reactive Triggers, Interventions & Interception") — never a general engine rule. It is **set-valued** (offers *all* the responder's matching + affordable cards in `candidateCardIds`) and **loops** by re-opening after each play; it resolves one of two ways, by which phase the triggers hooked. (Throughout: `phase → PendingIntervention`; `candidateCardIds` lists the offer; timeout timer starts; only `SubmitInterventionAction` from the responding player is accepted — **exempt** from the ③ turn-ownership check, its cost paid from the responder's `mana` (their next-turn pool, pre-loaded at turn-end — Turn Lifecycle); a **play** re-opens the window over the remaining offer, a **skip / timeout** ends the loop; timeout = an injected skip, logged as an input per Item 13; `phase → InProgress` at the end.)

**Declaration-hold (pre-phase — `heldAction` present).** Opened at ③′ when hand triggers fire on `ActionDeclaredEvent`. The triggering action is suspended into `pendingIntervention.heldAction`.
- On **play**: the chosen card resolves first (full pipeline, paying `effectiveCost` from `mana`); it runs ordinary effects (summon, grant keyword, gain armor, …) and may `cancel` or `retarget` the held action. The held action then **resumes, re-validated (②③)**: if `cancel`led or now illegal (attacker dead/returned, target gone, forced to a Taunt) it **fizzles** — dropped, not executed (no `ActionRejection`; no client awaits it) — and the loop ends; if still legal it **re-declares (③′)**, re-opening the **set-valued** window over the responder's *remaining* matching + affordable cards (the **re-declare loop**), or, if none remain, proceeding to ④. The responder may thus chain several of their *own* held cards (bounded by next-turn mana); a response is never intervened on *itself* (depth-1).
- On **skip / timeout**: the loop ends — the held action resumes, re-validates, and executes (④) or fizzles. The window does **not** re-open (a skip changes nothing).

**Post-reaction (post-phase — `heldAction` null).** Hand triggers that fire on regular events (including the cadence-deferred `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`) do **not** halt per-event. The matching events of **one action** are gathered and a **set-valued** window opens at **⑥′** — immediately after that action's ⑤/⑥ (see "Batching"), *not* deferred to the full-cascade settle. Because ⑦ runs only at the eventual settle, the mortally-wounded subject is still on the board when the window opens, so a save (invert a mortally-wounded minion to `currentHealth > 0`, heal a hero above 0) lands before any death wave collects it. **Post-reaction windows never pre-settle deaths** — the subject must remain on board for the response to act on, and the wave simply does not collect a minion the response lifted back above 0. The response resolves immediately (see "Response resolution"); a **play re-offers** the remaining matching + affordable set (the **re-offer loop**); a **skip / timeout** ends the window and the cascade resumes. There is no held action to resume.

**Batching — two granularities: the offered card set, and each card's target set.** (1) *Across cards* — the window is **set-valued**: it offers every reactive hand card of the responder that matched **this action** and is affordable; the player plays one or skips, and a play re-offers the rest (above). (2) *Within a card* — a single AoE produces many sub-events (one selector-carrying `DealDamageAction` → many `DamageTakenEvent`s, one per target), so a card's offered **targets** are its matched subjects from this action **∩ its declared `(selector, cardinality)` family** (the family's `All…/Alive…/Dead…` choice decides whether a mortally-wounded match is offered — inclusive for a "save" card). The triggering unit is therefore **one action's events** — never one event, never the whole cascade. `SubmitInterventionAction` carries `{chosenCardId, targetIds[]}`, validated by the ordinary §4 ③ membership check (`chosenCardId ∈ candidateCardIds`, `targetIds ⊆ matched`).

- **AoE, one save card → one card in the offer, all N subjects as its targets.** An AoE damaging **7** friendly minions while the responder holds a single "heal one minion +2 on `DamageTakenEvent`" card → the window offers that one card over all 7 (the dying ones included); pick one to heal/save.
- **Two separate damage instances (two actions) → two windows.** A card dealing 1 twice via two `DealDamageAction`s opens a window after each; a single-use reactive may be spent on **either** — **skip** the first (which ends *that* window's loop but does **not** consume or discard the card — it stays in hand, trigger registered) to hold for the second, or spend immediately. Distinct instances, not the AoE case.
- **Multiple reactive cards on one action → one set-valued window**, re-offered on each play; the responder chooses order by which card they pick each pass. (This replaces the former "one window per matched card in hand order"; replay determinism comes from the logged `SubmitInterventionAction` choices, not an engine-imposed card order.)

Declaration-hold windows are likewise **set-valued + looped**: each declared action is a single `ActionDeclaredEvent`, so a "counter/redirect the spell" card hooks the one `PlayCardAction`, and a **pre-damage** card hooks the AoE's **single selector-carrying `DealDamageAction`** (§3 `ITargetSelector` auto-hit) — one window for the whole AoE either way, fired at ③′ *before* the hits rather than at ⑥′ *after* them, and re-declared after each play.

Only **player-choice windows** open this way; **auto reactions** (secrets, board triggers) fire inline per-event in FIFO during ⑤ (and during a response — the depth-1 cap suppresses only player windows). A deterministic reaction with no decision (e.g. "heal *it* for 2", no target choice) belongs in the **auto/secret flavour**, enqueuing one effect per match and opening no window at all.

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

### A non-`bornNeutral` body dying in the neutral lane

The §4 ⑦ Phase-1 graveyard-routing rule reads exactly two inputs — `bornNeutral` (origin) and `ownerId` (lane) — and defines **two** destinations: `bornNeutral && ownerId == null` → the shared neutral graveyard; `ownerId != null` → that player's graveyard. The third combination, **`!bornNeutral && ownerId == null`** (a *player-originated* body dying *in* the neutral lane), is **deliberately left undefined** — it routes to neither, and the engine treats reaching it as an assertion gap rather than silently picking a graveyard.

- **Why it can't happen yet:** nothing in scope moves a player-originated minion into the neutral lane. `bornNeutral` is set **only** by the system `SpawnNeutralMinionAction`; control (`TakeControlAction`) only ever moves bodies *out* of the lane (into a player's board, giving them an `ownerId`), never in.
- **What would reach it (signalled, not committed):** a future **"release a player minion to neutral"** effect, or a **"player card that summons into the neutral lane."** Both produce a body that is in the neutral lane (`ownerId == null`) yet not `bornNeutral`.
- **Cost to enable:** an **owner-of-record** on the minion (the controller it should grieve back to when it dies neutral) — a new nullable field plus a routing branch. Intentionally **not** added now: there is no field to populate and no card to populate it, so adding it would be speculative.
- **Status:** open by design. Whoever introduces a player→neutral path owns defining this branch.

### Freezing a hero

`FreezeTargetAction` can target a **hero**, but only minion freeze is modelled — `isFrozen`/`frozenOnTurn` live on `MinionOnBoard`, not `PlayerState`.

- **Why deferred:** a hero attacks via its weapon, and the **hero-combat path itself is only lightly specced** (weapon swing, `heroAttack`, durability) — bolting hero-freeze on now would leave half-wired state. The thaw **rule** is identical to a minion's; what's missing is the storage and wiring.
- **Cost to enable:** add `heroIsFrozen`/`heroFrozenOnTurn` to `PlayerState`, have §4 ③ read the character-appropriate `isFrozen` when the attacker is a hero, fold heroes into the step-3 unfreeze sweep, and add hero freeze/thaw events (`HeroFrozenEvent`/`HeroThawedEvent`).
- **Status:** do it as part of the hero-combat pass; the freeze-thaw rule is already complete and applies unchanged.

### Hero armor

Armor appears as an interception effect (§3 — "gain armor") and as a hero-damage absorber (e.g. a secret "before the hero takes damage, add 8 armor"), but it is **not in the §1 data model**: `PlayerState` has `health` but **no `armor`**, and there is no `GainArmorAction`.

- **Why deferred (2026-06-09):** armor is part of hero defense, which rides the lightly-specced hero-combat path; adding it in isolation would leave its absorption slot in the §4 ④ damage-modifier precedence unpinned.
- **Cost to enable:** add `PlayerState.armor: int` + a `GainArmorAction` (and `ArmorGainedEvent`); pin armor's slot in the §4 ④ precedence (absorbs *after* flat reductions/caps, *before* health loss; Divine Shield is the minion analogue and stays separate); pre-damage interceptions that "add armor" then absorb at the damage moment exactly as the §3 example describes.
- **Status:** do it as part of the hero-combat pass. The intervention/secret mechanics that *consume* armor — a pre-damage hero secret absorbing a reflected hit (worked through 2026-06-09) — already resolve correctly once the field exists; only the storage and the precedence slot are missing.
