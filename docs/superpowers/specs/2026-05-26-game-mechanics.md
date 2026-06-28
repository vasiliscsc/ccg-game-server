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
neutralZone: MinionOnBoard[]          // ownerId = null. The neutral lane is a THIRD side (its own ownerId bucket), modeled as close to a player lane as possible so board-wide mechanics — triggers, auras, summon events — work out of the box. Taunt is per-LANE (a neutral Taunt forces attacks aimed INTO the neutral lane, but not into the opponent lane — and vice versa; §4 ③). Friendly/enemy is LANE-based (§3 `ITriggerCondition`): a player's friendly-scoped aura/trigger excludes the neutral lane, but a neutral minion's OWN friendly scope IS the neutral lane (so neutral minions buff/trigger off each other). Reaching the lane from a player effect needs an explicit `AllNeutralMinions`/`AllMinions` selector.
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone in this game mode
neutralGraveyard: GraveyardEntry[]    // single shared graveyard, owned by GameState (NOT per-player). A dead minion lands here iff bornNeutral && ownerId == null at death (system-spawned neutral that died in the neutral lane); see §4 ⑦ Phase 1 routing
turn: TurnState
timer: TimerState
phase: GamePhase                      // Mulligan | WaitingForPlayers | InProgress | PendingChoice | InterventionDecision | InterventionExtension | Ended
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
spawnPool: Card[]        // pool neutrals are drawn from (uniform, with replacement). DESIGN REMINDER (DR3, 2026-06-28): curate
                         // TIGHTLY (narrow stat band, no format-warpers) — the primary lever bounding neutral RNG swing + the
                         // incoming-player first-crack windfall. MAY also hold beefier 'boss' / CONTESTED objectives (high health,
                         // both players chip it over several turns); reward attribution = the ACTIVE PLAYER at the moment the boss
                         // dies (read activePlayerId at the ⑦ settle that reaps it) — coarser than last-hit-source (an off-turn
                         // killing blow still credits the active player) but simpler, and needs no event-source binding. All v2
                         // CONTENT on existing rails (a high-baseHealth neutral + a death/damage-reaction trigger; OnFire already
                         // has the event, §3 ITrigger) — NOT an engine feature.
firstSpawnTurn: int      // = GameConstants.NeutralFirstSpawnTurn (3): the first turn whose Interturn handover (Turn Lifecycle step 5) repopulates the lane. Before it the lane stays EMPTY; from it on, EVERY handover tops the lane up to maxCapacity (initial seed and recurring refill are one rule). Replaces the old repopulateOnTurnStart bool — neutral spawn timing is now fixed to the neutral handover (turn-END), NOT turn-start, so the upcoming player always gets first crack at what was added (§4 Turn Lifecycle step 5).
```

### GameConstants

Engine-wide constants — the **single source** for setup values and caps, so a future game mode (or a deck-conditional rule, e.g. "deck contains card X ⇒ that player's mana cap +1") overrides them in one place rather than at scattered call sites. Values are the v1 defaults (HS-faithful where an analogue exists). `MaxArtifacts`/`MaxInscriptions`/`MaxDeathWaves` were already referenced inline elsewhere — this block is now their definition too (closing the scattered-magic-number gap, review finding #28).

```
MaxBoardMinions          = 7     // per-player board cap (§4 ③ BoardFull; was an inline "7", #28)
MaxArtifacts             = 3     // artifact row cap (§1 PlayerState)
MaxInscriptions          = 3     // inscription row cap (§1 PlayerState; §3 Sigils)
MaxDeathWaves            = 16    // death-cascade iteration cap (§4 ⑦)
StartingHeroHealth       = 30    // each hero's starting AND maximum health (§1 PlayerState.health; the HealAction cap, #12)
StartingHandFirstPlayer  = 3     // opening hand dealt to the player going first
StartingHandSecondPlayer = 4     // opening hand dealt to the player going second (who also gets The Coin — Match Setup)
HandCap                  = 10    // max hand size; an 11th card is burned (HandOverflowEvent; also the full-hand bounce/give bound, #21)
MaxMana                  = 10    // mana-crystal ceiling (maxMana ramp cap; Turn Lifecycle step 4)
StartingMana             = 1     // turn-1 mana/maxMana seeded for BOTH players at setup (each player's opening turn has no preceding turn-end refresh — F1)
DeckSize                 = 30    // exact decklist size the engine asserts at init
MaxCopiesPerCard         = 2     // per-definitionKey copy limit (1 for a Legendary) — primarily server-side deck validation, asserted here as a guard
NeutralFirstSpawnTurn    = 3     // default NeutralZoneConfig.firstSpawnTurn — end of P1's second turn
InterventionDecisionWindow  = 1  // seconds; the UNCONDITIONAL Stage-1 decision window offered to the non-active player at every ③′/⑥′ (skip / skip-all / intervene). Rank-configurable, set at match init (§3 Reactive…)
InterventionExtensionWindow = 3  // seconds; the Stage-2 pick-a-card window opened by pressing "intervene". Rank-configurable
MaxInterventionsPerTurn     = 3  // intervene-presses (Stage-1→Stage-2 escalations) the non-active player may make per active-player turn — counted whether or not a card is then played; on top of mana; inscriptions (auto, in-line) exempt
```

### PlayerState

```
playerId, heroClass: string
health, mana, maxMana: int    // health: init = max = GameConstants.StartingHeroHealth (30); a HealAction caps a hero at this max (#12 — the minion "healed up to maxHealth" rule applied to heroes), armor (below) being the only over-cap. mana/maxMana: 0 at setup, turn-1 seeded to StartingMana (1) for BOTH players (Match Setup — each player's opening turn has no preceding turn-end refresh, F1), thereafter refreshed at turn-END (Turn Lifecycle step 4); mana MAY exceed maxMana via a gain such as The Coin (§2A ModifyManaAction, #22). (Hero entity-id addressing convention is review finding #12, walked separately.)
armor: int                    // hero damage ABSORBER (init 0, no cap): depleted before health at the §4 ④ damage-modifier precedence (hero slot — the minion analogue is Divine Shield). Gained only via GainArmorAction. NOTE heroes have NO attack stat, NO weapon, NO frozen state — heroes never attack (hero-weapon concept dropped 2026-06-11; the hero kit is the artifact row below)
hand: Card[]
deck: Card[]
board: MinionOnBoard[]        // player's own side only; ordered left to right
graveyard: GraveyardEntry[]   // unified — minions + spells + artifacts
artifacts: ArtifactOnBoard[]  // the player's artifact row (own board section, NOT minions) — cap GameConstants.MaxArtifacts = 3
                              // incl. the STARTER artifact equipped at match setup (one shared definitionKey for every hero —
                              // the hero-power replacement, 2026-06-11). See ArtifactOnBoard.
inscriptions: Card[]          // the player's INSCRIBED sigils (§3 "Sigils", 2026-06-11): reactive cards pre-armed face-down.
                              // Cap GameConstants.MaxInscriptions = 3; ONE per definitionKey. The Card itself moves zones
                              // (hand → inscriptions → graveyard on reveal). Identity hidden from the opponent; existence/count
                              // public (§2B directed/public sigil events).
cardsPlayedThisTurn: int      // Combo tracking (#23): counts THIS player's on-turn card plays — a `PlayCardAction` (incl. the inscribe branch) increments it. An off-turn **invocation** (`SubmitInterventionAction`) does NOT increment (Combo is an on-turn tempo mechanic; nothing off-turn primes it). Reset to 0 for the incoming active player at turn advance (Turn Lifecycle step 8). **Off-by-one note (G1, 2026-06-21):** the `++` lands at `PlayCardAction` ④ (commit), which **precedes** a Combo battlecry's eval at `ResolveCardAction` ④ (resolution) — so by the time Combo checks, *this card is already counted*. The Combo predicate is therefore `cardsPlayedThisTurn ≥ 2` (this card + ≥1 prior), **never** `> 0` (which would make Combo always-on); see §3 `ITriggerCondition`
fatigueCounter: int           // empty-deck-draw counter; init 0, persists all game, NEVER resets. Each draw from an empty deck does fatigueCounter++ THEN deals the new value to this hero (1,2,3,…) — see §2A DrawCardAction
autoSkipAll: bool             // client-set STANDING preference (default false): while true the engine elides this player's intervention DECISION windows (auto skip-all, no wait, no round-trip). Read at each window-open; flipping it true also resolves a currently-open decision window. Replay records flag TRANSITIONS, not per-window skips (§3 Reactive…, §4 ③′/⑥′). NOT the same as the `RespondInterventionAction` `SkipAll` response (F15): `autoSkipAll` is a turn-spanning standing flag covering ALL of this player's windows; `SkipAll` is a single response covering only the CURRENT action's remaining windows (its ③′ + ⑥′).
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
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus. **currentHealth follows maxHealth changes (#13, HS-exact):** when maxHealth RISES by Δ (a health buff/enchant or aura grant), currentHealth += Δ (the missing-health gap is preserved); when it FALLS (buff/aura removal, Silence), currentHealth = min(currentHealth, newMaxHealth) — **clamp current DOWN to the new max if it exceeds it, otherwise leave it unchanged.** Buff/aura-loss lowers the *cap*; it never deals fresh damage and **never kills** (HS-faithful: removing a positive Health buff cannot kill a minion — a minion already at/below the new max is untouched; e.g. a base-2-health minion buffed to 2/4, damaged to current 2, then silenced stays at 2/2, it does NOT die). The clause has **NO minimum** so the *one* death path is `maxHealth ≤ 0` from a **negative** buff/aura (`Σ` drives the cap to zero) — the collapse / aura-loss-death path → mortally wounded (F16; distinct from positive-buff removal, which floors at the printed base ≥ 1). Unified: newCurrent = min(currentHealth + max(0, Δmax), newMax) — the `max(0, Δmax)` is load-bearing: it makes a *fall* a pure clamp (no health subtracted), a *rise* carry current up by Δ (gap preserved). Applied wherever maxHealth changes — BuffMinionAction ④, SilenceMinionAction ④ (strips enchants), the ⑥ aura recompute (Δ vs the previous pass). Damage/heal move currentHealth directly, never maxHealth.
currentHealth: int            // takes damage; healed up to maxHealth. ≤0 (or destroy-marked) ⇒ pending death: the minion stays ON the board — still targetable, countable, and keyword-active — until the next death-wave settle (§4 ⑦), not removed on the spot (amendment B). The two doom kinds differ in SAVABILITY (2026-06-12): ≤0 health = "mortally wounded" (fires MinionMortallyWoundedEvent, savable before the settle — heal for a damage wound, or restore maxHealth for an aura/buff-collapse wound, which heal cannot fix; F16); destroy-marked = FINAL (no event, no save; only preventable at the destroy action's ③′ declaration — §2A DestroyMinionAction)
keywords: string[]            // maintained EFFECTIVE view = intrinsicKeywords ∪ grantedKeywords ∪ auraKeywords; resolved to IKeyword at runtime; what the engine queries. (Keyword analog of effectiveTribes.)
intrinsicKeywords: string[]   // from the card definition, seeded at summon; Silence clears
grantedKeywords: string[]     // PERMANENT non-aura grants after summon; retained across a RetainEnchantments bounce; Silence clears. Aura grants are NOT here.
auraKeywords: string[]        // aura-granted, recomputed each aura pass; never persisted, never bounce-retained; Silence does not touch (they recompute). ANY keyword may be aura-granted — keywords are pulled from the effective view, never a bus subscription (§3 IKeyword; keyword-collapse 2026-06-02).
intrinsicTribes: Tribe        // copied from the card's tribes at summon; immutable; survives Silence
grantedTribes: Tribe          // permanent tribe grants ("becomes a Beast"); Silence clears to Tribe.None
auraTribes: Tribe             // recomputed each aura pass; never persisted (parallels auraAttackBonus); drops when the aura leaves
effectiveTribes: Tribe        // computed = intrinsicTribes | grantedTribes | auraTribes; all engine tribe checks use this
summoningSick: bool           // raw fact (replaces the old derived `canAttack` bool): true while this minion entered a board THIS turn and has not yet woken at its controller's turn-start. Set on board-entry by BOTH summon and control; cleared by the turn-start sweep (Turn Lifecycle step 7). Attack eligibility — including the Charge-any-target / Rush-minions-only distinction — is PULLED from effective keywords at §4 ③, never baked into a stored verdict here. Neutrals are never summoningSick (no turn of their own; command bypasses eligibility and retaliation is ungated).
attacksUsedThisTurn: int      // raw counter, incremented on each attack; reset at the controller's turn-start sweep. The ONLY attack-state stored on the minion — the per-turn attack BUDGET is deliberately NOT a field: it is PULLED from effective keywords at §4 ③ (Windfury → 2, else 1; extensible to future budget-granting keywords), so it can never desync from the keyword (silencing Windfury drops the budget to 1 on the spot). NOT consulted for a neutral commanded via CommandAttackAction — command is a card-granted activation, not the minion's own budget (§2A / §4 ③).
summonOrder: int              // monotonically increasing per session; used for trigger fire ordering
isFrozen: bool                // frozen → cannot attack (§4 ③ AttackerFrozen)
frozenOnTurn: int             // turn.number when the freeze landed (meaningful only while isFrozen); drives the end-of-turn thaw rule (Turn Lifecycle step 3 / §3 IKeyword Freeze)
rebornAvailable: bool         // the one-time Reborn charge — distinct from the "reborn" keyword tag, which persists. **Initialized `true` for EVERY minion** at summon (F5 — the charge is keyword-INDEPENDENT, so a Reborn *granted after summon* still fires); reborn fires only when `keywords has reborn && rebornAvailable` (§3 IKeyword / §4 ⑦), so a non-Reborn minion never reborns despite a live charge. The reborn-summon path summons the copy with it false (the no-loop guard); consuming it (NOT the keyword) is what reborn does. Silence leaves it untouched — stripping the `reborn` keyword already gates off reborn (F5/option-a). See §4 ⑦.
// TRIGGERS — not stored as fields. This minion's ITrigger(s), Deathrattle, and keyword-hooks are registered into IEventBus by its ICardHandler (resolved via definitionKey) at summon (OnSummon, ④) and dropped when it leaves the board. This is UNIFORM across summon paths (F19, simplified 2026-06-21): a TOKEN (SummonMinionAction) and a CARD-PLAYED minion (ResolveCardAction ④) both register OnSummon + publish MinionSummonedEvent inline at the summon's own ④; a card-played minion's Battlecry is simply enqueued AFTER, so its triggers are already live when the Battlecry runs (it reacts to its own Battlecry — a documented HS divergence; see §2A ResolveCardAction ④ / §3 trigger-timing note). Live by virtue of the board zone (§3 "Reactive Triggers, Interventions & Interception"). The bus carries only ITrigger; keywords are pulled from the effective `keywords` view (§3 IKeyword), not subscribed.
```

### ArtifactOnBoard

Artifacts (2026-06-11) **replace the hero-power system** — there is no `heroPower`/`UseHeroPowerAction`. A new first-class entity in its own per-player board section: **passive** (hosts triggers — and may host an `IAura`), **active** (activate → effects, optional mana cost and target), or **both**.

```
artifactId: string            // new entity-ref space (debug-trace prefix `a`)
definitionKey: string         // sole link to the definition + handler — registers triggers/auras on equip (zone-scoped,
                              // §3 "Reactive Triggers"), supplies activation effects + target selector + the cost formula
ownerId: string               // artifacts are always player-owned; there are no neutral artifacts
baseActivationCost: int?      // null = passive-only (not activatable). The EFFECTIVE activation cost is PULLED per
                              // activation from the definition's cost formula over (artifact, state) — default = base;
                              // the starter overrides with base + usesThisTurn (escalating draw; base = 2 — DR2, 2026-06-28:
                              // a CONSISTENCY / anti-variance tool — a bandaid for losing to the shuffler, NOT a value engine. The real
                              // tempo cost makes drawing a "draw or develop?" decision (never a free default) and self-targets the
                              // flooded player (spare mana = exactly when it's affordable). `base` is a tunable BALANCE number; the
                              // flood-vs-screw mechanism (plain draw vs loot/dig) and the symmetric grind-tilt are playtest-watch, not
                              // pre-decided). Validation and client
                              // preview read the same pull (§4 ③) — never a stored verdict.
durability: int?              // null = infinite. ONE counter; the definition declares which moment(s) consume a point —
                              // activation, trigger-fire, or both. Reaching 0 enqueues DestroyArtifactAction.
charges: int                  // generic accumulator, init 0 — instance state for quest-counter designs (mutated only via
                              // ModifyArtifactChargesAction → ArtifactChargesChangedEvent)
usesThisTurn: int             // activations this turn; reset at the controller's turn start (Turn Lifecycle step 8);
                              // feeds escalating-cost formulas. There is NO per-turn activation budget — cost and
                              // durability are the only limiters.
equippedOnTurn: int
// NOT combat entities: no attack/health — never attackers/defenders, never death-wave members, not Silence-able,
// not aura-RECEIVERS (nothing to buff). Removal = durability exhaustion, owner discard, or an explicit destroy effect.
```

### Card

```
id, name: string
definitionKey: string         // the only cross-entity link into the card-definition library (Identity section); distinct from `id`, which is the per-instance allocator
type: CardType                // Minion | Spell | Artifact   (Artifact replaced HeroPower, 2026-06-11; Weapon REMOVED 2026-06-11 — heroes never attack, no weapon cards exist)
rarity: CardRarity
tribes: Tribe                 // [Flags] intrinsic taxonomy from the definition; Tribe.None = tribeless (valid). Gameplay tag; NOT cleared by Silence. See "Tribes" below.
baseManaCost: int
baseAttack?, baseHealth?: int // Minion only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
grantedKeywords: string[]     // keywords carried over from a RetainEnchantments bounce/shuffle. On play, these migrate to the new minion's grantedKeywords (and keywords) — they are not part of the card's base definition.
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // the card's definition body
handlerKey?: string
// REACTIVE TRIGGERS — a card whose definition carries a reactive ITrigger is a SIGIL (§3 "Sigils"). Its ICardHandler registers the trigger into IEventBus live-by-zone, dropped when the card leaves the hand by any route (play / discard / return / shuffle-into-deck — #25); the SAME trigger behaves per zone: in HAND → the sigil may be INVOKED (player-choice intervention window); in the INSCRIPTIONS zone → it auto-resolves (an INSCRIPTION; player choices degrade to random — §3). Every sigil supports both deployments. (Most cards carry no reactive trigger and just sit inert in hand.)
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
  snapshot: MinionOnBoard   // full state at death; carries definitionKey → card fully derivable
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  definitionKey: string     // a spell has no board snapshot, so it stores its own identity here → card fabricated from this on recast

GraveyardArtifact : GraveyardEntry
  artifactState: ArtifactOnBoard // full snapshot (carries definitionKey → card derivable); both DISCARD and DESTROY land here
  destroyedOnTurn: int
```

### Supporting types

```
TurnState           — activePlayerId: string?, number: int, interventionsUsed: int   // activePlayerId null = the neutral lane's "turn" — held ONLY during the Interturn neutral handover (§4 Turn Lifecycle step 5): neither player is active, so neither player's triggers fire — only the neutral group does (§3 IEventBus). Never null while either player may submit an action. interventionsUsed = the non-active (responding) player's intervene-presses THIS turn; reset to 0 at turn advance (Turn Lifecycle step 6), ++ on each Stage-1→Stage-2 escalation; once it reaches GameConstants.MaxInterventionsPerTurn the decision window stops opening (§4 ③′/⑥′).
TimerState          — secondsRemaining: int
PendingChoice       — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[],
                       context: JsonElement  // stores effect continuation. Chained choices = a continuation issuing the next
                                             // StartChoiceAction — the future v2 card-crafting flow rides this unchanged
                                             // (see notes/2026-06-11-card-crafting-v2.md; v1 builds NOTHING for it)
ChoiceType          — enum { Discover, Target, Modal }   // (F9, 2026-06-17) the PRESENTATION discriminator the client renders by. Discover = pick 1 of N generated cards; Target = pick 1 entity from a mid-effect candidate set (an ITargetSelector, §3 "Player-choice" mode); Modal = pick a mode of a "Choose One" card. Mulligan is NOT a member — it rides its own SubmitMulliganAction. The enum grows by one member when a genuinely new presentation ships.
ChoiceOption        — record (string optionId, string? entityId = null, string? definitionKey = null, string? label = null)
                       // (F9) a DISPLAY + ANSWER-ROUTING token, NEVER the effect itself. optionId = the stable handle SubmitChoiceAction.selectedOptionId references AND the per-mode dispatch key the continuation switches on (a Modal mode's effect lives in the card definition's modes[], reached by optionId — it is NOT carried here; reorder/localize never misroutes). entityId = an existing board/hand entity to act on (Target); definitionKey = a card definition to mint (Discover); label = display string. All payload fields optional, so an option is as thin or rich as its ChoiceType needs. INVARIANT: the effect-spec ALWAYS lives in the card definition / PendingChoice.context continuation — a new mechanic needs new continuation logic, never a new ChoiceOption field (this is what keeps the choice wire schema stable). Safe to send to the client: it never serializes executable effects.
PendingIntervention — respondingPlayerId: string, heldAction: GameAction?, candidates: InterventionCandidate[],
                       timeoutSeconds: int
                       // TWO-STAGE window (2026-06-16). The responder is ALWAYS the non-active player (off-turn only).
                       // The record backs both GamePhases: InterventionDecision (the unconditional Stage-1 window —
                       // skip / skip-all / intervene; candidates MAY be empty, so "intervene" is offered only when non-empty)
                       // and InterventionExtension (Stage-2 — pick ONE card + targets, or back out). respondingPlayerId =
                       // the non-active player; heldAction null = post-reaction (⑥′, re-offer loop), non-null = declaration-hold
                       // (③′, re-declare loop). Each PLAY re-opens a fresh decision window (offer shrinks); a skip / skip-all /
                       // timeout ends it. timeoutSeconds = the current stage's window (Decision vs Extension GameConstant).
InterventionCandidate — cardId: string, targetIds: string[]
                       // one offered card + its ENGINE-COMPUTED legal targets (2026-06-10, visibility): at ⑥′ = the
                       // card's matched subjects from this action ∩ its (selector, cardinality) family, recomputed each
                       // re-offer pass against the current board; at ③′ = the card's own targeting (cancel/retarget-type
                       // cards target the held action itself). The SINGLE source for both the §4 ③ submit-validation
                       // (set membership) and the responder-directed InterventionPromptEvent (§2B) — so the client
                       // renders the offer with no game logic.
MulliganState       — player1Completed: bool, player2Completed: bool
EffectContext       — sourceId: string, sourcePlayerId: string, targetId: string?,
                       state: GameState (read-only), spellDamageBonus: int
                       // BASE resolution context — passed to IEffect.Execute, ITargetSelector.Select, and keyword pulls (§3). `sourceId` is the unified entity ref (minion OR card) and `sourcePlayerId` its controller, set by the enqueuer from the action's own source fields (§3 "Source Attribution"). Id-based precisely because the source may be a minion, not a card.
CardPlayContext : EffectContext — + card: Card
                       // SUPERSET for ICardHandler.OnPlay only: adds the resolved Card the handler reads its `definition` from. Here card.id == sourceId and playerId == sourcePlayerId; it inherits targetId/state/spellDamageBonus, so OnPlay can pass itself to the IEffects it dispatches and bake spell-damage at enqueue.
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

Cards and minions are independent entities with independent **but type-prefix-disjoint** ID allocators — so a bare instance id is globally unambiguous and the engine reads entity type from the id's prefix (§2A). They are linked only by a shared `definitionKey` (string — e.g. `"river_crocolisk"`, `"boar"`) into the card-definition library.

**Identity rules:**

- `Card.id` and `MinionOnBoard.minionId` are allocated from two distinct counters; **IDs are never reused** within a session.
- **ID format — type-prefixed, per-session monotonic, never reused:** minions `m0, m1, …`; cards `c0, c1, …`; artifacts `a0, a1, …`. **Neutral minions share the `m` allocator** — *every* minion (player-owned or neutral) has an `m`-id; neutrality is `ownerId == null`, **not** a separate id space, and the id is **stable across `TakeControlAction`** (control flips `ownerId`, never the id). The two **heroes/players** are the fixed seats **`p1` and `p2`** (= `GameState.player1`/`player2`): a hero **is** its player (its stats live on `PlayerState`), addressed by its `playerId`, with **no separate hero-entity id**. `playerId` is a **match-local seat**, never an external/account id — the Game Server maps account ↔ seat. `ownerId` ∈ {`p1`, `p2`, `null`} is the owner *field* (null = the neutral lane), not an entity id. The type-prefixes keep all instance ids globally disjoint, so the uniform `entityId` target field (§2A) resolves by id alone.
- The `definitionKey` field on `Card` and on `MinionOnBoard` points into the static card-definition library. It is the only cross-entity link.
- Playing a Card to summon a Minion does **not** transfer the Card's `id` to the minion. The Card leaves play (to graveyard or fabricated-elsewhere); the Minion is a new entity with a fresh `minionId`.
- Tokens (minions summoned by effects with no originating card) have a `minionId` and a `definitionKey`. There is no `Card.id` associated with them while on the board.
- A `GraveyardMinion` stores **no card** — only its `snapshot` (full death state, including `definitionKey`). Lineage-aware effects derive everything from it: "resummon the last minion that died" uses `snapshot.definitionKey` (a *new* `minionId` is allocated); "add the dead minion's *card* to hand" / resurrect-as-card **fabricates** a Card on demand from `snapshot` via the Fabrication rule above (minting a fresh `Card.id` at that moment). This holds uniformly for tokens and played-from-hand minions alike — `MinionOnBoard` never retains its originating Card, so the card form is always a fabrication, never a stored reference.

### Minion → Card Transitions (Bounce / Shuffle-In)

Any action that moves a minion off the board into a card-shaped zone (hand or deck) takes a `policy` parameter of type `MinionToCardPolicy`:

```
MinionToCardPolicy = Stripped | RetainEnchantments
```

The effect that triggers the transition decides which policy applies (e.g. a Sap-style spell uses `Stripped`; a Recall-style spell uses `RetainEnchantments`). The action itself encodes the policy; the handler implements both behaviours.

**A transition is not a death (#21).** Moving a minion off the board into a card zone is the minion *surviving* as a card — no Deathrattle, no death wave. The one exception falls out of the destination being full: a bounce into a hand already at `GameConstants.HandCap` (or a shuffle into a deck with no room, were that ever in scope) cannot place the card, so the minion is **destroyed** instead — routed through `DestroyMinionAction`, which *is* a death, so the Deathrattle fires (§4 ⑦). The asymmetry ("bounce with room = no deathrattle; bounce into a full hand = deathrattle") is not a special rule — deathrattle is gated purely on whether the minion died, and only the full-destination case is a death. (A `GiveCardAction` into a full hand burns the card per the draw-overflow rule — no minion is involved.)

**Fabrication rule:** every minion→card transition fabricates a *new* Card with a freshly allocated `Card.id`. The minion's `minionId` is retired. Lineage to any earlier originating Card (if one existed) is not preserved on the fabricated Card.

**What's retained, by policy:**

| Field on the new Card | `Stripped` | `RetainEnchantments` |
|---|---|---|
| `definitionKey` | from minion | from minion |
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

**Replay semantics:** when the fabricated Card is played again, the resulting minion inherits `grantedKeywords` from the Card into both `keywords` and `grantedKeywords` on the new minion, and inherits `modifiers` into the new minion's `enchantments`. The new minion always has a fresh `minionId`.

**Scope note — keyword grant tracking:** `grantedKeywords` only captures non-aura buffs (effects that mutate the minion permanently). Aura-applied keywords are never added to `grantedKeywords`; they live only in `keywords` and are rewritten on every aura recalculation. Silence clears both `keywords` and `grantedKeywords`. This design avoids any per-source ledger in v1; if a future card requires per-source attribution (e.g. "remove only the Taunt that Sergeant gave," not the base Taunt), the model must extend `grantedKeywords` to `KeywordGrant { sourceId, keyword }`.

---

## Section 2: Actions + Events

Actions are immutable commands (input layer). Events are immutable facts (output layer). Every state change is produced by an action and announced via one or more events. The client receives events only — never raw state diffs. Both are C# records.

### 2A. GameAction types

All actions carry a nullable `SourcePlayerId` (null = system-issued) and a `RequestedAt` timestamp. The engine infers entity type (minion vs hero vs card vs artifact) from the ID's **type-prefix** at processing time (`m*` minion, `c*` card, `a*` artifact, `p1`/`p2` hero — §1 "Card and Minion Identity") — actions do not carry type discriminators. A **hero** `targetId` is simply its `playerId` (the seat `p1`/`p2`); there is no separate hero id.

**ID convention:** a `cardId` / `minionId` / `targetId` field names an **existing instance** in a zone (a `Card.id`, a `MinionOnBoard.minionId`, …); a `definitionKey` field names a **library key to fabricate or summon from**, where no instance exists yet (summoning a token, generating a card into hand, equipping an artifact, transforming). The two are never interchangeable — see "Card and Minion Identity" (§1). **`EntityId`** (the §3 `ITargetSelector.Select` return and `IEntityPredicate.Matches` parameter type) is a documented **alias for `string`** in v1 — the same instance ids these fields hold (`minionId`/`cardId`/player id); it reads `EntityId` at the selector/predicate boundary only to signal "an entity instance ref" intent. Promoting it to a typed wrapper (as the old Unity project did) is a possible Epic-01 refinement, **not** v1; until then treat `EntityId` ≡ `string` everywhere (F14).

**Player-initiated:**

| Action | Key fields |
|---|---|
| `PlayCardAction` | playerId, cardId, targetId?. ③ the standard gate (turn ownership; `effectiveCost ≤ mana`; target ∈ the card's selector). **Two ④ branches by card + context — (1) Normal play = COMMITMENT (2026-06-13, #34):** pay mana (`ManaChangedEvent`), move the card **out of hand**, `cardsPlayedThisTurn++`, emit `CardPlayedEvent` (**not** `SpellCastEvent` — that fires at *resolution*, `ResolveCardAction` ④, so a countered spell never casts; D2, 2026-06-21), write a **spell**'s `GraveyardSpell` entry **here** (so a countered spell is already in the graveyard during the window — both outcomes, user's pick), and **enqueue `ResolveCardAction { card, playerId, targetId? }`** — carrying the **committed `card` snapshot** (taken here at commit, while the card still holds its `modifiers`/`effectiveCost`; `card.id`/`card.definitionKey` subsume the former separate `cardId`/`definitionKey` fields, G2 2026-06-22), which lets the resolution emit a cost-readable `SpellCastEvent`. The card's **effects do NOT run here** — commitment and resolution are split so an interception can sit *between* them (the idiom `AttackAction` / `ActivateArtifactAction` already use: commit/pay at ④, enqueue the consequence; card plays were the odd one out). A **minion** card writes **no** graveyard entry at commitment (its body, summoned at resolution, grieves as a `GraveyardMinion` if it later dies — avoids a double entry). **(2) Inscribe** = playing a **sigil** outside a response window (§3 "Sigils"): same cost gate (pay-on-inscribe, the trigger fires free later); ③ additionally rejects `InscriptionSlotsFull` (row of 3) / `SigilAlreadyInscribed` (duplicate definitionKey); ④ moves the card hand → `inscriptions`, registers its reactive trigger (zone-entry, `birthEpoch` stamped), `cardsPlayedThisTurn++`, emits the owner-directed `SigilInscribedEvent` + public thin `InscriptionAddedEvent` **instead of** `CardPlayedEvent`/`SpellCastEvent` (identity-hidden) — a **terminal ④, no `ResolveCardAction`**. Playing a sigil from hand *inside* a window goes via `SubmitInterventionAction` (an **invocation**) instead. **Interception suppression (DR1, 2026-06-28):** if the played card's definition declares `Inexorable`/`Uncounterable` (§3 "Non-interceptable actions"), `PlayCardAction` stamps the play-unit `suppressDeclaration` here — both this action and the enqueued `ResolveCardAction` skip their ③′ (so the cast can be neither prevented nor countered); `Inexorable` also sets the `propagate` bit, threading the skip onward to the effects `ResolveCardAction` enqueues. |
| `ResolveCardAction` | card, playerId, targetId? — **the resolution half of a card play**. The `card` is the **committed snapshot** threaded from `PlayCardAction` ④ (G2, 2026-06-22): it supplies `card.id` (the former `cardId`) and `card.definitionKey` (the former `definitionKey`), which is why those two are no longer separate fields — and, crucially, it preserves the spell's `modifiers`/`effectiveCost` for the resolution's `SpellCastEvent` to carry (a `GraveyardSpell` stores `definitionKey` only, so the snapshot is the *only* surviving source of the cast cost). (enqueued by `PlayCardAction` ④, never submitted directly; 2026-06-13, #34). An ordinary queue citizen: **its ③′ is the counterspell window** — the held action is *this resolution*, so a trigger that cancels here is a true **Counterspell** (costs already sunk + card already spent at the play's ④), whereas a trigger that cancelled the earlier `PlayCardAction` ③′ *prevented the cast* (nothing sunk, card still in hand). The trigger's condition picks which; the pending `ResolveCardAction` **is** the MTG "spell on the stack" (no new zone — the queue is the zone). **④:** run the card's effects — a spell's effects (emit **`SpellCastEvent` here, at resolution**, now carrying the full `card` snapshot (G2, 2026-06-22) so cost/stat-gated cast-synergy can read `effectiveCost` — D2, 2026-06-21: it is the spell-side analog of `MinionSummonedEvent`, so cast-synergy "whenever/after you cast a spell" fires **only if the spell resolves**; a spell **countered** at this action's ③′ never reaches ④, so it casts nothing — current-HS Counterspell semantics. The pre-resolution ③′ `ActionDeclaredEvent` is the *interception* hook, not where synergy lives), or a minion card's **summon + Battlecry** (F19, simplified 2026-06-21): **(1) summon the body** through the **ordinary summon path** — place it on the board, register its persistent `ITrigger`s/Deathrattle/keyword-hooks (`OnSummon`), and return `MinionSummonedEvent`, all at this ④ (epoch N) — **identical to what `SummonMinionAction` does for a token**; **(2) enqueue the Battlecry**'s effect actions as ordinary queue citizens (each keeps its own ③′/⑥′, so the Battlecry stays fully **intervenable** — a "Battlecry: deal 3" opens a counterspell-style ③′ on its `DealDamageAction`; nothing is swallowed). **No sub-drain, no deferred epoch.** Because the Battlecry is enqueued at this ④ **before** `MinionSummonedEvent` publishes at ⑤, it sits ahead of any summon-reaction in the queue and resolves first — so **other** minions' "after you summon" reactions (Knife-Juggler-style) fire **after** the Battlecry (HS-faithful: Razorfen Hunter's Boar is summoned by the Battlecry before Knife Juggler throws). The body, placed at step (1), is on the board when the Battlecry runs, so a "deal N to all *other* minions" Battlecry correctly sees and excludes it. The new minion does **not** react to its own `MinionSummonedEvent` (the equal-epoch creation filter, §3 `IEventBus`); it **does** react to its own Battlecry's later-epoch events — a **deliberate, documented divergence** from HS's "Battlecry before triggers go live" (§3 trigger-timing note; "Unaddressed Features → A minion triggers off its own Battlecry"). The token path (`SummonMinionAction`, no Battlecry) is now exactly step (1) — the two summon paths no longer differ in timing. *(A minion killed during its Battlecry was a real summoned minion: it dies normally to the ⑦ settle — there is no "skip registration" rider, since registration already happened at step (1).)* **Fizzle** (cancelled at ③′ / illegal on resume — e.g. its target left): effects never run, emit `CardPlayFizzledEvent { playerId, cardId }`, **no refund** (mana stays spent; a spell's graveyard entry stays). A **minion** card that fizzles is **removed from the game** (F10, 2026-06-17) — **no** graveyard entry: it never reached the board and never *died*, so it must not become a resurrection/recursion target ("resurrect a friendly minion that died" reads `GraveyardMinion`s), and there is no `diedOnTurn` to define for a thing that didn't die. The card instance simply ceases to exist (present in no zone — the cheapest sink, no tracked exile zone needed; HS-faithful — a countered card is not recyclable). A minion card that resolves writes nothing here. **Interception suppression (DR1):** inherits the play-unit `suppressDeclaration` from `PlayCardAction`, so an `Uncounterable`/`Inexorable` card's resolution skips its own ③′ (uncounterable); under `Inexorable` (`propagate` set) it threads the skip to the Battlecry/effect actions it enqueues — under `Uncounterable` it does not, leaving those effects interceptable (§3 "Non-interceptable actions", §4 ③′). |
| `AttackAction` | attackerId, targetId. **Handler (④), after ③ re-validation:** (1) snapshot the attacker's effective **base attack** (HS-style attack-lock; damage *modifiers* are pulled later, per `DealDamageAction` ④); (2) **consume the activation** — `attacksUsedThisTurn++`; (3) **break Stealth** if the attacker has it — an own-moment of *attacking*: clear the Stealth keyword and emit `StealthBrokenEvent` (attacking is what drops Stealth; merely being targeted/dealing non-attack damage does not — closes the Stealth half of hole #4); (4) emit `AttackPerformedEvent { attackerId, targetId }` (the post-interception **actual** target); (5) **enqueue two independent `DealDamageAction`s** — the strike (attacker → target, amount = the snapshotted attack) and, iff the defender's attack > 0, the retaliation (defender → attacker) — processed **sequentially, not atomically** (§3). **Freeze interaction:** a frozen attacker never reaches ④ (rejected at §4 ③ `AttackerFrozen`); conversely the `attacksUsedThisTurn` consumed in step 2 feeds the end-of-turn unfreeze sweep (Turn Lifecycle step 3) — a minion frozen **this turn while exhausted** (it had already swung) **stays** frozen through its next turn, one frozen while it **still had an attack thaws** at the end of this turn, so Freeze costs exactly one attack. |
| `ActivateArtifactAction` | playerId, artifactId, targetId? — activate an **active** artifact (replaces `UseHeroPowerAction`, 2026-06-11). ③: own turn; artifact owned; activatable (`baseActivationCost != null`, else `ArtifactNotActive`); **pulled effective cost** ≤ `mana` (the definition's cost formula — the starter's = `base + usesThisTurn`); target ∈ the definition's selector. ④: pay the pulled cost, `usesThisTurn++`, enqueue the definition's activation effects, consume `durability` if activation is a declared consumer (`ArtifactDurabilityLostEvent`; at 0 → enqueue `DestroyArtifactAction`), emit `ArtifactActivatedEvent`. **Unlimited per turn** (no budget field — cost/durability limit it); does **not** increment `cardsPlayedThisTurn` (not a card play; Combo unaffected); declared at ③′ like any action (interceptable). |
| `DiscardArtifactAction` | playerId, artifactId — the owner's in-turn right to free a slot: own turn, **free**, unlimited per turn. Unregisters the artifact's triggers/auras, snapshots it to the owner's graveyard (`GraveyardArtifact`), emits `ArtifactDiscardedEvent` — and **only** that (never `ArtifactDestroyedEvent`; the voluntary/involuntary split is two disjoint event types, not a `cause` field — the session-7 idiom). |
| `EndTurnAction` | playerId |
| `SubmitMulliganAction` | playerId, cardIdsToKeep[] |
| `SubmitChoiceAction` | playerId, selectedOptionId (= the chosen `ChoiceOption.optionId`). **No `choiceId`** (F9, 2026-06-17): at most one `PendingChoice` is in flight at a time (the pipeline halts on it — §4 "PendingChoice Interruption"), so there is nothing to disambiguate. ③: `selectedOptionId ∈` the pending choice's `options`. **No ③′** (A3, 2026-06-28): choice-resolution machinery, not a declarable board move — like `SubmitInterventionAction` it publishes no `ActionDeclaredEvent`; the continuation actions it re-enqueues each declare on their own. ④: resume the serialized continuation, dispatching on `optionId`. |
| `SurrenderAction` | playerId |

**Responder-initiated (off-turn)** — submitted by the **non-active** player during a `PendingIntervention` window (Phase Guard `InterventionDecision`/`InterventionExtension`; the window is *opened* by the system `StartInterventionAction` below). Player-submitted (**not** system actions — they are phase-gated, see §4 ②), but exempt from the ③ turn-ownership check **and from ③′ declaration** (window-response machinery, not a state-mutating board move — A2, 2026-06-28):

| Action | Key fields |
|---|---|
| `RespondInterventionAction` | respondingPlayerId, response: `Skip \| SkipAll \| Intervene` — the **Stage-1 decision response** (Phase Guard: `InterventionDecision` only). `Skip` → resume / continue; `SkipAll` → suppress this action's remaining windows (covers its ③′ **and** ⑥′, until the next action declaration — scoped to the CURRENT action only; distinct from the turn-spanning standing `PlayerState.autoSkipAll`, F15); `Intervene` (legal only if `candidates` non-empty **and** `turn.interventionsUsed < MaxInterventionsPerTurn`, else rejected `NoMatchingCard` / `NoInterventionBudget` — §2C, §4 ②) → `turn.interventionsUsed++`, `phase → InterventionExtension`, emit the public `InterventionWindowOpenedEvent { stage: Extension }` (the escalation tell) + the responder-directed extension prompt (timeoutSeconds = `InterventionExtensionWindow`). |
| `SubmitInterventionAction` | respondingPlayerId, chosenCardId? (null = **back out** — no card played, continues like a skip, but the intervene-press was already counted against the cap), targetIds[] — the **Stage-2 response** (Phase Guard: `InterventionExtension` only). Validated by the ordinary ③ membership check (`chosenCardId ∈ candidates`, `targetIds ⊆` that candidate's `targetIds`); the chosen card's `effectiveCost` is paid from the responder's `mana` (their pre-loaded off-turn pool — Turn Lifecycle). Resolves the played card (an **invocation** — it does **not** increment the invoker's `cardsPlayedThisTurn`, #23), then runs the re-declare (③′) / re-offer (⑥′) loop — a play re-opens a fresh Stage-1 decision window over the *remaining* offer (§4 "PendingIntervention Interruption"). |

**System/effect-issued** — emitted by `IEffect`/`ITrigger` implementations; processed through the same pipeline as player actions:

| Action | Key fields |
|---|---|
| `DrawCardAction` | playerId, sourceId? — single-card draw. ④ handler, three branches: **(a)** deck non-empty, hand has room → move the top card to hand, emit the owner-directed `CardDrawnEvent` + public thin `CardDrawnPublicEvent` (#2); **(b)** deck non-empty, hand full → burn it, emit `HandOverflowEvent`; **(c)** **deck empty → FATIGUE**: `fatigueCounter++`, emit `FatigueDamageEvent { amount = fatigueCounter }`, then enqueue `DealDamageAction { target = the player's hero, amount = fatigueCounter, sourceId = null }`. Fatigue is **sourceless** (null) even when an effect forced the draw, and carries **no special damage path** — routing through the one damage channel means it inherits ④ modifier precedence (so `PlayerState.armor` absorbs it automatically — the hero absorb slot), interception/⑥′ windows, and the hero → `HeroMortallyWoundedEvent` → ⑧ settle path (deck-out loss, incl. mutual-deckout draw, falls out for free). Empty-deck and full-hand are mutually exclusive per draw (no card drawn ⇒ nothing to burn). A multi-card "draw N" effect enqueues N `DrawCardAction`s, so each empty-deck one ticks fatigue once. |
| `DealDamageAction` | target: `entityId` **or** `ITargetSelector` (AoE — evaluated at ④ over the current board), amount, sourceId — **the one channel for all damage, combat included** (see §3 "Reactive…"). A selector-carrying instance is a single declared action → a single ③′ interception window. **Combat = two independent instances** (attacker strike + defender retaliation), each separately declared and processed sequentially — *not* one atomic unit (§3, 2026-06-06) |
| `HealAction` | targetId, amount, sourceId |
| `DestroyMinionAction` | minionId, sourceId? — the **fiat doom** (effect-text "destroy"; also what a Poisonous hook enqueues). ④ handler: sets the **marked-for-destruction flag** — no health change, **no event** (silent until `MinionDiedEvent` at the ⑦ settle; emphatically *not* `MinionMortallyWoundedEvent`, which is health-domain only — 2026-06-12, option A). **The mark is final:** no save clears it — heal is irrelevant (health untouched), and ⑦ collects a marked minion regardless of health (a marked minion that is *also* at ≤ 0 health is simply dead twice over). **The only counterplay is preventing the mark**: intercept this action at its ③′ declaration (cancel/retarget) before ④ applies it. Consumers of the flag: the §4 ③ aliveness check (a marked minion cannot *initiate* an attack; it still retaliates) and ⑦ Phase-1 collection. A future "cleanse the mark" effect would be an additive unmark action — rejected for v1 (Item-10 disposition: additive anytime). |
| `SummonMinionAction` | definitionKey, ownerId?, boardPosition, currentHealthOverride?, sourceId? — summons a fresh minion from the definition. **`currentHealthOverride` (#20):** null ⇒ summon at full `maxHealth` (the normal case); a non-null value sets `currentHealth` to it after the body is built (clamped to `[1, maxHealth]`). **Reborn** (§4 ⑦ Phase 3) is the sole v1 user — it passes `1`. |
| `GainArmorAction` | playerId, amount, sourceId? — the sole `armor` gain path (no cap): `armor += amount`, emit `ArmorGainedEvent`. Armor is *consumed* inside the `DealDamageAction` ④ handler (the precedence's hero absorb slot), never by this action |
| `EquipArtifactAction` | playerId, definitionKey, sourceId? — the one equip path: playing an Artifact **card** enqueues it, and effects (battlecries etc.) enqueue the same action. A **player play** onto a full row (3) rejects at ③ (`ArtifactSlotsFull` — discard first, the BoardFull analogue); an **effect-enqueued** equip onto a full row **fizzles** at ④ re-validation (the full-board-summon pattern). Handler: append to `artifacts[]`, register the definition's triggers/auras (zone-entry, `birthEpoch` stamped — §3), emit `ArtifactEquippedEvent`. **Match setup** equips each player's starter artifact via this action. |
| `DestroyArtifactAction` | artifactId, sourceId? — system/effect destruction + the durability-exhaustion path: unregister triggers/auras, snapshot → owner's graveyard (`GraveyardArtifact`), emit `ArtifactDestroyedEvent` (never `ArtifactDiscardedEvent`). |
| `ModifyArtifactChargesAction` | artifactId, delta, sourceId — the sole `charges` mutation path → `ArtifactChargesChangedEvent`. (The dual-artifact example: passive trigger "a neutral minion dies" enqueues `+1`; active "deal `charges` damage to target", durability 3 consumed by activation.) |
| `GiveCardAction` | playerId, definitionKey, sourceId? — fabricate a card into the player's hand (hand-cap rules as for draw); emit the owner-directed `CardAddedToHandEvent` + public thin `CardAddedToHandPublicEvent` (#2) |
| `ReturnToHandAction` | minionId, policy: MinionToCardPolicy, sourceId? — bounce a board minion back to its owner's hand (§1 "Minion → Card Transitions"). A bounce is **not a death** (no Deathrattle). **Full-hand rule (#21):** if the owner's hand is at `GameConstants.HandCap`, the minion cannot fit and is instead **destroyed** — the handler enqueues `DestroyMinionAction` so it dies normally and its **Deathrattle fires** (§4 ⑦). HS-faithful, and consistent: only a *death* fires a deathrattle, and a full-hand bounce is the only bounce that is a death. |
| `TransformMinionAction` | minionId, newDefinitionKey, sourceId? — **④ handler (#15): full reset** to `newDefinitionKey` (HS-faithful — transform is *replacement*, not death, so **no Deathrattle fires**). Rebuilds the body: `baseAttack`/`baseHealth`/`intrinsicKeywords`/`intrinsicTribes` from the new def; `enchantments → []`, `grantedKeywords → []`, `grantedTribes → Tribe.None`; `currentHealth = maxHealth` (full — **damage and buffs wiped**, aura bonuses recompute at ⑥). **State:** `summoningSick = true` (can't attack this turn unless the new def has Charge/Rush, pulled at §4 ③), `attacksUsedThisTurn = 0`, `isFrozen = false`, `rebornAvailable = true` (fresh body = unspent charge; whether it actually reborns is gated by the new def's `reborn` keyword — F5); `summonOrder` reset to a fresh monotonic value. **Triggers:** unregister the old `ITrigger`/Deathrattle/auras, register the new def's with a **fresh `birthEpoch`** (current epoch — so the new body does not fire on the transforming event). **Preserved:** `minionId` (in-place mutation — mid-cascade references stay valid; "newness" comes from the reset + fresh epoch), board position, `ownerId`/`bornNeutral` (a transformed neutral stays neutral). Emits `MinionTransformedEvent`. |
| `TakeControlAction` | minionId, newOwnerId, boardPosition?, sourceId? — **CONTROL** (permanent): re-homes the minion onto `newOwnerId`'s board and sets `ownerId`. The card's `ITargetSelector` defines legal `minionId` (neutral-only = `AllNeutralMinions`, enemy-only = `AllEnemyMinions`, either = `Union(...)`; no new selectors). Handler: **if the destination board is full** (`newOwnerId` already at `GameConstants.MaxBoardMinions`), the minion cannot be re-homed and is instead **destroyed in place** — it still carries its **original** `ownerId` (the full-board check precedes the re-home), so the handler enqueues `DestroyMinionAction` and it dies normally to its **original owner's** graveyard with its Deathrattle firing as that owner's (HS-faithful — a take-control onto a full board *destroys* the target, verified 2026-06-22 [Take control, hearthstone.wiki.gg]; this is the `ReturnToHandAction` full-hand-bounce pattern **#21**, **not** the full-board-summon *fizzle* — `TakeControl` moves a pre-existing minion, so there is a real entity to destroy, and "reject `BoardFull`" was wrong since the action is always effect-issued with no submitter to reject to, G4 2026-06-22). Otherwise (room exists) set `ownerId`, move zones, mark the minion **summoning-sick** (`summoningSick = true`) — whether it may act this turn is the ordinary pulled eligibility (Charge → any target, Rush → minions only) at §4 ③, *not* decided here — **re-bucket** its existing trigger subscriptions under the new owner (the bus orders firing by each host's owner vs. the active player, so however it tracks ownership it must now reflect `newOwnerId`) — this is **not** a re-run of `OnSummon`: triggers are neither re-created nor re-fired, and their `birthEpoch` is **preserved** (a moved entity, not a new one) — aura-recalc, emit `MinionControlChangedEvent`. The minion is now non-neutral and dies to `newOwnerId`'s graveyard. |
| `CommandAttackAction` | attackerId (a neutral), targetId, sourceId — **COMMAND** (one-shot, no zone change): a card-granted single attack *activation* by a neutral minion. Effect-issued. Re-validates at §4 ③ with the **relaxed attacker rule** (a neutral may attack only via this action; never via player `AttackAction`); legal `targetId` = anything a controlled minion could attack (opponent characters incl. hero, or another neutral) under the same **per-lane Taunt** rule. Honors the attacker's keywords: a **Windfury** neutral makes **two** strikes at the one target, **sequenced independently** — the second re-validates, so a mortal wound from the first retaliation fizzles it (§4 ③ aliveness). Ignores `attacksUsedThisTurn` and bypasses summoning-sickness entirely (neutrals are never `summoningSick`) — command is a card-granted activation, not the minion's own budget. Desugars into the same combat `DealDamageAction` pair(s) as `AttackAction`, so retaliation lands normally. |
| `BuffMinionAction` | minionId, attackDelta, healthDelta, sourceId — appends a permanent `StatModifier` to the minion's `enchantments` and recomputes `attack`/`maxHealth`; `currentHealth` follows the `maxHealth` change per the **#13 rule** (§1 `maxHealth`). |
| `SilenceMinionAction` | minionId, sourceId? — **④ handler (#14):** strip the minion to its base card text. **Clears:** `enchantments → []`, `intrinsicKeywords → []`, `grantedKeywords → []`, `grantedTribes → Tribe.None`, `isFrozen → false` (+ clear `frozenOnTurn` — Silence removes Freeze); **unsubscribes** the minion's `ITrigger`(s) + Deathrattle from the bus and **stops emitting** its `IAura`(s). Then recompute `attack`/`maxHealth` (now base + aura bonuses only) — `currentHealth` follows the `maxHealth` drop per the **#13 rule** (clamp to `min`, damage persists, ≤0 ⇒ mortally wounded). **Does NOT touch:** `intrinsicTribes`, `baseAttack`/`baseHealth`, `summoningSick` (Silence never wakes a minion), `currentHealth` *damage* (persists), `rebornAvailable` (F5/option-a — clearing the `reborn` keyword already gates off reborn; leaving the charge intact lets a *later* re-grant of Reborn fire), or a destroy-mark (final — Silence cannot cleanse it). `auraKeywords`/`auraTribes`/aura stat bonuses aren't stored, so they recompute at ⑥ — a silenced minion still *receives* future auras (Silence ≠ immunity). Emits `MinionSilencedEvent`. |
| `FreezeTargetAction` | targetId (a **minionId** — heroes cannot be frozen: heroes never attack (2026-06-11), so freeze has nothing to deny them), sourceId? — sets `isFrozen = true` and stamps `frozenOnTurn = turn.number` on the target; emits `MinionFrozenEvent`. Thawing is the end-of-turn unfreeze sweep (Turn Lifecycle step 3), never here. |
| `ModifyManaAction` | playerId, delta, sourceId? — the sole arbitrary-`mana` mutation path (**The Coin** = `delta +1`). **Clamping (#22):** `mana` floors at **0** (a negative delta cannot push it below 0) and has **no ceiling** — a gain MAY exceed `maxMana` (The Coin can leave the second player above their crystal count, HS-faithful). It never touches `maxMana` (the ramp is a turn-end concern, Turn Lifecycle step 4). Emits `ManaChangedEvent`. |
| `ShuffleDeckAction` | playerId, sourceId? |
| `DiscardCardAction` | playerId, cardId? (null = random), sourceId? |
| `SpawnNeutralMinionAction` | definitionKey? (null ⇒ draw one uniformly **with replacement** from `neutralZoneConfig.spawnPool` via this action's ④ `IRandom` — the neutral-handover refill path enqueues `maxCapacity − currentCount` of these with a null key, Turn Lifecycle step 5), position?, sourceId? — summons a minion into the neutral lane (`ownerId = null`, `bornNeutral = true`), entering the board through the **same routine as a regular summon** (registers its `ICardHandler` triggers/keywords; `summoningSick` is moot — neutrals have no turn). Emits the regular **`MinionSummonedEvent` with `ownerId == null`** — *not* a neutral-specific event — so summon-watching mechanics fire out of the box. A summon, not a play ⇒ no Battlecry (matches `SummonMinionAction`). **Full-lane rule (#19):** spawning into a lane already at `neutralZoneConfig.maxCapacity` **fizzles** at ④ re-validation (the full-board-summon pattern — these actions are always system/effect-issued, never a player play, so they fizzle rather than reject). The handover refill path never overfills (it enqueues exactly `maxCapacity − currentCount`); the fizzle bites only an *effect*-issued spawn racing a full lane. |
| `StartChoiceAction` | waitingPlayerId, choiceType, options[], context. Its ④ handler emits the chooser-directed `ChoicePromptEvent` (options) + the public thin `ChoiceStartedEvent` (optionCount) — §2B "Directed events" |
| `StartInterventionAction` | **system action** (bypasses Phase Guard) that opens a **Stage-1 decision window** for the non-active responder: respondingPlayerId (**always the non-active player** — §3 responder rule), heldAction? (null = ⑥′ post-reaction window — e.g. a dying-window save), candidates[] (`{cardId, targetIds[]}` — the responder's matching + affordable hand cards, **MAY be empty**: the window is UNCONDITIONAL, §3), causeEventIds[]? (⑥′ only — the matched events the window is *about*), timeoutSeconds (`InterventionDecisionWindow`). Sets `phase → InterventionDecision`; emits the responder-directed `InterventionPromptEvent` + the public `InterventionWindowOpenedEvent { stage: Decision }` (§2B). Opened only when `turn.interventionsUsed < MaxInterventionsPerTurn` **and** the responder's `autoSkipAll` is false (§4 ③′/⑥′). |
### 2B. GameEvent types

All events carry an `eventId: long` (append-order sequence number, unique per session — the stable reference other artifacts use to point at an event, e.g. the ⑥′ intervention prompt's `causeEventIds[]`), an `OccurredAt` timestamp, and an `originEpoch: int` (the `currentActionEpoch` of the action that produced the event; used by `IEventBus` for creation-epoch visibility filtering — see §3 `IEventBus`). Events are append-only.

**Bus ⊋ log.** Almost every event below is a **handler-produced state delta** (emitted at ⑤) and populates the append-only **event log** — the client wire format and replay archive. The bus, however, carries a *superset*: the lone **`ActionDeclaredEvent`** is a transient **pre-execution interception signal** (emitted at ③′, before any state change) that rides the bus only so `ITrigger`s can subscribe — it is **excluded from the persisted event log and the client wire format**, because it represents intent, not a delta. So "the event log is exactly the state-delta stream" stays true even though the bus delivers one extra signal type.

**Directed events (2026-06-10, intervention visibility).** Most events broadcast to both players' wire streams (and spectators/replay). A few are **directed**: delivered only to one player's wire and absent from the opponent's and the spectator stream — the pattern the mulligan events already used ("each with only their own options"). Directed in v1: `MulliganStartedEvent` is directed **per-player** (each player sees only their own options) with **no** public sibling — the bare fact "mulligan started" needs no separate broadcast (both players are mulliganing simultaneously and each already received their own directed event; F12). The other **five** directed events **each** pair with a public thin sibling where the opponent/spectators still need the bare fact: `ChoicePromptEvent`, `InterventionPromptEvent`, `SigilInscribedEvent`, `CardDrawnEvent`, `CardAddedToHandEvent` (the last two added 2026-06-13, #2) → `ChoiceStartedEvent`, `InterventionWindowOpenedEvent`, `InscriptionAddedEvent`, `CardDrawnPublicEvent`, `CardAddedToHandPublicEvent` respectively. The library marks directedness on the event type; the Game Server's per-player fan-out honors it (the split exists precisely so a prompt's private payload — Discover options, intervention candidates — never crosses to the other wire). `eventId`s are assigned globally at append, so directed filtering never renumbers anyone's stream.

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
| `CardDrawnEvent` | **directed → owner**: playerId, card — the drawer sees the card; everyone else gets the public thin `CardDrawnPublicEvent` (2026-06-13, #2 — closes the hand-reveal leak: a single broadcast event carrying `card` would expose the drawn card to the opponent, contradicting ⑥′'s own no-expose rule) |
| `CardDrawnPublicEvent` | **public thin**: playerId — "drew a card" (identity hidden); the sibling broadcast to the opponent + spectators |
| `CardAddedToHandEvent` | **directed → owner**: playerId, card (given by effect, not drawn) — owner sees the card; everyone else gets the public thin `CardAddedToHandPublicEvent` |
| `CardAddedToHandPublicEvent` | **public thin**: playerId — "gained a card" (identity hidden) |
| `HandOverflowEvent` | playerId, burnedCard — **public** (HS reveals the burned card to both players) |
| `FatigueDamageEvent` | playerId, amount — the empty-deck draw outcome (sibling of `HandOverflowEvent`): announces the `fatigueCounter` bump (amount = the new counter) and telegraphs the impending self-damage. Fires at the `DrawCardAction` ④ handler **before** the enqueued fatigue `DealDamageAction` lands its `DamageTakenEvent` (same shape as `AttackPerformedEvent` → `DamageTakenEvent`). The dedicated event — not a flag on `DamageTakenEvent` — is how the client/telemetry knows a hero hit was fatigue (no damage-`cause` discriminator exists; see §2B Mana/Hero note). |
| `CardPlayedEvent` | playerId, card, targetId? — emitted at `PlayCardAction` **④** (commitment; 2026-06-13, #34). **Not** emitted for an inscribe play (identity-hidden; the §2B sigil events replace it — §3 "Sigils") |
| `CardPlayFizzledEvent` | playerId, cardId — a committed card play whose enqueued `ResolveCardAction` was **cancelled** (true Counterspell) or became **illegal** on resume: effects never run and there is **no refund** — mana stays spent, the card stays out of hand, any spell graveyard entry stays (the commitment deltas already fired at the play's ④). §2A `PlayCardAction`/`ResolveCardAction`, 2026-06-13, #34 |
| `CardDiscardedEvent` | playerId, card — **public** (HS reveals a discarded card to both players) |
| `CardReturnedToHandEvent` | playerId, card (bounced from board) |
| `CardShuffledIntoDeckEvent` | playerId, card |

**Board:**

| Event | Key fields |
|---|---|
| `MinionSummonedEvent` | minion, ownerId? — **the single summon event for every lane, neutral included** (`ownerId == null` ⇒ summoned into the neutral lane). A neutral spawn (`SpawnNeutralMinionAction`) emits this same event, so any "when a minion is summoned" trigger — including a lane-based `FriendlyOnly` one on a neutral minion reacting to its lane-mates — catches it out of the box. There is **no** separate neutral-summon event (the principle: the neutral lane behaves like a regular lane so mechanics work without special-casing). |
| `MinionMortallyWoundedEvent` | minionId, sourceId — fires the instant a minion enters **health-domain** pending-death (`currentHealth ≤ 0` — from damage, or from a buff/aura loss collapsing `maxHealth` to ≤ 0), **before** the death-wave settle removes it; distinct from `MinionDiedEvent` (Phase-1 removal). The hook for dying-window reactions — e.g. a **save-from-lethal** intervention (the "R1" avenue: raise a mortally-wounded minion's effective health back above 0 before the settle). **Savability caveat (F16):** only a **damage-domain** wound (`currentHealth ≤ 0 < maxHealth`) is **heal**-saveable; a **maxHealth-collapse** wound (`maxHealth ≤ 0` from aura/buff loss) is **not** — `HealAction` caps at `maxHealth` (≤ 0), so a heal is a no-op. The window still opens (the event fires either way), but the only save for a collapse wound is **restoring `maxHealth`** (the aura returning, a re-buff). So "savable" ≠ "heal-saveable." **A destroy-mark does NOT fire this event (narrowed 2026-06-12, option A):** fiat dooms are unsavable, so no dying window may open to bait a futile save — a marked minion is silent until `MinionDiedEvent` at the settle; its counterplay window was the `DestroyMinionAction`'s own ③′ declaration. (No `cause` discriminator — and with marks excluded, every remaining cause is a health crossing, genuinely indistinguishable-by-need; see Mana/Hero note below.) |
| `MinionDiedEvent` | minionId, snapshot, sourceId, diedOnTurn |
| `MinionTransformedEvent` | minionId, newCard |
| `MinionControlChangedEvent` | minionId, fromOwnerId? (null = was neutral), toOwnerId — emitted by `TakeControlAction` after the minion is re-homed onto its new owner's board (summoning-sick on arrival; it may still act this turn if it has Charge/Rush — pulled at §4 ③). A commanded attack (`CommandAttackAction`) does **not** emit this — it changes no owner — it rides the ordinary combat events. |
| `NeutralZoneRepopulatedEvent` | spawnedMinions[] — **renderer-only batch marker** for a neutral-zone repopulation at the **Interturn handover** (§4 Turn Lifecycle step 5 — the neutral lane's own "turn", `activePlayerId = none`); carries no mechanic. The actual per-minion summons go through `SpawnNeutralMinionAction` → one `MinionSummonedEvent` each (so summon-watching triggers fire normally); this event is an optional summary the client may use to animate the batch. *(The former separate `NeutralMinionSpawnedEvent` was removed — a neutral spawn is just a `MinionSummonedEvent` with `ownerId == null`.)* |
| `DeathWaveStartedEvent` | waveIndex |
| `DeathWaveEndedEvent` | waveIndex, minionsResolved |
| `StabilizationAbortedEvent` | wavesReached, lastWaveMinionIds[] — fatal; always immediately followed by `GameEndedEvent { reason: NoContest }` |

**Combat:**

| Event | Key fields |
|---|---|
| `AttackDeclaredEvent` | attackerId, targetId — the combat-specific **declaration**, published at `AttackAction` **③′** (the *intended*, pre-interception target): the typed hook for "when this attacks" interception (e.g. a redirect). **Bus-only: not a state delta — excluded from the persisted event log and client wire format** (treated exactly like `ActionDeclaredEvent`; 2026-06-13, #5 — preserves the "event log = state-delta stream" invariant, no third visibility class). Renderer coverage without it: no window → `AttackPerformedEvent` (④, committed) anchors the animation with no perceivable gap; during a window → the directed `InterventionPromptEvent` + the public `InterventionWindowOpenedEvent` (which carries the cause). May still fizzle/retarget before ④. |
| `AttackPerformedEvent` | attackerId, targetId — the **committed** swing, emitted at `AttackAction` **④** (the post-interception *actual* target). Announces the activation-consumed delta (`attacksUsedThisTurn++`) and is the natural anchor for a future "after this attacks" trigger. The swing's **damage is not here** — it lands via the two enqueued `DealDamageAction`s' own `DamageTakenEvent`s. **Replaces the removed `AttackResolvedEvent`**, which bundled both hit amounts into a single delta — impossible under two-action combat, where neither hit has landed at ④ and a card may interleave between them. |
| `DamageTakenEvent` | targetId, amount, armorAbsorbed, overkill, sourceId — `armorAbsorbed` = the portion eaten by hero `armor` at the ④ absorb slot (always 0 for minion targets / unarmored heroes); health loss = `amount − armorAbsorbed`. A **field**, not a separate armor-loss event — same idiom as `overkill` (one occurrence, split delta), distinct from the disjoint-event idiom (discard vs destroy), which is for *different occurrences* |
| `HealedEvent` | targetId, amount, sourceId |

**Status / Keywords:**

| Event | Key fields |
|---|---|
| `MinionSilencedEvent` | minionId |
| `MinionFrozenEvent` | minionId |
| `MinionThawedEvent` | minionId — frozen status cleared by the end-of-turn unfreeze sweep (Turn Lifecycle step 3). |
| `DivineShieldBrokenEvent` | minionId |
| `StealthBrokenEvent` | minionId — Stealth keyword cleared because the minion attacked (`AttackAction` ④). Parallels `DivineShieldBrokenEvent` (a one-time keyword consumption). |

**Mana / Hero:**

| Event | Key fields |
|---|---|
| `ManaChangedEvent` | playerId, mana, maxMana |
| `HeroMortallyWoundedEvent` | playerId, sourceId — fires the instant a hero crosses to ≤0 health (from any damage — combat, spell, fatigue, self-damage), **before** the ⑧ win-check finalizes the loss; the hero analogue of `MinionMortallyWoundedEvent`, distinct from `GameEndedEvent` (the ⑧ finalization). The hook for save-the-hero reactions (heal-above-0 / immune in response to lethal — e.g. an Ice-Block-style effect realised as a post-reaction rather than damage prevention; for prevention see "Interception", §3). **No `cause` discriminator:** it was informational only (no card branches on it; save-from-lethal fires regardless of cause) and nothing in the engine populated it. Fatigue is identified by the `FatigueDamageEvent` that immediately precedes its `DealDamageAction`; other causes are reconstructable from the source/preceding events. Dropped rather than fed by a damage-cause taxonomy (YAGNI). |
| `ArmorGainedEvent` | playerId, amount — armor *loss* is not a separate event: absorption rides `DamageTakenEvent.armorAbsorbed` |

**Artifacts** (replace the HeroPower events, 2026-06-11):

| Event | Key fields |
|---|---|
| `ArtifactEquippedEvent` | playerId, artifact |
| `ArtifactActivatedEvent` | playerId, artifactId, targetId? — the activation delta and the subscription point for `OnArtifactActivated` triggers (the renamed Inspire; no separate `InspireTriggeredEvent`-style marker — the activation event itself is the hook) |
| `ArtifactDurabilityLostEvent` | artifactId, remainingDurability — the `remaining: 0` tick immediately precedes the enqueued `DestroyArtifactAction` |
| `ArtifactChargesChangedEvent` | artifactId, newCharges, delta |
| `ArtifactDiscardedEvent` | playerId, artifactId — **voluntary** removal only (owner's in-turn discard). Disjoint from `ArtifactDestroyedEvent` — never both; "leaves play for any reason" subscribes to both |
| `ArtifactDestroyedEvent` | playerId, artifact (snapshot) — **involuntary** removal: durability exhaustion or a destroy effect |

**Sigils** (§3 "Sigils", 2026-06-11):

| Event | Key fields |
|---|---|
| `SigilInscribedEvent` | **directed → owner only**: playerId, card — the full echo of the inscribed card (mulligan-style delivery; §2B "Directed events") |
| `InscriptionAddedEvent` | playerId — the **public thin form**: existence/count only, identity stays hidden (the face-down tell; unmaskable anyway — the mana delta is public state) |
| `SigilRevealedEvent` | playerId, card — **public** reveal at fire time, emitted immediately before the inscription's response actions resolve; the card then moves `inscriptions` → graveyard (one-shot; `GraveyardSpell` as-is) |

**Effects / Triggers:**

| Event | Key fields |
|---|---|
| `SpellCastEvent` | playerId, **card**, targetId? — carries the **full committed card snapshot** (mirrors `CardPlayedEvent`, G2 2026-06-22), threaded from commit via `ResolveCardAction.card`; this is what lets a `Subject(p)` cast-trigger read the spell's `effectiveCost`/`modifiers` (`Subject(CostAtLeast(5))` = "whenever you cast a 5+ spell that resolves") — by publish time the card is a `GraveyardSpell` (`definitionKey` only), so the event-carried snapshot is the sole surviving source of the cost. Fires at the spell's **resolution** (`ResolveCardAction` ④), the spell-side analog of `MinionSummonedEvent`; **not** at commitment (D2, 2026-06-21). So "whenever/after you cast a spell" cast-synergy fires **only if the spell resolves**: a spell **countered** at `ResolveCardAction`'s ③′ fires `CardPlayedEvent` (played, mana spent, → `GraveyardSpell`) but **no** `SpellCastEvent` — matching current HS, where Counterspell suppresses *all* cast triggers (verified 2026-06-21; the pre-2018 "whenever" early-fire was patched out). The **pre**-resolution declaration is the ③′ `ActionDeclaredEvent` (the interception / Counterspell hook, §3 pillar 2); cast-synergy is a **post**-reaction and belongs on this event. |
| `BattlecryTriggeredEvent` | minionId |
| `DeathrattleTriggeredEvent` | minionId |
| `ComboTriggeredEvent` | cardId, playerId |
| `ActionDeclaredEvent` | action — the universal **pre-execution** signal: published at stage **③′** (§4) for every dispatched action *after* it validates and *before* it executes (④), carrying the action itself. The single hook for **interception** (declaration-phase) reactions; subscribers filter on the carried action's runtime type/params via the condition library, so **no per-action-type `*Declared` taxonomy is required**. Published afresh on **each (re-)declaration**: a held action resumed after a responder *played* an intervention re-validates and, if still legal, **re-declares** — re-publishing this event so the responder's remaining reactive cards get another window (the session-6 **re-declare loop**, §4 ③′); a **skip/timeout** resumes the held action without re-declaring. Redirect/inscription loops are bounded not by suppressing re-declaration but by the **depth-1 nesting cap** (a response opens no further *player* window on itself) plus the responder's finite mana. **Bus-only: not a state delta, so excluded from the persisted event log and client wire format** (see the "Bus ⊋ log" note above). (The typed `AttackDeclaredEvent` is the combat-specific declaration at the same ③′ timing — the typed "when this attacks" interception hook, bus-only per #5; it is *not* a renderer convenience and *not* the general hook — `ActionDeclaredEvent` is the general hook, `AttackDeclaredEvent` the typed combat-specific one, both mechanically load-bearing.) |

**Choice / Intervention:**

| Event | Key fields |
|---|---|
| `ChoiceStartedEvent` | waitingPlayerId, choiceType, optionCount — **public thin form** (2026-06-10): "choosing among N" is all anyone else sees. The options themselves are deliberately **not** here — a broadcast `options[]` would leak Discover options to the opponent |
| `ChoicePromptEvent` | **directed → waiting player only**: waitingPlayerId, choiceType, options[] — the renderable prompt (mulligan-style directed delivery; §2B "Directed events") |
| `ChoiceResolvedEvent` | waitingPlayerId, selectedOptionId — public; opaque to the opponent without the directed `options[]` |
| `InterventionWindowOpenedEvent` | respondingPlayerId, phase (③′ \| ⑥′), **stage** (`Decision` \| `Extension`), **cause**, timeoutSeconds — **public**. `stage: Decision` fires at EVERY ③′/⑥′ window the engine opens — unconditional, so it conveys **no** information about whether a matching card is held. `stage: Extension` fires when the responder presses **intervene** — the **explicit, accepted escalation tell** (2026-06-16, replacing the old contingent-window tell): the acting player learns a matching reactive card exists, never which. The **cause** is the **held action verbatim** at ③′ / `causeEventIds[]` refs at ⑥′ — the *actor's* info (public-by-nature, session-8 ruling), never the responder's candidates; the inscribe-redaction carve-out applies. Carrying the cause closes the spectator/replay gap for a **cancelled no-delta action** (2026-06-13, #5/#34 window-cause rider) |
| `InterventionPromptEvent` | **directed → responder only**: respondingPlayerId, cause (the **held action verbatim** at ③′ \| `causeEventIds[]` at ⑥′ — refs into the responder's own wire, never re-embedded payloads), candidates[] (`{cardId, targetIds[]}`, engine-computed), timeoutSeconds — everything the client needs to render the window with no game logic (§3 visibility; §4 ③′/⑥′) |

### 2C. Action Rejection

When validation (§4 ②③) fails, the action is rejected with a **structured code** — never an English string baked into the engine. A rejection is the negative arm of `Submit`'s return; it is **not** a `GameEvent`, never touches the event bus, and never enters the event log. (Rejected actions cause no state change, so on replay they simply re-reject — consistent with Item 6's "log = real state-changing player actions only".)

```csharp
enum ActionRejectionCode {
    WrongPhase, NotActivePlayer, CardNotInHand, NotEnoughMana,
    InvalidTarget, TargetStealthed, MustTargetTaunt,
    AttackerNotAlive, AttackerNotControlled, AttackerCannotAttack, RushCannotTargetHero, AttackerFrozen, AttackerExhausted, BoardFull,
    ArtifactSlotsFull, ArtifactNotActive,
    InscriptionSlotsFull, SigilAlreadyInscribed,
    NoInterventionBudget, NoMatchingCard,   // §4 ② Intervene guards — exhausted press budget / no matching+affordable candidate (F8)
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

A small set of interfaces forms the engine: the **extensibility layer** (`IKeyword`, `IEffect`, `ITrigger` with its `ITriggerCondition`, the shared `IEntityPredicate`, `ITargetSelector`, `IAura`, `IWinCondition`, `ICardHandler`) plus the **infrastructure backbone** (`IActionHandler`, `IEventBus`, `IActionQueue`, `IRandom`).

### `IActionHandler<TAction>`

One handler per action type. Receives an action and the current `GameState`, mutates state, and returns the events produced. All state mutation in the engine happens inside action handlers — nowhere else.

```csharp
interface IActionHandler<TAction> where TAction : GameAction {
    IEnumerable<GameEvent> Handle(TAction action, GameState state);
}
```

### `IEventBus`

Broadcast backbone. Dispatches in the **canonical board order** (see Deterministic Ordering): three ordered groups per event type — the current (active) player's triggers, then the opponent's, then the **neutral zone's** — fired in that sequence, and within each group sorted by current board index at publish time (not at subscription time, so a minion that fills a vacated slot fires in the position it currently occupies). **Within each player's group, that player's artifact-hosted triggers fire after their minion-hosted ones** (artifact slot order `artifacts[0..2]` as the within-row index, mirroring board index; 2026-06-11), **and their inscription-hosted triggers after those** (inscription order `inscriptions[0..2]`; §3 "Sigils" — subject to the actor-attribution skip rule there) — artifacts and inscriptions append to their owner's existing bucket rather than forming new global groups, which preserves the active-player-first promise (an active player's artifact fires before any opponent trigger). The neutral group is the trigger-side counterpart of the neutral zone's last slot in the death sort; it makes trigger fire order and death sort order **one rule**. **Exception — the neutral handover (§4 Turn Lifecycle step 5, "Interturn"):** that step runs as the **neutral lane's own turn** with `turn.activePlayerId = none` (null). With no active player, the active and opponent groups are **empty** — so *neither* player's triggers fire on the neutral-lane repopulation, **only the neutral group does**. This is the exact dual of neutral minions being excluded from player turns (above): players are excluded from the neutral lane's turn. No cross-player ordering exists to bias — strictly fairer than any fixed seat order.

This applies to **board-wide event triggers** (on-damage, on-death, on-spell-cast, on-summon, …) — a neutral minion's `FriendlyOnly`-conditioned trigger fires off its **lane-mates** (friendly is lane-based; see `ITriggerCondition`), so neutral minions trigger off each other just as a player's do. It does **not** make neutral minions fire **turn/hero-context** triggers, which stay empty for want of a referent (not because of the friendly relation): **Start/End-of-Turn** (enumerated per player in the Turn Lifecycle — a neutral minion has no turn of its own), **OnArtifactActivated** (the renamed Inspire — "after *you* activate an artifact"; a neutral host has no artifact row of its own), **Combo** (no hand / turn play-count). This is all condition semantics, not a bus special-case. (Artifacts themselves, by contrast, participate in turn-scoped triggers normally — they have an owner with turns.)

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

**Invariant — subscriptions happen only inside action handlers.** Every bus subscription occurs during stage ④ (`ICardHandler.OnSummon` registering this card's `ITrigger`s), so every listener has a well-defined `birthEpoch`. Nothing subscribes during event publication (⑤) or aura recalculation (⑥). This invariant is what makes the epoch comparison total. **Note (F19, simplified 2026-06-21):** a *card-played* minion subscribes at `ResolveCardAction`'s **own** ④ (epoch N) — the same ④ that places the body and returns `MinionSummonedEvent` (originEpoch N) — exactly as a token does via `SummonMinionAction`. The equal-epoch rule (`birthEpoch N < originEpoch N` is **false**) keeps it from reacting to its own summon. Its Battlecry, however, is **enqueued** and runs at **later** epochs (N+1, …), and since `N < N+1` the minion **does** receive — and react to — its own Battlecry's events. That self-Battlecry reaction is a **deliberate divergence** from HS's "Battlecry before triggers go live" (§2A `ResolveCardAction` ④; "Unaddressed Features → A minion triggers off its own Battlecry"). (Keywords do **not** subscribe — their behaviour is pulled at stage ④, see §3 `IKeyword` — so the bus carries only `ITrigger` subscriptions.)

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
| Spell Damage +X | *pulled* and aggregated at spell-resolution start (`EffectContext.spellDamageBonus`); magnitude in the keyword string (`spell_damage:1`) |
| Reborn | read by `DeathResolutionService` Phase 3 (`keywords has reborn && rebornAvailable`); firing consumes `rebornAvailable`, the keyword persists (counts for auras, survives bounce/copy) |

**2. Role-interface hooks** — a keyword whose behaviour *enqueues a follow-on action* at a specific moment implements a small hook interface. The owning action handler iterates the acting minion's **effective** keywords implementing that interface and invokes each — no subscription, all inside stage ④:

```csharp
interface IKeyword { string KeywordId { get; } }

// Invoked by DealDamageAction over the SOURCE's effective keywords, after damage applies.
interface IOnDealtDamage : IKeyword {
    void OnDealtDamage(string sourceId, string targetId, int amount,
                       GameState state, IActionQueue queue);
}
```

v1 hooks (`IOnDealtDamage`): **Lifesteal** → enqueue `HealAction` for the source's owner — **a no-op when the source is unowned (`ownerId == null`, a neutral minion): there is no hero to heal** (2026-06-13, #4); **Poisonous** → enqueue `DestroyMinionAction` on **any minion it damages, heroes excepted** (2026-06-13, #3: the relational "enemy" scope is removed — the lane model made "enemy" wrong for a neutral source; a minion is destroyed by the venom whichever side dealt it). The handler:

```csharp
ApplyDamage(target, amount);
foreach (var kw in EffectiveKeywords(source).OfType<IOnDealtDamage>())
    kw.OnDealtDamage(source, target, amount, state, queue);
```

Further "own-moment" hook points (on-take-damage, on-attack, …) are added as new role interfaces when a card first needs one; each is invoked the same way by its owning handler. **Freeze** is not a keyword hook — it is a status: `FreezeTargetAction` sets `isFrozen = true` and stamps `frozenOnTurn = turn.number` (`MinionFrozenEvent`), and a frozen minion cannot attack (§4 ③ `AttackerFrozen`). It thaws in the **end-of-turn unfreeze sweep** (§4 Turn Lifecycle step 3), which thaws the ending player's frozen minions **unless** the freeze landed *this* turn while the minion was already exhausted — so Freeze costs **exactly one attack**: a minion frozen with an attack still available misses that turn and thaws at its end, while one frozen *after* it has swung (e.g. into a retaliation-freezer) stays frozen through its next turn. Freeze is **minion-only**: heroes never attack (2026-06-11), so a hero has no attack for freeze to deny — there is no hero frozen state.

### `IEffect`

Atomic unit of "do something to game state." Used by card definitions and trigger implementations. Effects enqueue actions — they never mutate state.

```csharp
interface IEffect {
    void Execute(EffectContext context, IActionQueue queue);
}
```

`EffectContext` carries: `sourceId`, `sourcePlayerId`, `targetId?`, a read-only `GameState` reference, and `spellDamageBonus: int`. (`sourceId` is the unified entity reference — formerly `sourceCardId`, generalized because a triggered effect's source is often a *minion*, not a card. See "Source Attribution" below.)

#### Source Attribution

Every action carries a `sourceId`, and `EffectContext` is built from the **action's own** source fields. Attribution follows one rule:

> **`sourceId` is the entity whose effect this is — set by whoever *enqueues* the action — and is never inherited from the upstream cause.**

- A played card's effects (including Battlecry) → `sourceId` = the card.
- A minion's triggered effects (Deathrattle, On-Damage, …) → `sourceId` = that minion. A trigger's `OnFire` stamps `sourceId = its host entity`; it does **not** pass along the source of the action that fired the trigger.
- Artifact effects → that artifact.
- `sourcePlayerId` = the source's controller, captured at enqueue time (from the death snapshot if the source is already dead).

So if Yeti's Deathrattle damages a minion, the damage's `sourceId` is **Yeti**, not the spell that killed Yeti. This determines friendly/enemy evaluation (Item 2 conditions), the Lifesteal heal target, and which entity "dealt" the damage for any watching trigger.

**Attribution ≠ keyword application.** `sourceId` is *data* on the event, so a watcher correctly credits Yeti even though Yeti is dead. But Lifesteal/Poisonous are **keyword hooks pulled from the source's effective keywords** (§3 `IKeyword`), and a dead Yeti is off the board — so `EffectiveKeywords(source)` is empty and the hooks do **not** fire for Yeti's own post-death Deathrattle damage. No listener lifecycle is involved; this falls out of "read effective keywords from live board state" with no special-casing. See "Unaddressed Features" for the case where a card would *want* dead-source keyword effects (enabled by a death-snapshot keyword fallback).

**`spellDamageBonus`** is a read-only value computed **once at the start of spell resolution** and held on the context for the whole cast. Every `DealDamageEffect` in that spell reads the *same* snapshot and bakes the final number into its enqueued `DealDamageAction` (the action handler stays dumb — it applies the number it's given). Because the bonus is spell-scoped, not hit-scoped: a spell that deals damage twice still applies the bonus to both hits, and a spell that kills its own Spell-Power minion mid-cascade does not lose the bonus for its second half. It is `0` for any non-spell source, so it cannot leak into combat or artifact-activation damage.

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

Covers the full trigger catalog: Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous, via `IAura`), On Damage Taken, On Friendly Minion Death, On Spell Cast, On Artifact Activated (the renamed Inspire — fires off `ArtifactActivatedEvent`), Combo. New trigger types are added by implementing this interface. `ShouldFire` is typically delegated to an `ITriggerCondition` (below) rather than hand-written per card.

### `IEntityPredicate`

The shared **entity-property** layer: a predicate answering *"does THIS entity satisfy P, relative to the source?"* It is consumed directly by `ITargetSelector.Filter` (which entities does an effect act on?) **and** by `ITriggerCondition` (does the event's relevant entity satisfy P?). Factoring it out (F4 → option B, 2026-06-17) writes each property check **once** and gives both worlds the same `All`/`Any`/`Not` tree — crucially enabling **negation inside a `Filter`** ("all non-Beasts", "damaged minions that aren't Legendary"), which the set-algebra selectors (`Union`/`OtherThan`) cannot express.

```csharp
interface IEntityPredicate {
    bool Matches(EntityId subject, GameState state, string sourceId);  // sourceId = the source/host entity, for relational checks (Friendly/Enemy)
}
```

**Friendly/enemy is LANE-based** — by `ownerId`, treating the three lanes (player 1's board, player 2's board, the neutral lane) as the three "sides," with `null == null` meaning "same side." For source `s` and the tested subject `e`:

```
Friendly(s, e)  ⟺  s.ownerId == e.ownerId          // null == null → neutral lane-mates are friendly
Enemy(s, e)     ⟺  s.ownerId != e.ownerId  &&  e.ownerId != null
```

- **`Friendly`** = same `ownerId` as the source. For a player source this is its controller's board (Hearthstone-identical, since your board == minions you control); for a **neutral source (`ownerId == null`)** it is the **neutral lane** (the other `ownerId == null` minions). So a neutral minion's "friendly" effects target its lane-mates.
- **`Enemy`** = an *opposing* side. For a player source: the **opponent's board only** — the neutral lane is **not** enemy (the separate `AllNeutralMinions` category). For a **neutral source**: **both** players' boards. The `e.ownerId != null` clause makes a neutral minion **never anyone's enemy**. `Friendly`/`Enemy` are mutually exclusive; exhaustive only from a neutral source (a player source has the neutral "neither" bucket). Heroes carry their player's id, so the same predicates classify friendly/enemy heroes; neutral has no hero.

**Catalog — parameterless:** `Friendly`, `Enemy`, `IsSelf` (`subject == sourceId`), `IsDamaged` (`currentHealth < maxHealth`), `IsMortallyWounded` (`currentHealth ≤ 0`, still on board).
**Parameterized factories:** `HasTribe(Tribe)` (intersects `effectiveTribes` — aura-granted tribes match while the aura holds), `HasKeyword(string)` (effective keywords), `MinionTypeIs(definitionKey)`, `CardTypeIs(CardType)`, `CostAtLeast(n)` / `CostAtMost(n)` (a card's `effectiveCost`), and further property tests (`AttackAtLeast(n)`, …) added as cards need them.
**Combinators** (compose into a tree, the same shape reused by `ITriggerCondition`): `All(p1, …)` (AND; `All()` = always true), `Any(p1, …)` (OR; `Any()` = always false), `Not(p)`.

### `ITriggerCondition`

Reusable, composable predicates answering one question: **"should this trigger fire?"** They are the **event** layer built on `IEntityPredicate` (above) — separated from `ITrigger` so the common firing filters are written once and shared across every card. An `ITrigger.ShouldFire` typically delegates to a single condition or a composed tree.

```csharp
interface ITriggerCondition {
    bool Matches(GameEvent evt, GameState state, string sourceId);
}
```

`sourceId` is the trigger's host entity — the hosting **minion, artifact, or sigil (in hand or inscribed)** (2026-06-13, #10b). A condition is one of two kinds:

- **event-structural** — about which *role* an entity plays in the event (`SelfIsSource` / `SelfIsTarget` / `SelfIsRelated`); no entity-predicate form.
- **subject-property** — `Subject(p)`: apply an `IEntityPredicate` `p` to the event's **relevant entity** (the dying minion in `MinionDiedEvent`, the cast card in `SpellCastEvent`, …). The named conditions below (`FriendlyOnly`, `HasTribe`, `CostAtLeast`, …) are exactly `Subject(p)` for the predicate of the same name — so the lane-based **Friendly/Enemy** relation and every tribe/cost/type check are **shared with selectors**, never re-implemented here.

Genuinely **controller/turn-context** conditions are a *different* axis and stay empty for a neutral host because it lacks the referent, not because of the friendly relation: Start/End-of-Turn (no turn of its own — §3 `IEventBus`), OnArtifactActivated (no artifact row of its own), Combo (no hand / turn play-count). The **`Combo`** condition's predicate is **`host.owner.cardsPlayedThisTurn ≥ 2`** — explicitly **≥ 2, not > 0**, because the Combo card increments the counter at its own `PlayCardAction` ④ (commit) *before* the battlecry evaluates at `ResolveCardAction` ④ (resolution), so the card is already counted (G1, 2026-06-21; §1 `cardsPlayedThisTurn`). HS-faithful reading: "you played *another* card before this one this turn."

**Event-structural conditions — parameterless:**

| Condition | Matches when |
|---|---|
| `AlwaysFires` | always |
| `SelfIsSource` | the event's acting/source entity is `sourceId` |
| `SelfIsTarget` | the event's target entity is `sourceId` |
| `SelfIsRelated` | the event's primary subject (e.g. the dying minion in `MinionDiedEvent`) is `sourceId` (≡ `Subject(IsSelf)`) |

**Subject-property conditions** — each is `Subject(p)` over the `IEntityPredicate` of the same name (§3 `IEntityPredicate`), applied to the event's **relevant entity**:

| Condition (sugar for) | Matches when |
|---|---|
| `FriendlyOnly` (`Subject(Friendly)`) | the relevant entity is on the **same side** as `sourceId` — for a neutral host, the neutral lane |
| `EnemyOnly` (`Subject(Enemy)`) | … on an **opposing side** — for a player host, the opponent's board (not neutral); for a neutral host, either board |
| `HasTribe(Tribe)` | the relevant minion's `effectiveTribes` intersects the flags — "whenever a **Beast** dies" |
| `MinionTypeIs(definitionKey)` | the relevant minion has that *specific* `definitionKey` |
| `CardTypeIs(CardType)` | the relevant card is a Minion / Spell / Artifact — "whenever you play a Spell **or** an Artifact" |
| `CostAtLeast(n)` / `CostAtMost(n)` | the relevant card's `effectiveCost` meets the threshold |

**Combinators** — the same `All` / `Any` / `Not` tree as `IEntityPredicate`, here composing conditions. Examples: *"whenever a friendly Murloc or Beast dies"* = `All(FriendlyOnly, HasTribe(Tribe.Murloc | Tribe.Beast))` (the `[Flags]` intersection folds the tribe OR into one test); a heterogeneous OR stays `Any(HasTribe(Tribe.Beast), MinionTypeIs("patient_zero"))`; *"any **other** friendly minion died"* = `All(FriendlyOnly, Not(SelfIsRelated))`.

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

- **Battlecry** resolves through the **card-resolution pipeline** (`ResolveCardAction` ④ — not the now-commit-only `PlayCardAction`, per #34; F11) and may pause for targeting via `PendingChoice`. Its effect actions are enqueued as ordinary, fully-windowed queue citizens (each keeps its ③′/⑥′ — intervenable, never swallowed). **Trigger timing (F19, simplified 2026-06-21):** the minion's persistent triggers go **live at its summon** — `OnSummon` registers at `ResolveCardAction` ④ (epoch N), the same ④ that places the body and returns `MinionSummonedEvent` — and the **Battlecry is enqueued after**, so it runs at later epochs *with the minion's triggers already live*. Consequences: the minion does **not** react to its own `MinionSummonedEvent` (equal-epoch creation filter, §3 `IEventBus`), but it **does** react to its own Battlecry's later-epoch events. This is a **deliberate divergence from Hearthstone**, which keeps a minion's triggered effects inactive until its Battlecry fully resolves (an HS minion never triggers off its own Battlecry — verified 2026-06-21); we trade HS-faithfulness on this one axis for engine uniformity (triggers live by board zone, no "summoned-but-dormant" sub-state — see "Unaddressed Features → A minion triggers off its own Battlecry"). **Other** minions' "after you summon" reactions still resolve **after** the Battlecry — because the Battlecry is enqueued *before* `MinionSummonedEvent` publishes, those reactions sit behind it in the queue (HS-faithful on this axis). Mechanism in §2A `ResolveCardAction` ④.
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

`Select` is a **pure function of `GameState`** (read via the context) — no mutation, no RNG, no enqueueing. Like the condition library it evaluates *relative to* the source, using the **same lane-based `Friendly`/`Enemy` predicates** (§3 `IEntityPredicate`): `AllFriendlyMinions` = minions sharing the source's `ownerId` (a neutral source's friendlies are the neutral lane); `AllEnemyMinions` = minions on an opposing side (`src.ownerId != m.ownerId && m.ownerId != null` — so a player source's enemy set excludes the neutral lane, which is the separate `AllNeutralMinions`; a neutral source's enemy set is both boards). `Self`, `FriendlyHero`, `EnemyHero` classify by the same rule (neutral has no hero).

**Singletons — parameterless** (referenced by reference, no per-call allocation): `AllEnemyMinions`, `AllFriendlyMinions`, `AllNeutralMinions`, `AllMinions`, `AliveEnemyMinions`, `AliveFriendlyMinions`, `AliveNeutralMinions`, `AliveMinions`, `DeadEnemyMinions`, `DeadFriendlyMinions`, `DeadNeutralMinions`, `DeadMinions`, `EnemyHero`, `FriendlyHero`, `AllEnemyCharacters`, `AllFriendlyCharacters`, `Self`.

**Factories — parameterized:** `AdjacentTo(id)`, `OtherThan(selector, id)`, `Union(s1, s2, …)`, `Filter(selector, IEntityPredicate)` — the general entity-property filter (§3 `IEntityPredicate`); e.g. *"all damaged friendly minions"* = `Filter(AllFriendlyMinions, IsDamaged)`, *"all non-Beast enemies"* = `Filter(AllEnemyMinions, Not(HasTribe(Tribe.Beast)))`. `MinionsWithTribe(Tribe)` / `MinionsWithKeyword(string)` remain as named sugar for `Filter(AllMinions, HasTribe(t))` / `Filter(AllMinions, HasKeyword(k))`.

**Control / command targeting needs no new selectors.** A "take control of any neutral or enemy minion" card declares `Union(AllNeutralMinions, AllEnemyMinions)`; a neutral-only or enemy-only variant uses the corresponding singleton; "command a neutral to attack another neutral" uses `OtherThan(AllNeutralMinions, attackerId)`, and "command into the opponent lane" uses `Union(AllEnemyMinions, EnemyHero)`. Per-lane Taunt is **not** encoded in these selectors — it is enforced uniformly by the §4 ③ validator on top of the card's declared set, exactly as for a normal `AttackAction`.

**`All…` / `Alive…` / `Dead…` (mortally-wounded inclusion).** A minion at `currentHealth ≤ 0` that is **still on the board** — not yet removed by death resolution — is **"mortally wounded"** (a.k.a. pending death). (This window is transient under any model but becomes a designer-meaningful state under the cascade-settle death model — Fireplace amendment B, §4 ⑦ — where such a minion can persist across same-cascade reactions.) The three on-board minion families partition exactly on this distinction — **`All… = Alive… ∪ Dead…`**, disjoint:

| Family | Returns | Use when |
|---|---|---|
| `AllEnemyMinions` / `AllFriendlyMinions` / `AllNeutralMinions` / `AllMinions` | **every on-board minion** of that side, **mortally-wounded included** | the effect should treat the doomed-but-present body as real — HS-faithful AoE and board-count effects ("deal 1 to all enemies", "damage equal to enemy minions") |
| `AliveEnemyMinions` / `AliveFriendlyMinions` / `AliveNeutralMinions` / `AliveMinions` | only minions with **`currentHealth > 0`** | the effect must **not** act on a corpse-in-waiting — e.g. "return a random alive enemy minion to hand", "buff your lowest-health alive minion" |
| `DeadEnemyMinions` / `DeadFriendlyMinions` / `DeadNeutralMinions` / `DeadMinions` | only the **mortally-wounded** (`currentHealth ≤ 0`, still on board, pending removal) | the effect targets the dying specifically — a **save-from-lethal** intervention or finisher, responding to lethal damage to rescue (or pick off) a dying minion (the "R1" avenue, amendment B), "destroy all mortally-wounded enemies", counting the doomed |

Each `Alive…` singleton is exactly `Filter(<corresponding All…>, Not(IsMortallyWounded))` (i.e. `currentHealth > 0`) and each `Dead…` is `Filter(<corresponding All…>, IsMortallyWounded)` (`currentHealth ≤ 0`), surfaced as named singletons for ergonomics. **The predicates are health-only, deliberately (2026-06-12):** a destroy-marked minion at positive health sits in `Alive…`, not `Dead…` — correct on both ends, since `Dead…` is the *save-targeting* family and marked minions are unsavable (§2A `DestroyMinionAction`), while for everything else (AoE, counts, retaliation) a marked minion behaves as the living board presence it still is. **Heroes have no mortally-wounded state** — a 0-health hero ends the game at §4 ⑧ rather than lingering — so there is no `Alive`/`Dead` hero/character variant. **Graveyard minions are *not* covered here:** an already-removed minion (resurrection / "died this turn" pools) is not a board target — selecting from the graveyard to *summon* is a separate primitive, not an `ITargetSelector`.

**Ordering.** The returned list is **ordered by the canonical board order** (Deterministic Ordering) shared with trigger fire order and death/deathrattle sort: current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index, with heroes slotted by the selector's own definition. Ordered, not unordered, because sequential within-action application (per-target damage, `DamageTakenEvent` emission order, keyword pulls) is order-sensitive and determinism demands a fixed order. (Deaths are no longer interleaved per hit — they settle at ⑦, amendment B — but the per-target application order still must be fixed.) Selectors never invent a second ordering.

**Three consumption modes** — a selector's candidate set covers every targeting feature with no new mechanism:

| Mode | How the effect consumes `Select(context)` |
|---|---|
| **Auto-hit** (AoE) | Enqueue **one** action carrying the **`ITargetSelector` reference** (the same one-action-carries-selector pattern as Random-K below); the stage-④ handler evaluates it over the current board and applies the effect to every result, in returned order — e.g. Flamestrike is a *single* `DealDamageAction` carrying `AllEnemyMinions`. Pure: no RNG. **One action ⇒ one ③′ declaration ⇒ one interception/pre-damage window** for the whole AoE (with the targets as that window's selector), rather than N per-entity actions raising N windows (see §3 "Reactive…", "Batching"). The handler still emits one per-target event (e.g. a `DamageTakenEvent` each). |
| **Random-K** | Enqueue **one** action carrying the **`ITargetSelector` reference** (not a pre-computed pool) + `k`; the **stage-④ handler evaluates the selector against the *current* board, then draws K** using that action's epoch-derived `IRandom`. Carrying the selector rather than a frozen `EntityId[]` keeps selection both pure *and* current — if a candidate is mortally wounded or removed between enqueue and the draw, the ④ evaluation reflects it (and the card chooses the right `All…/Alive…/Dead…` family to decide whether the dying still count). Randomness is consumed only at ④, preserving the Item-6 invariant verbatim and keeping each draw attributable to one action's epoch (clean for the `StabilizationAbortReport`). |
| **Player-choice** (Discover, mid-effect targeting) | Issue `StartChoiceAction` with `options =` the candidate set → existing `PendingChoice` (§4). The selector *feeds* the choice; it does not subsume it — "compute targets" and "interrupt for player input" stay separate concerns. |

**Player-target legality is unified into the same primitive.** A card/effect requiring a player-supplied target declares a `(selector, cardinality)` pair. The §4 ③ validator's *"is a legal target type for this card"* check is then exactly: **`chosen ∈ selector.Select(context)`** (plus the cardinality/optionality rule) → `InvalidTarget` otherwise. The *same* `AllEnemyMinions` selector that an AoE uses to hit every enemy is what a single-target spell uses to define its legal targets — so previewed legality (client highlighting), validation, and effect resolution can never disagree. This replaces the spec's earlier hand-wave with one source of truth (parallel to how §4 ③ made the validator the single source of truth for action legality).

**JSON encoding** of selector references in card definitions is a `DefaultCardHandler` detail, *not* part of this spec — same treatment as the condition tree.

### Reactive Triggers, Interventions & Interception

This subsection unifies the **sigil** family (reactive cards — both deployments, invoked and inscribed; see "Sigils" below), the locked Intervention system, and the dying-window — the **save-from-lethal** beat, where a player responds to lethal damage to rescue a minion before it dies (the "R1" avenue in amendment B) — into **one** mechanism over `ITrigger`. Decided 2026-06-04 (Fireplace menu point D — the secrets/armed-reactive question); the armed flavour landed 2026-06-11 as the sigil model. Three structural ideas — **zone-scoped hosting**, **two hook phases**, and **interception as ordinary effects + re-resolution** — under one **delivery model** (who responds, and how the window is shown), reworked 2026-06-16.

**The responder & window delivery (2026-06-16 — supersedes the contingent-window and dual-responder model; resolves review findings #16/#17).**

**Who responds.** Interventions — both **invocations** (hand sigils) and **inscriptions** (face-down sigils) — are live **only while their owner is not the active player**. The active player never intervenes on their own turn, and **no one** intervenes while `activePlayerId == null` (the Interturn). The responder is therefore single-valued — always the non-active player. This (a) makes **depth-1 automatic** (the active player can't counter a response on their own turn), and (b) **dissolves source-based actor attribution** — own-turn draw/fatigue can't spring the active player's own reactive cards because those cards are simply off (#17). It is **HS-faithful**: Hearthstone Secrets only trigger on the opponent's turn. **Permanents are unaffected** — board, artifact, deathrattle, and aura triggers are *not* interventions and fire at ⑤ regardless of turn. **Corollary (D1):** since the active player has **no** ③′/⑥′ window on their own turn, **self-inflicted lethal on your own turn is unsaveable by reactive cards** — own-board AoE clipping your own hero, self-damage, or fatigue crosses to lethal with no window to answer it. By design (it falls out of the single-valued non-active responder) and HS-consistent (Ice Block is a Secret = off-turn); forward planning and non-reactive effects are the only mitigations.

**How the window is delivered — unconditional + two-stage.** At every action the active player's cascade declares (③′) and after every such action's ⑤/⑥ (⑥′), a **decision window** opens for the non-active player **unconditionally — even with no matching card** (reversing the former contingent model), so window *existence* never tells the active player whether a reactive card is held. Two stages:
- **Decision** (`InterventionDecisionWindow`; GamePhase `InterventionDecision`): **skip** / **skip-all** / **intervene** via `RespondInterventionAction`. `intervene` is offered only with ≥1 matching + affordable card; `skip-all` suppresses the rest of *this action's* windows (its ③′ **and** ⑥′); no response = skip (timeout).
- **Extension** (`InterventionExtensionWindow`; GamePhase `InterventionExtension`): entered by pressing intervene; pick one card + targets via `SubmitInterventionAction`, or back out.

**The press is the rationed resource.** A non-active player may press **intervene** at most `MaxInterventionsPerTurn` times per active-player turn (`TurnState.interventionsUsed`, reset at turn advance) — counted whether or not a card is then played, uniformly at ③′ and ⑥′ (each re-declare / re-offer play is a fresh press), and **on top of** the off-turn mana pool. Inscriptions, firing in-line at ⑤, never open a window and never consume it. **Once the budget is spent the decision window stops opening** for the rest of the turn — the cascade just proceeds. This leaks nothing: each escalation already fired the public `InterventionWindowOpenedEvent { stage: Extension }`, so the active player has counted the presses and the budget is publicly known.

**Pacing.** The active player's turn clock is **paused** while a window is open. A client may set a standing **`autoSkipAll`** flag (`PlayerState.autoSkipAll`), read at each window-open: when set, the window is elided (auto skip-all, no wait, no round-trip), and flipping it true also resolves a currently-open decision window; replay records flag *transitions*, not per-window skips. **Pinned/deferred:** a per-turn real-time budget on the base (decision) windows — the slow-play vector the press-cap does not cover — is **not** built in v1 (Unaddressed Features).

**1 — A trigger is live by virtue of its host's zone (aura-like).** An `ITrigger` is registered by `ICardHandler` when its host enters a zone and unregistered when it leaves — exactly as a board minion's triggers register in `OnSummon` and drop on death. The same lifecycle simply runs on **other zones**:

| Host zone | Registered on | What a fired trigger does | Status |
|---|---|---|---|
| **board** (minion) | `OnSummon` (entering board) | ordinary reaction | existing |
| **hand** (sigil) | the `DrawCard`/`GiveCard` handler (entering hand), dropped when the card **leaves the hand by any route** (#25 — play / discard / return / shuffle-into-deck / any future hand-exit such as transform-in-hand; a sigil shuffled into the deck does **not** keep a live hand trigger) | when its owner is **non-active**, the matched card is offered in the owner's **two-stage decision/extension** window (see "The responder & window delivery"); playing one is an **invocation** | **this amendment** |
| **inscriptions** (sigil) | the `PlayCardAction` inscribe branch (entering the zone), dropped on reveal | **auto-resolves** without a prompt (an **inscription**) **while its owner is non-active** (off-turn only — HS-faithful); player choices in the response degrade to **random** (see "Sigils") | **resolved 2026-06-11; off-turn gating 2026-06-16** |
| **artifact row** (`ArtifactOnBoard`) | the `EquipArtifactAction` handler (entering the row), dropped on destroy/discard | **auto-resolves** (board-side zone — a passive artifact's triggers, and any `IAura` it hosts, behave exactly like a minion's; no player window). A trigger-fire consumes `durability` iff the definition declares trigger-fire a consumer | **added 2026-06-11** (artifacts) |

All of this obeys the existing invariants unchanged: registration happens inside an action handler (④), so every reactive trigger has a `birthEpoch` and the creation-epoch filter (§3 `IEventBus`) stops a freshly-drawn card from reacting to the very event that drew it; the bus still carries only `ITrigger`. The hand is "armed" simply by holding the card — there is no separate arming step, and **what opens a window is precise and inspectable**: a card in hand whose live reactive trigger matched. This realises the locked Intervention scope ("play one card / skip") literally — the window only ever offers a card *designed* with a reactive trigger.

**2 — Two hook phases; that is the whole taxonomy.** Every action has a **pre** phase and **post** events:

- **Pre — the declaration phase (`ActionDeclaredEvent`, stage ③′).** A trigger here fires *before* the action resolves and may **intercept** it (prevent / alter). This is the held-action / suspend-resume kind. It fires **immediately, per action, mid-cascade** — there is no other coherent timing, since interception must precede that action's ④. A response does not end the matter: if it **plays** a card, the held action re-validates (②③) and — if still legal — **re-declares (③′)**, re-opening the window over the responder's *remaining* matching + affordable cards; the loop runs until the responder **skips** (the held action proceeds to ④) or holds nothing affordable, or the action **fizzles** (the **re-declare loop**, 2026-06-09). Multiple held interventions on one declaration are thus the responder chaining their *own* cards, in an order they set by which they pick each pass — **not** a nested stack (depth-1 still bars a further *player window* on a response; an auto host — inscription or artifact — may still fire — and even intercept — inline; see §4 ③′).
- **Post — the regular events.** A trigger here reacts *after* the fact and cannot prevent — it just acts. This **includes `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`**: a "save" reaction on those is an ordinary post-reaction whose *before-finalization* timing is a gift of the cadence deferral (⑦ death / ⑧ win are settle stages — §4), **not** special machinery. "Consequence-deferral" is therefore not a third category; it is a post-reaction that exploits the deferral. **Post-reaction player windows are set-valued and re-offer on play (stage ⑥′)**: after a *single* action's ⑤/⑥, the engine opens **one** window presenting *all* the responder's matching + affordable hand cards; the player plays **one** (choosing it and its targets) or **skips**. A played card's targets come from its *own* matched-subject set, so an AoE (one selector-carrying action hitting N minions) offers that one card over all N subjects — **one** window, not N. Each **play** re-offers the *remaining* matching set against the same (frozen) events (the **re-offer loop**, 2026-06-09); a **skip** ends the window (only a play changes the offer). Two *separate* damage actions remain two distinct instances → two windows. The triggering unit is one action's events — **never per-event, never per-cascade** (see §4 "Batching"). Deterministic, no-choice reactions belong on auto hosts (an inscription, or a board/artifact trigger) and open no window at all.

**3 — Interception is ordinary effects + re-resolution — there is no "disposition vocabulary."** An interception response runs as **normal effects** (summon, grant a keyword, gain armor, return-to-hand, deal damage — anything a card can do). The held action is then **re-resolved through full validation (②③)**, so almost every "alteration" emerges for free:

- grant Divine Shield to the defender → on resume the shield (a keyword pull at the damage moment) absorbs the hit;
- grant Poisonous to the defender → its resumed counter-damage destroys the attacker;
- gain armor / a standing damage modifier → read at the damage moment, no action edit (`PlayerState.armor`, §1; consumed at the ④ hero absorb slot);
- destroy / return / silence the attacker, or summon a **Taunt** token → on resume the held action **re-validates and either fizzles or is forced to the Taunt** by the ordinary rules.

Only **two** manipulations cannot be expressed as board-state mutation, and they are the *entire* irreducible surface — not a vocabulary:

- **cancel** — drop the held action (Counterspell: nothing on the board invalidates a legally-cast spell);
- **retarget** — substitute the held action's target *in place*, without re-declaring it (Misdirection's random redirect, Spellbender's spell redirect — not derivable from Taunt).

**Completeness rests on one invariant:** *anything a card can intercept is a discrete action* (so it has a declaration phase). Intrinsic, deterministic modifiers (the Divine Shield keyword, armor) are **pulled inline at resolution**, never declared — consistent with the `IKeyword` pull model (§3 `IKeyword`). So one interception point has **two realisations**: intrinsic *pulls* (hot-path, no window) and extrinsic *subscriptions* (sigils — inscribed or held, possibly windowed).

**This subsumes damage prevention (menu point A — now closed).** Ice-Block-style "prevent the fatal hit, no side-effects" is an **interception on `DealDamageAction`** (`cancel`, or grant the target immunity, at its declaration) — *not* a `HeroMortallyWoundedEvent` post-reaction, which would let the damage (and its lifesteal/reflect side-effects) register before undoing the death. Three things make this universal and deterministic:

- **All damage flows through `DealDamageAction`, combat included.** `AttackAction` *enqueues* `DealDamageAction`s for the two combatants rather than applying damage inline, so the predamage declaration covers **every** source — Ice Block / redirect / prevention stops combat damage too, not just spells. **Combat is two independent damage actions, not one atomic unit (decided 2026-06-06).** The attacker's strike and the defender's retaliation are each first-class actions — each with its own ③′ declaration and its own ⑥′ window — processed **strictly sequentially** through the queue. This is the §4 ⑥′ rule applied verbatim ("one window per action; two instances = two windows"); an atomic single-window combat would be an *exception* to that rule, so it was rejected. Consequences: **(a)** interception/reaction windows are **per recipient** — the natural granularity, since Ice Block / Divine Shield protect one entity, not "a combat" — and a held card may interleave **between** the two hits (e.g. bounce the defender to dodge its retaliation); **(b)** a same-combat reaction sees **per-hit** board state (the defender's "when damaged" reaction fires *before* the retaliation lands) — a deliberate, narrow divergence from HS's apply-both-then-react; **(c)** **base attack is snapshotted at `AttackAction` ④** (HS-style attack lock), while damage *modifiers* are pulled at each damage action's ④ (precedence below). Because death is deferred to ⑦, **both combatants are mortally-wounded-but-on-board until the queue drains, so they die in one shared wave**, and a lethal-but-not-removed defender **still deals its retaliation with full effect** (keywords + source-side modifiers pulled live). This makes **dying-swing retaliation** — Poisonous/Lifesteal on a dying defender, or "retaliation doubled while mortally wounded" — a supported mechanic *by construction*, not special machinery.
- **AoE damage is one declaration.** An auto-hit AoE is a *single* selector-carrying `DealDamageAction` (§3 `ITargetSelector`), so it raises **one** ③′ window with the targets as its selector — the pre-damage twin of post-reaction batching (one window for 7 minions), differing only in that it fires **immediately, before the hits** (pre-damage cannot defer to the settle). Per-target protection inside it is a grant (immune / Divine Shield) read at ④, not a new op.
- **④ damage-modifier precedence (pinned).** After ③′ interception, the handler computes each target's effective amount in a fixed order: **(1)** base `amount` (spell-damage bonus already baked at enqueue, §3 `IEffect`); **(2)** multiplicative modifiers (double-damage); **(3)** flat reductions, floored at 0; **(4)** caps; **(5)** **immune** → 0, stop (no shield break, no armor loss, no health loss); **(6)** the **absorb slot, by target type** — *minion*: **Divine Shield** → if still > 0, break it (`DivineShieldBrokenEvent`) and set 0; *hero*: **armor** → `absorbed = min(armor, amount)`, `armor −= absorbed`, `amount −= absorbed` (reported as `DamageTakenEvent.armorAbsorbed`, §2B — partial absorption lets the remainder through, unlike the all-or-nothing Shield); **(7)** apply the remainder to `currentHealth`/`health`, compute overkill, emit `DamageTakenEvent`, and if `≤ 0` emit `MinionMortallyWoundedEvent` (hero → `HeroMortallyWoundedEvent`). Only the baked spell-damage bonus (1) and the absorb slot / events (6–7) exist in v1; steps 2–4 are the slots future modifiers drop into, in this order, so stacking stays deterministic and replayable.

**Non-interceptable actions — `Inexorable` / `Uncounterable` (DR1, 2026-06-28).** Two play-phase keywords let a card opt its play out of the ③′ interception checkpoint, so trivial cards don't tax every cascade with a window nobody would answer (the LoR *burst* lesson — windows only where a response is meaningful). They are **declared on the card definition and read at `PlayCardAction`** — play-phase keywords in the Battlecry/Combo family, *not* on-board effective keywords: they never enter the `MinionOnBoard` keyword pull and are inert once the card has resolved. Both treat **`{PlayCardAction, ResolveCardAction}` as one play-unit** that skips ③′, so the card can be neither *prevented* (the `PlayCardAction` ③′) nor *countered* (the `ResolveCardAction` ③′). They differ only in reach over the actions `ResolveCardAction` *enqueues* — a spell's `DealDamageAction`, a minion's Battlecry effects (which the F19 flow enqueues from the resolution, so they sit on the own-effect spine):

- **`Uncounterable`** — the play-unit only. Enqueued effects keep their ③′, so the card resolves uncounterably but its damage / Battlecry stays interceptable (Ice-Block / redirect still work).
- **`Inexorable`** — the play-unit *and* its whole own-effect spine. The suppression propagates to every action `ResolveCardAction` (and they in turn) enqueue, so neither the card nor its effects can be intercepted.

Mechanism: a `suppressDeclaration` flag is stamped on the play-unit and threaded across the commit→resolve link **unconditionally** (they are one play), then onward to enqueued effects **only under `Inexorable`** (a `propagate` bit) — exactly the `sourceId` enqueue-thread of "Source Attribution" above, except that `sourceId` *never* inherits while this flag deliberately may. A **trigger** reacting to one of the resulting events is a fresh origin: it stamps its own `sourceId` and does **not** inherit the suppression, so downstream consequences re-enter ③′ normally (**option A — "you can't stop it," not "you can't answer its aftermath"**). The **post side is untouched**: ⑤ events still publish (announcements, permanents, inscription autos, wire/renderer/trace) and ⑥/⑥′ post-reaction windows still open — only the pre-execution declaration is elided. The absence of a window leaks nothing: the keyword is a public property of the played card (revealed at play), so it never tells the active player anything about the *responder's* hidden hand.

**Visibility — the responder's side (pinned 2026-06-10; queued topic #4).** One **information ceiling**: a window prompt never widens the responder's visibility — it shows exactly **their own candidates** (their hand, plus engine-computed legal targets per card — §1 `InterventionCandidate`) **+ the cause**, where the cause is either public-by-nature or already on their wire:

- **③′:** the **held action verbatim** — its full §2A params (`PlayCardAction: Fireball → your m3`). Prior art is **unanimous** (MTG stack objects, YGO chain links, LoR stack-with-targets, FaB attacks): every game with a responder window shows the pending object in full — informed response *is* the point of a player-choice window. Safe by construction: declared params are pre-resolution facts (randomness/draws resolve at ④), and casting = the moment of public reveal. One carve-out: a `PlayCardAction` that **inscribes a sigil** declares **redacted** — "inscribes a sigil" + the mana spent, never the card identity ("Sigils" below). This is the survey's one *sanctioned* hidden-identity case: face-down pre-armed cards protect **their owner's** card, never the actor's pending action.
- **⑥′:** **refs to the matched events** (`causeEventIds[]` — they were published at ⑤ and are already on the responder's wire before the window opens); the prompt re-embeds nothing, so it can never out-reveal the wire (a trigger matching the opponent's `CardDrawnEvent` does not expose the drawn card).
- **Windows are unconditional (structural) and two-stage** (2026-06-16 — reverses the former contingent model). A Stage-1 decision window opens at **every** ③′/⑥′ regardless of whether a match is held, so its existence is uninformative; the **escalation** to the Stage-2 extension window is the explicit, accepted tell — `InterventionWindowOpenedEvent { stage: Extension }` tells the acting player a matching card exists (never which), and pressing intervene then backing out is a self-inflicted reveal. The decision/extension durations are `GameConstants` (rank-configurable), the responder runs only off-turn, and the press budget (`MaxInterventionsPerTurn`) caps escalations — full model in "The responder & window delivery" above. This deliberately adopts LoR-style structural windows over the old Arena/Master-Duel contingent tell, with the `autoSkipAll` toggle and turn-clock pause as the fluidity levers.

### Sigils — one reactive card, two deployments (2026-06-11)

A **sigil** is a card whose definition carries a reactive `ITrigger` (v1 content: spell-typed; the hosting seam is type-agnostic, so ambush-style sigil minions remain possible). There is **no secret card kind**: every sigil supports **both** deployments, and the host zone — already the behavioral discriminator in the hosting table above — *is* the mode selector:

- **Invocation** — hold the sigil in hand; when its trigger matches inside a window, the owner may play it via `SubmitInterventionAction` (full choice: see the cause, pick targets, or decline). Pays from the carried off-turn pool at response time.
- **Inscription** — pre-commit it face-down via `PlayCardAction` on one's own turn (pay-on-inscribe; the trigger fires free later). It **auto-resolves on the first matching event** — no prompt, no decline.

The two modes differ on exactly three axes, all zone-derived: **choice** (window has it; inscription doesn't), **depth** (player windows are depth-1-capped; inscriptions fire inline even on a response's sub-actions), and **economy** (invoke pays later from the carried pool and costs a hand slot; inscribe pays now from the on-turn pool, frees the hand, and shows a face-down tell).

**Rules of the inscribed mode:**

- **Choice degradation → random.** An inscribed sigil's response resolves every player decision at random via `IRandom` (counter-reseeded — deterministic, replay-safe): a declared `(selector, cardinality)` target requirement consumes as **random-K drawn at ④** (the existing `ITargetSelector` consumption mode) instead of player-choice; modal choices pick uniformly. Same selector, same cardinality — the mode flips, the card doesn't. Every sigil is therefore inscribable, even ones players would rarely inscribe.
- **Off-turn firing rule (2026-06-16 — replaces the source-based actor-attribution rule):** an inscription is live **only while its owner is the non-active player**, firing in-line at ⑤ on a matching event during the opponent's turn (and never while `activePlayerId == null`). This is **HS-exact** — Hearthstone Secrets trigger only on the opponent's turn — and it **dissolves the sourceless-attribution problem** (#17): own-turn draw/fatigue can't spring the owner's own inscription because the inscription is simply off, with no `sourceId` reasoning needed. It supersedes the former "never on your own action" attribution and its "punish an opponent's invocation during my turn" corollary (both removed). A player's own deathrattles resolving off-turn likewise never spring their own inscriptions — now for the simpler reason that it isn't their turn.
- **Constraints:** cap `GameConstants.MaxInscriptions = 3`; **one inscription per `definitionKey`** (`SigilAlreadyInscribed`); full row rejects at ③ (`InscriptionSlotsFull`). An inscribed copy and a held copy of the same sigil are **independent** — the inscription auto-fires in dispatch order, and the held copy is still offered in the subsequent window (each mode follows its own rule).
- **One-shot:** firing consumes the inscription — reveal (`SigilRevealedEvent`), resolve, then `inscriptions` → graveyard. Persistent standing triggers are what **passive artifacts** are for; the two stay distinct.
- **Empty-pool guard:** an inscription does **not** fire if a required selector pool is empty at fire time — it stays inscribed, unconsumed (HS-faithful "won't trigger if it would do nothing"; fire-and-fizzle-and-consume is a pure feel-bad). **Scope of the guarantee (#26):** the check is at **fire time**, which is not the response's own ④. It prevents only *foreseeably*-empty fires; a pool that is non-empty at fire time but empties between the reveal and the enqueued response's ④ (same cascade) can still fizzle the already-revealed-and-consumed inscription. This residue is narrow (it needs an intervening same-cascade action to drain the pool) and accepted — closing it would require deferring consumption to resolution-time, a larger change with its own ordering hazards.
- **Stealth, across the two modes (#32):** the same sigil treats Stealth differently by mode, and this is intended. **Invoked**, its player-chosen target is gated by `TargetStealthed` at ③ (you cannot hand-pick a stealthed enemy). **Inscribed**, the degraded **random-K at ④** draws over the raw selector pool, which *includes* stealthed minions (selectors don't model Stealth) — so an inscribed sigil can hit a stealthed minion that its invoked twin could not. HS-consistent (random effects ignore Stealth); the "the mode flips, the card doesn't" line above is about the selector/cardinality, not this Stealth-reachability difference.
- **Dispatch slot:** inscription-hosted triggers fire **inside their owner's dispatch group, after that owner's artifacts**, in inscription order (`inscriptions[0..2]`) — the artifact precedent extended one slot (§3 `IEventBus`). Auto-chains (one inscription's response springing another of the **same owner's** — only the non-active player's inscriptions are live on any given turn) need no depth rule: each fire consumes its card and the rows cap at 3, so chains terminate structurally.
- **Visibility:** existence public, identity hidden — inscribing emits the owner-directed `SigilInscribedEvent` + public thin `InscriptionAddedEvent` *instead of* `CardPlayedEvent` (§2B); full public reveal only at fire time. The armed-state telegraph question is thereby resolved: the zone shows N face-down cards, HS-style (full hiding is unachievable anyway — the mana delta is public). Hand-held sigils need no telegraph rule: they are ordinary hidden hand cards; the only tell is the explicit **escalation** event when their owner presses intervene (`InterventionWindowOpenedEvent { stage: Extension }`, §2B), identical to any invocation.

*(Engine identifiers keep the `Intervention*` names for the window machinery — `PendingIntervention`, `SubmitInterventionAction`, `InterventionPromptEvent`; sigil / invoke / inscribe is the game vocabulary for the card family, the acts, and the zone, layered on those primitives.)*

**Nothing remains deferred in this subsection.** *(History: the secret bundle — auto flavour, zone, constraints, telegraph, cost timing, identity — resolved 2026-06-11 as the sigil model above; the **marked-for-destruction scope** resolved 2026-06-12, option A — fiat dooms are unsavable and silent, preventable only at the destroy action's ③′ declaration (§2A `DestroyMinionAction`, §2B `MinionMortallyWoundedEvent`); batching: §4 "Batching".)* See the borrow-list note.

### `IAura`

Continuous effects recalculated after every board-affecting action. Aura bonuses are **never** stored in `enchantments` — they live in `auraAttackBonus`/`auraHealthBonus` and are fully rewritten on every recalculation pass. An aura is registered when its host **enters its zone** — a minion at summon (`OnSummon`, ④), a passive artifact at equip (`EquipArtifactAction` ④) — and dropped when the host leaves; `SourceEntityId` names that host (minion or artifact).

```csharp
interface IAura {
    string AuraId { get; }
    string SourceEntityId { get; }   // the hosting entity — a minion OR a passive artifact (§1 ArtifactOnBoard). Renamed from SourceMinionId (2026-06-13, #9): artifacts host auras now, and they implement this literal contract.
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

**Aura scope & the neutral lane.** An aura's reach is entirely its `Calculate` — there is **no hardcoded neutral wall**; scope is the same lane-based `Friendly`/`Enemy` + selector semantics used everywhere (§3 `IEntityPredicate`/`ITargetSelector`). Consequences, all derived (no new machinery — recalc already zeroes/rewrites the `aura*` fields on **every** minion including the neutral lane, §4 ⑥): a **player's** friendly-scoped aura ("your other minions +1/+1") excludes the neutral lane (a player's `friendly` is its own board); a **neutral** minion's friendly-scoped aura buffs its **lane-mates** (its `friendly` is the neutral lane); reaching the lane from a player aura requires an explicit `AllNeutralMinions`/`AllMinions` selector in `Calculate`; **adjacency** auras (`AdjacentTo`) are per-lane, since each side is its own ordered array with no cross-lane adjacency.

**Self-conditional ("while …") auras.** A `Calculate` may read its own host (`SourceEntityId`) and return a **self-targeting** `AuraEffect` gated on the host's state — the general form of a "while <condition>" self-buff. *Example:* "while this minion is damaged, +3 Attack" returns `[AuraEffect(host, +3, 0)]` when `host.currentHealth < host.maxHealth`, else `[]`. The bonus rides the ordinary `auraAttackBonus`/`auraHealthBonus`/`auraKeywords` channel — recomputed each ⑥, never persisted, Silence-safe — so it needs **no dedicated storage or toggle event**. **This is what replaced the former `Enrage` keyword** (and `EnrageStateChangedEvent`), removed 2026-06-22 (spec-review-3 G6): Enrage was just the "while-damaged → +Attack" instance, and authoring it as a self-conditional aura unlocks the whole family ("while at full health", "while you hold a Dragon", "while damaged → has Windfury" via `GrantedKeywords`, …) with no engine keyword.

**Convergence caveat (single-pass recalc).** Recalc is **one pass, not a fixpoint iteration**. A self-conditional aura is well-defined as long as its **output does not feed its own predicate**: "while-damaged → +Attack" is stable because the predicate is on *health* and the output on *attack* (and the damage gap `maxHealth − currentHealth` is itself invariant under health auras, per the #13 rule, so the predicate is insensitive to other auras' read/write order). A self-**referential** aura whose output drives a predicate on the *same* stat — e.g. "while Attack ≤ 3, +2 Attack" — has **no stable value** (3→5→3→…) and would flip each recompute; such cards are **undefined — do not author them** (inherent ill-definition, not an engine bug to resolve). Cross-referential aura *pairs* (one reads what another writes) are deterministic per pass; their cross-pass behaviour is the general `IAura` ordering matter, independent of this note.

### `IWinCondition`

The **alternate-win seam** consulted by §4 ⑧. Same registration shape as `IAura` — an entry is registered when its host **enters its zone** (a quest artifact at equip, a minion at summon) and dropped when the host leaves; it is **pulled, not pushed** — ⑧ re-runs every registered `Evaluate` against the freshly-settled state, never a flag set mid-cascade (see ⑧ for why pull-at-settle is the form that composes with dying-window saves and the mutual-lethal draw rule).

```csharp
interface IWinCondition {
    string ConditionId { get; }
    string SourceEntityId { get; }          // the hosting entity (minion or artifact), as IAura
    EntityId? Evaluate(GameState state);    // the player who wins by THIS condition, or null
}
```

The hero-death check is **not** an `IWinCondition` — it is ⑧'s hardwired built-in (always evaluated, no host). **v1 ships zero registered `IWinCondition`s** (deck-out/fatigue already route through the built-in hero check as damage); the interface and ⑧'s list-iteration exist so alternate-win content — quest-reward-as-win, mill-as-win, alternate-resource thresholds — is **v2 content on a fixed seam**, never settle-loop surgery. A registered condition that fires publishes the ordinary `GameEndedEvent`; a future content reason (e.g. `QuestComplete`) joins the §2B `reason` set when the first such card is authored.

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
  - **`PendingChoice` timeout policy (#24):** a timed-out Discover / mid-effect targeting choice is resolved by the Game Server injecting a `SubmitChoiceAction` for a **random legal option** — the **server's** roll, not the engine's `IRandom` stream (the engine stays a pure function of its logged inputs; the injected action *is* the logged input, so replay is exact regardless). Random (not "first option") is chosen for fairness — a roped Discover should not be a predictable always-leftmost pick. **Mulligan is the exception: a timed-out mulligan keeps all cards** (no replacements) — HS-faithful and the safe default. (The choice of *what* to inject is Game Server policy; GameLogic only guarantees the injected action replays deterministically.)
- **Single, total-order input stream** — one sequence per game, not per-player. The engine processes inputs in a single total order; per-player streams would reintroduce interleaving ambiguity.
- **Ruleset-version pinning** — a command-log replay re-runs engine logic + card `definition` JSON, so it is only valid against the **same engine + card-definition version** it was recorded under (the classic command-log-replay fragility across patches). The replay package carries a ruleset/content version stamp.
- **Command log vs. event log — two artifacts, two jobs.** The **command/input log is canonical**: compact, re-simulatable, the source of truth (but version-fragile per above). The **event log is the client wire format**, spectator/late-join stream, and version-robust archive: self-contained state deltas the client renders with **no game logic**, but *not* re-simulatable. (The event log is exactly the **handler-produced deltas**; transient bus signals — `ActionDeclaredEvent` — are excluded, as they carry no state change. See §2B "Bus ⊋ log".) The asymmetry: the **event log is derivable from the command log + seed, never the reverse** (events record outcomes, not the inputs/rolls that produced them) — so the command log is the richer artifact and the event log the safer/self-contained one.

### Debug Trace (`ITraceSink`)

A **third, purely diagnostic stream** alongside the two §3 `IRandom`-Replay artifacts (command log = canonical/re-simulatable; event log = client wire format): the debug trace has **no replay, persistence, or wire role**, is server-side only, and its shape may change freely between versions. Primary use: when a test fails, read the trace to step through what the engine actually did, action by action. Emitted only when a sink is attached (tests, local repro) — **zero overhead otherwise**.

```csharp
interface ITraceSink { void Record(TraceRecord r); }   // engine ctor takes ITraceSink? — null = no tracing
```

**Emission model.** The pipeline calls the sink from *inside* stages — necessarily so: a stage-③ rejection produces zero events by invariant (§2C), so no post-processor over the event stream could ever see it. **Diffs are computed by structural compare** of `GameState` before/after each action — the clone cost is paid only when a sink is attached, and compare-based diffing guarantees **no mutation can escape the trace unseen** (the property that matters when hunting a bug; an instrumented-setter approach can only miss writes). A `JsonLinesTraceSink(Stream)` ships as the default implementation: one compact JSON object per line (System.Text.Json — the stack the data model already commits to via `JsonElement`), written **append-as-it-happens** — a crash or hang mid-cascade leaves a valid, readable trace up to the failure point (precisely when the trace is wanted most). Pretty-printers and the eventual **visualizer are downstream consumers, out of library scope** (same ruling as the animation layer and bot scaffolding); the schema below is what makes them possible.

**Record vocabulary** (JSON discriminator `k`; one record per *thing that happened*, in engine order — quiet stages emit nothing, per the stages-when-they-act principle):

| `k` | When | Carries |
|---|---|---|
| `state` | Match start + each time a cascade fully settles to idle | Full `GameState` snapshot incl. ordered deck contents, plus `v` (trace format version), `sessionId`, `rngSeed`, `epoch`. **Elides `definition` bodies** — cards/minions carry `definitionKey` only (definitions are static content, reproducible from the library — the Fabrication-rule philosophy applied to the trace) |
| `action` | One processed action (per ④; carries its `epoch`) | The action's own §2A params, then **only-if-nonempty**: `events` (⑤ payloads inline), `diff`, `enqueued` (each entry tagged `src` = enqueuing entity, so deathrattle-vs-trigger attribution is legible). Aura recalc (⑥) needs no marker — it surfaces as `aura*`/`attack`/`maxHealth` diff entries |
| `reject` | ②/③ refusal | The submitted action + error code. Zero events/diff by invariant — the record proves the action was submitted and refused |
| `fizzle` | A held action **cancelled** at ③′ or **illegal on resume** (re-validation fails after a response) | The fizzled action + `reason` (`Canceled` \| `Illegal(code)`) + any `events` it emitted (a card play emits `CardPlayFizzledEvent`; a cancelled `DealDamageAction`/attack is eventless). Distinct from `reject` — that is the zero-event pre-④ refusal of a freshly *submitted* action; a fizzle is a *resumed* action that dropped (2026-06-13, #8/#34) |
| `window` | An intervention window opens (③′ or ⑥′) | `at`, `stage` (`Decision` \| `Extension`), responder, candidates, held action if any. The *resolution* is **not** in this record — the later `RespondInterventionAction`/`SubmitInterventionAction` trace as their own `action` records (append-only, real-time order; the re-declare/re-offer loop traces naturally as alternating `window`/`action` records) |
| `choice` | A `PendingChoice` opens | Same logic; `SubmitChoiceAction` resolves it as its own `action` record |
| `settle` | Per death wave (⑦); ⑧ **only when it acts** (a hero ≤ 0) | ⑦: `wave`, `deaths[]`, events, diff (zone moves), enqueued deathrattles/reborns. ⑧: the win/draw/save outcome |

**Worked examples.** All three were audited across the three streams in-session (bus = log + declarations; log = pure state deltas; wire = log filtered by directedness) and replay-checked from both the command log and the event log.

**(1) Combat → strike → retaliation → death → deathrattle.** `m3` (3 health) attacks `m7` (4 health); `m3` dies to the retaliation (overkill 1) and its deathrattle enqueues a real `DealDamageAction` tagged `src: m3` (no `DeathrattleAction` type exists — a deathrattle *is* its effect's ordinary actions, plus `DeathrattleTriggeredEvent` in the settle record):

```json
{"k":"action","epoch":42,"action":{"type":"AttackAction","attacker":"m3","target":"m7"},"events":[{"type":"AttackPerformedEvent","attacker":"m3","target":"m7"}],"diff":{"m3.attacksUsedThisTurn":[0,1]},"enqueued":[{"type":"DealDamageAction","src":"m3","tgt":"m7","amount":2},{"type":"DealDamageAction","src":"m7","tgt":"m3","amount":4}]}
{"k":"action","epoch":43,"action":{"type":"DealDamageAction","src":"m3","tgt":"m7","amount":2},"events":[{"type":"DamageTakenEvent","tgt":"m7","amount":2,"armorAbsorbed":0,"overkill":0,"src":"m3"}],"diff":{"m7.currentHealth":[4,2]}}
{"k":"action","epoch":44,"action":{"type":"DealDamageAction","src":"m7","tgt":"m3","amount":4},"events":[{"type":"DamageTakenEvent","tgt":"m3","amount":4,"armorAbsorbed":0,"overkill":1,"src":"m7"},{"type":"MinionMortallyWoundedEvent","minion":"m3"}],"diff":{"m3.currentHealth":[3,-1]}}
{"k":"settle","wave":0,"deaths":["m3"],"events":[{"type":"MinionDiedEvent","minion":"m3"},{"type":"DeathrattleTriggeredEvent","minion":"m3"}],"diff":{"m3":["p1.board[0]","p1.graveyard"]},"enqueued":[{"type":"DealDamageAction","src":"m3","tgt":"p2","amount":1}]}
```

**(2) Fireball countered (the #34 commit/resolve split).** `PlayCardAction` commits at ④ (mana paid, card → graveyard, `CardPlayedEvent` — **no** `SpellCastEvent`) and enqueues `ResolveCardAction`; `p2`'s Counterspell intercepts the **resolution**'s ③′; the resolution **fizzles** with no refund (mana stays 0, Fireball stays in the graveyard). Because `SpellCastEvent` would only have been emitted at the resolution's ④ — which never runs — **no cast-synergy ("whenever you cast a spell") triggers off the countered Fireball** (D2):

```json
{"k":"action","epoch":50,"action":{"type":"PlayCardAction","player":"p1","card":"c12","target":"m7"},"events":[{"type":"ManaChangedEvent","player":"p1","delta":-4},{"type":"CardPlayedEvent","player":"p1","card":"c12","target":"m7"}],"diff":{"p1.mana":[4,0],"c12":["p1.hand","p1.graveyard"]},"enqueued":[{"type":"ResolveCardAction","card":"c12","def":"fireball","player":"p1","target":"m7"}]}
{"k":"window","at":"③′","stage":"Decision","responder":"p2","held":{"type":"ResolveCardAction","card":"c12","def":"fireball","target":"m7"},"candidates":[{"card":"c20","targets":[]}]}
{"k":"action","epoch":51,"action":{"type":"RespondInterventionAction","responder":"p2","response":"Intervene"}}
{"k":"window","at":"③′","stage":"Extension","responder":"p2","candidates":[{"card":"c20","targets":[]}]}
{"k":"action","epoch":52,"action":{"type":"SubmitInterventionAction","responder":"p2","card":"c20","targets":[]},"events":[{"type":"CardPlayedEvent","player":"p2","card":"c20"},{"type":"SpellCastEvent","player":"p2","card":"c20"}],"diff":{"p2.mana":[3,0]}}
{"k":"fizzle","action":{"type":"ResolveCardAction","card":"c12","def":"fireball","target":"m7"},"reason":"Canceled","events":[{"type":"CardPlayFizzledEvent","player":"p1","card":"c12"}]}
```

**(3) Intercepted deathrattle (the #18 enqueue-and-drain).** A settle wave enqueues a deathrattle's `DealDamageAction` (`src` = the dead `m3`), which drains through the **full pipeline** — so it reaches its own ③′, where `p2` cancels it; the damage **fizzles** eventlessly:

```json
{"k":"settle","wave":0,"deaths":["m3"],"events":[{"type":"MinionDiedEvent","minion":"m3"},{"type":"DeathrattleTriggeredEvent","minion":"m3"}],"diff":{"m3":["p1.board[0]","p1.graveyard"]},"enqueued":[{"type":"DealDamageAction","src":"m3","tgt":"p2","amount":3}]}
{"k":"window","at":"③′","stage":"Decision","responder":"p2","held":{"type":"DealDamageAction","src":"m3","tgt":"p2","amount":3},"candidates":[{"card":"c30","targets":[]}]}
{"k":"action","epoch":61,"action":{"type":"RespondInterventionAction","responder":"p2","response":"Intervene"}}
{"k":"window","at":"③′","stage":"Extension","responder":"p2","candidates":[{"card":"c30","targets":[]}]}
{"k":"action","epoch":62,"action":{"type":"SubmitInterventionAction","responder":"p2","card":"c30","targets":[]},"events":[{"type":"CardPlayedEvent","player":"p2","card":"c30"}],"diff":{"p2.mana":[2,1]}}
{"k":"fizzle","action":{"type":"DealDamageAction","src":"m3","tgt":"p2","amount":3},"reason":"Canceled","events":[]}
```

**Conventions.**
- **Entity refs are bare ids** (`m3` minion, `c12` card, `a2` artifact, `p1`/`p2` the two players — a **hero is its player id**, no separate `h*` ref; §1 "Card and Minion Identity"); the id → `definitionKey`/name join lives in `state` records — every line stays short, and the visualizer/pretty-printer does the join.
- **Diff keys are dotted paths** (`m7.currentHealth`, `p1.mana`), values `[from, to]`. **Zone moves** use the entity id as key and zone paths as values (`"m3": ["p1.board[0]", "p1.graveyard"]`); board *reorders* trace the same way (index-to-index).
- Action/event payloads inline their §2 fields verbatim — the trace schema **defers to §2** rather than re-declaring shapes.
- **No `queueAfter`** — the remaining queue is derivable from `enqueued` + FIFO order; printing it per action is pure redundancy (a future verbose flag could add it if the queue *discipline itself* ever needs debugging).

JSON Lines keeps the artifact greppable and line-diffable (`grep MinionDiedEvent trace.jsonl`; diff two runs line-by-line; `jq` for ad-hoc queries), and the line-per-record framing means a 30-action cascade reads top-to-bottom as a ledger even raw. Relation to `StabilizationAbortReport` (§4 ⑦): the report's `cascadeTrace` is the event-only cousin captured unconditionally; reproducing the abort under an attached `ITraceSink` yields the strictly richer per-action view (diffs, rejections, enqueue attribution).

### Deterministic Ordering

| Concern | Owner |
|---|---|
| **Canonical board order** (one rule, two consumers) | Active player's `board[0..n]` by index → **their `artifacts[0..2]` by slot → their `inscriptions[0..2]` by index** (trigger side only, 2026-06-11) → opponent's `board[0..n]` by index → **their `artifacts[0..2]` → their `inscriptions[0..2]`** → **neutral zone by index**. Used identically for **trigger fire order** (`IEventBus`, snapshot at *publish* time) and **death/deathrattle/reborn sort** (`DeathResolutionService`, snapshot at *collect* time — artifacts never appear there: they have no health and are not death-wave members). Board index primary; `summonOrder` is the disambiguation fallback. Heroes are not `board[]` members and enter sequencing only via `ITargetSelector` per its own definition. Neutral minions take part in board-wide *event* triggers (last) but **not** turn-scoped Start/End-of-Turn triggers (no turn of their own) — see §3 `IEventBus`. |
| Event visibility (new subscribers) | `IEventBus` — snapshot at publish + `birthEpoch < originEpoch` creation-epoch filter; a listener never receives events from the action that created it |
| Death wave phases | `DeathResolutionService` — Phase 1 (remove) → Phase 2 (deathrattles) → Phase 3 (reborns) |
| Deathrattle before Reborn | `DeathResolutionService` — Phase 3 runs only after all Phase 2 actions complete |
| New deaths during deathrattles | `DeathResolutionService` — deferred to next wave, not resolved mid-wave |
| Resolution cadence | `GameEngine` — one action at a time; ①–⑥ run per action, draining the queue; ⑦ death + ⑧ win fire **only when the queue empties** (cascade settled), then loop. Deaths are batched at the settle point, never mid-cascade — so triggered reactions resolve before removals (amendment B) |
| Interception (declaration window) | `GameEngine` — stage ③′: an action suspends before ④; the response resolves **immediately** and fully (board/auto triggers fire inline; the **depth-1 cap** bars only a further *player window* on a response — *nesting* — while auto hosts — inscriptions, board/artifact triggers — still fire, and may intercept, inline), then the held action re-validates and, if still legal, **re-declares (③′)** — re-opening the **set-valued** window over the responder's remaining matching+affordable cards until they **skip** or the action fizzles/executes. **Iteration** (one responder chaining their own cards on one action) is bounded by **both** the per-turn press budget (`MaxInterventionsPerTurn` — each chained play is a fresh intervene-press) **and** next-turn mana, whichever binds first; **nesting** is capped at depth-1. The only deviation from FIFO (point D) |
| Post-reaction window | `GameEngine` — stage ⑥′: after each action's ⑤/⑥, **one set-valued window** offering all the responder's matching + affordable reactive hand cards; play one (card + targets from its matched set) or **skip**; a **play** re-offers the remaining set, a **skip** ends it. Response resolves immediately before the queue resumes; deaths **not** pre-settled. Per action — never per-event, never per-cascade |
| Randomness | `IRandom` — per-action stream derived from `mix(rngSeed, currentActionEpoch)`; consumed only in stage ④ handlers; draw order fixed by the rows above |

---

## Section 4: Action Processing Pipeline

The pipeline is a **loop**, not a single linear pass (cadence amended 2026-06-03 — Fireplace amendment B). Stages **①–⑥ run per action** as each is dequeued; triggered reactions enqueue further actions, and the engine keeps dequeuing and running ①–⑥ until the **action queue is empty** — the point at which the cascade started by a top-level action has fully *settled*. Only then do the **settle stages** run: **⑦ Death Resolution** and **⑧ Win Check**. **⑨** drives the loop (dequeue next, or trigger the settle when the queue is empty). The engine never advances a stage until the current one completes.

Because deaths are processed only at the settle point, **every triggered reaction from the settling cascade resolves *before* any minion is removed**: a minion reduced to ≤0 HP (or destroy-marked) is **pending death** — still on the board, still targetable and countable, still keyword-active — until the settle, not removed on the spot. (Terminology, 2026-06-12: "mortally wounded" names the *health-domain* kind, which fires `MinionMortallyWoundedEvent` and is savable; a destroy-mark shares only the lingering board presence — no event, no save; §1 `currentHealth` / §2A `DestroyMinionAction`.) This is the Hearthstone-faithful cadence; it replaced an earlier per-action model where ⑦ ran at the tail of *every* action (so a dying minion vanished before same-event reactions ran). See amendment B in the borrow-list note for the design rationale, the mechanics it unlocks (overkill, retaliation, simultaneous-death, count-the-doomed, Intervention × lethal), and the worked example.

When a debug trace sink is attached (§3 `ITraceSink`), the pipeline emits one trace record per processed action (④), rejection (②/③), window/choice opening (③′/⑥′), and death wave (⑦) — plus full-state snapshots at match start and each settle-to-idle. See §3 for the record vocabulary.

### ① Submit

Player action arrives via WebSocket, or system action is dequeued from `IActionQueue`. `Submit` returns a `SubmitResult` — `Accepted(events)` or `Rejected(ActionRejection)` (see §2C).

### ② Phase Guard

Rejects the action if it is not valid in the current `GamePhase`, returning `Rejected(WrongPhase)` with no state mutation (see §2C).

| Phase | Accepted actions |
|---|---|
| `WaitingForPlayers` | None — pre-game: awaiting both players' connections. **Match Setup** runs on the second connect, then → `Mulligan`. |
| `Mulligan` | `SubmitMulliganAction` only |
| `InProgress` | All player-initiated actions |
| `PendingChoice` | `SubmitChoiceAction` from the waiting player only |
| `InterventionDecision` | `RespondInterventionAction` (`Skip` \| `SkipAll` \| `Intervene`) from the responding (non-active) player only — `Intervene` rejected (`NoInterventionBudget` \| `NoMatchingCard`) unless `turn.interventionsUsed < MaxInterventionsPerTurn` **and** `candidates` non-empty |
| `InterventionExtension` | `SubmitInterventionAction` from the responding player only (Stage-2; `chosenCardId` null = back out) |
| `Ended` | None |

System actions bypass Phase Guard — they are trusted internal commands.

### ③ Action Validator

Precondition checks. Returns `Rejected(ActionRejection)` with the matching code and no state mutation on failure (codes per §2C):

- Turn ownership: is this the active player? → `NotActivePlayer`
- Mana affordability: `effectiveCost ≤ player.mana` → `NotEnoughMana`
- Target validity: target is not stealthed (for opponent spells/attacks) and is a **member of the card/effect's declared target selector's output** (`chosen ∈ selector.Select(context)`, plus the cardinality/optionality rule — see §3 `ITargetSelector`) → `InvalidTarget` / `TargetStealthed`. **Aliveness is *not* checked here** (2026-06-13): the policy lives entirely in the declared selector (`All…/Alive…/Dead…`), so a save-targeting `Dead…`/`All…` effect may legally target a mortally-wounded (≤0-health, still-on-board) minion — a flat "is alive" gate would contradict the `Dead…` family. (The *attacker*-aliveness check for `AttackAction` is separate — see "Attack eligibility" below.) The same selector defines the client's legal-target highlighting, so preview and validation cannot drift.
- Attacker control: the submitter may attack only with a minion **it controls** — `attacker.ownerId == submitter` → else `AttackerNotControlled`. This is an explicit check (it closes the gap where the old "is this the active player?" line let a player swing with an opponent-owned minion), and it means **neutrals are never default attackers** (`ownerId == null` controls for no one). The **sole exception** is `CommandAttackAction`, a card-granted effect whose validation permits a *neutral* `attackerId` (the relaxed attacker rule); it still runs every other check below.
- Taunt enforcement — **per lane** (amended 2026-06-07): the constraint is scoped to the **target's lane**. Attacking into the **opponent lane** → if that opponent has Taunt minions *on their own board*, the attack must target one (else `MustTargetTaunt`); face is legal when none. Attacking into the **neutral lane** → if a neutral Taunt is present, the attack must target it. **Cross-lane Taunt never applies** — a neutral Taunt does not lock you out of the opponent's hero, nor an opponent Taunt out of a neutral. (Supersedes the earlier "Taunt is ignored for neutral minions" scope; neutral Taunt is now honored, but only within the neutral lane.) The same rule governs `CommandAttackAction`.
- Attack eligibility: the attacker is **alive** — `currentHealth > 0` and not destroy-marked (else `AttackerNotAlive`) — not frozen (`isFrozen = false`, else `AttackerFrozen`), and has an activation left (`attacksUsedThisTurn < budget`, else `AttackerExhausted`, where the per-turn **budget** is *pulled* from effective keywords — Windfury → 2, else 1 — never a stored field, so silencing Windfury immediately drops the budget and a once-swung minion is correctly exhausted). **Summoning-sickness + haste is one *pulled*, target-aware check, not a stored verdict:** if the attacker is `summoningSick` (it entered a board this turn and has not woken at a controller turn-start — set on entry by **both** summon and control), the engine pulls its effective keywords —
  - **Charge** → may attack any legal target;
  - **Rush** → may attack a **minion only** (enemy or neutral; per-lane Taunt still applies), and a hero/face target is rejected `RushCannotTargetHero`;
  - **neither** → it cannot attack at all → `AttackerCannotAttack`.

  A non-sick attacker may hit any legal target. Pulling Charge/Rush here (rather than baking a `canAttack` bool at entry) is the §3 `IKeyword` philosophy: aura-granted haste works mid-turn and silenced haste is correctly lost, with no recompute — and the shared board-entry routine (summon **and** control) does nothing but set `summoningSick`, never a per-handler charge/rush rule. (Neutrals are never `summoningSick`; a `CommandAttackAction` bypasses this eligibility entirely, and retaliation is ungated — see the fizzle note next.)

  The aliveness check is what makes a **mortally-wounded attacker fizzle on re-validation**: a "destroy / return the attacker" interception (§3) leaves the attacker ≤0 / destroy-marked but on board (death is deferred to ⑦), and the held attack, re-entering ③ on resume, is rejected here before it can swing — and likewise the second strike of a **Windfury command** fizzles if the first retaliation mortally wounded the neutral. **This fizzle is the *attacker's* concern only — never the defender's retaliation.** Eligibility (③) gates the *active swing* of an `AttackAction` / `CommandAttackAction`; a defender's **retaliation** is a separate `DealDamageAction` whose source is the defender, and it is **not** subject to this check — so a **mortally-wounded defender still retaliates with full effect** (live keywords + stats, because death is deferred to ⑦). A minion killed-but-not-removed swings back; it just cannot *initiate* a fresh swing.
- Board space: player board has fewer than `GameConstants.MaxBoardMinions` (7) minions before summoning → `BoardFull`

**Validation is a pure, standalone-invokable predicate.** Stages ② and ③ together form a side-effect-free function `(action, state) → Ok | Rejection` (where `Rejection` is the `ActionRejection` of §2C) — no mutation, no dispatch (④), no events. It is the **single source of truth for legality**: `Submit` runs it before ④, and any "is this action legal?" / "what actions are legal?" query — a future `GetLegalActions(playerId)`, bot move generation, or client pre-flight (greying out unaffordable cards / illegal targets) — **must call this same predicate** rather than re-deriving legality. This guarantees previewed/enumerated legality can never drift from what `Submit` actually accepts. (The positive-enumeration API itself — `GetLegalActions`, via brute-force-then-filter over candidate actions — is deferred to the future bot-support epic; only the standalone-predicate seam is part of the core.)

### ③′ Declaration & Interception Window

A pre-execution checkpoint inserted between validation (③) and dispatch (④) — it does **not** renumber ④–⑨. (Decided 2026-06-04, Fireplace point D; the model lives in §3 "Reactive Triggers, Interventions & Interception".) For every action that reaches dispatch:

1. Publish **`ActionDeclaredEvent { action }`** — the universal pre-execution signal (one event type for all actions; subscribers filter on the carried action's type/params). Most actions have no matching reactive trigger and pass straight to ④. **An action carrying `suppressDeclaration` is not declared at all (DR1, 2026-06-28):** like the `Start*`/`Respond*` system actions below, an `Inexorable`/`Uncounterable` card's play-unit — and, under `Inexorable`, the effects it enqueues (§3 "Non-interceptable actions") — publishes **no** `ActionDeclaredEvent` and opens **no** decision window, passing straight to ④ with no ③′, so no auto-interceptor or player window can touch it. Its post side (⑤/⑥/⑥′) is unaffected.
   - **Window/choice-opening is an engine-internal checkpoint, not a declarable action (#27).** The decision/extension windows and Discover prompts are opened by the engine *here* (steps 2–3), inline — **not** by a queued action a sigil could intercept. **Two disjoint classes of action skip ③′** (publish no `ActionDeclaredEvent`, open no window): (i) the **system / effect-issued** window-and-choice *openers* `StartInterventionAction` / `StartChoiceAction`, which also **bypass Phase Guard** (§4 ②, "System actions bypass Phase Guard"); and (ii) the **player-submitted, phase-gated** window/choice *responses* `RespondInterventionAction` / `SubmitInterventionAction` / `SubmitChoiceAction`, which do **not** bypass Phase Guard — each is gated to its own phase at ② and is exempt only from the ③ turn-ownership check and from ③′. The unifying property is **"window/choice machinery, not a state-mutating board move"** — *not* "system action": labelling the player-submitted responses "system actions" (corrected A2/A3, 2026-06-28) was wrong, since a system action bypasses Phase Guard whereas these are phase-gated; and `SubmitChoiceAction` was previously omitted from this exemption altogether (A3), which by the letter would have made resolving a Discover open a spurious decision window. "Every action declares" is scoped to the state-mutating *game moves*; the machinery above is exempt. This closes the recursion surface where a sigil could "cancel the opening of a window," "counter a Discover," or "counter the *picking* of a Discover option." (`Start*` remains the queueable form an *effect* uses to issue a choice — e.g. a Discover battlecry enqueues `StartChoiceAction`; that enqueue is still not itself interceptable. The pre-game `SubmitMulliganAction` and the administrative `SurrenderAction` likewise do not declare at ③′.)
2. **Auto hosts resolve inline (no halt):** an **off-turn inscription** (§3 "Sigils"; choices degrade to random) or a board/artifact trigger that matched fires here. **Then, unless there is no off-turn opponent (`activePlayerId == null`), the engine opens a Stage-1 decision window for the non-active player** — *unconditionally*, whether or not they hold a matching hand card (§3 "The responder & window delivery") — **suppressed only** by a prior `skip-all` this action, an exhausted budget (`turn.interventionsUsed == MaxInterventionsPerTurn`), or the responder's `autoSkipAll` (in which case the action proceeds straight to ④ as an auto skip-all). When opened, it suspends the action as `pendingIntervention.heldAction`, records the responder's matching + affordable cards (**possibly empty**) with their computed targets in `candidates` (§1 `InterventionCandidate`), sets `phase = InterventionDecision`, and emits the responder-directed `InterventionPromptEvent` (cause = the **held action verbatim**; §3 visibility) and the public `InterventionWindowOpenedEvent { stage: Decision }` (§2B). `Intervene` (`turn.interventionsUsed++`) advances to `InterventionExtension` (§4 "PendingIntervention Interruption").
3. **Depth-1 cap (nesting only):** while an interception response is resolving, **no further *player window*** opens to halt it — but an **auto host — an inscription or a board/artifact trigger — may still fire, and even intercept, inline** (e.g. a pre-damage hero-protecting inscription firing on a `DealDamageAction` that a response enqueues), since the cap bounds only *player windows*, not reactions. (Auto-resolving reactions chain in FIFO order.) This bars a nested LIFO stack *of player windows*. It does **not** bar **iteration**: the responder may chain several of *their own* held cards against the one action — see the re-declare loop below.

**Re-declare loop (2026-06-09 — reverses the former "declared once" rule).** When a held action resumes after a **played** response, it re-enters ③ for re-validation; if still legal it is **re-published to ③′**, re-evaluating the responder's *remaining* matching + affordable cards and re-opening the window if any remain (else it proceeds to ④). The loop ends when the responder **skips** (the held action proceeds to ④), when no affordable match remains, or when re-validation **fizzles** the action (cancelled / retargeted off its subject / attacker dead — §3, §4 ③). A **skip never re-opens** the window — only a play can change the offer. The loop is bounded by **both** the per-turn press budget (`MaxInterventionsPerTurn` — each re-declare play is a fresh intervene-press, §3) **and** the responder's next-turn mana (spent from `mana`, pre-loaded at the previous turn-end — see Turn Lifecycle); whichever binds first ends it. See §3 for what a response may do (ordinary effects + re-resolution; only `cancel`/`retarget` touch the held action directly).

### ④ Dispatch → `IActionHandler`

The registered handler for this action type is invoked. It mutates `GameState` and returns `events[]`. This is the sole point of state mutation in the pipeline. `GameEngine` increments `GameState.currentActionEpoch` as the action enters this stage; every event the handler returns is stamped with that `originEpoch`, and any subscription the handler registers (e.g. via `ICardHandler.OnSummon`) is stamped with the same value as its `birthEpoch` — see §3 `IEventBus`. The engine also derives this action's `IRandom` from `mix(GameState.rngSeed, currentActionEpoch)`; every random outcome the handler produces is drawn from that instance (see §3 `IRandom`).

### ⑤ Publish Events → `IEventBus`

Events are published in the order returned by the handler. For each event, `IEventBus` snapshots the subscriber lists at publish time and fires them filtered by creation epoch (`birthEpoch < originEpoch`, see §3 `IEventBus`) in the canonical board order: current player's list first (minions by current board index, then their artifacts by slot, then their inscriptions by slot), then opponent's list (the same), then the neutral zone's. The epoch filter means a listener never receives an event from the action that created it. Subscribers enqueue new actions via `IActionQueue` — they do not process them inline.

### ⑥ Aura Recalculation

All registered `IAura.Calculate(state)` implementations are run. `auraAttackBonus` and `auraHealthBonus` on every minion — **including the neutral lane** — are zeroed and fully rewritten (a neutral minion stays at 0 bonus unless some aura's `Calculate` selects it; see §3 `IAura` scope note). A minion newly at ≤0 `maxHealth` (an aura-loss death) **enters pending-death** (publishing `MinionMortallyWoundedEvent`) — but it is **not removed here**; like a damage-killed minion it lingers, mortally wounded, until the next settle (⑦).

Recalculation runs **per action** (⑥, while draining) — after any action that changes board composition, minion stats, or keywords — and also after each action processed in Phase 2 of the death wave and after Phase 3 reborn summons. Keeping aura recalc per-action (cheaper than deferring it) means mid-cascade reactions always read fresh stats; only *death* is deferred to the settle, not aura math.

### ⑥′ Post-reaction Window

A checkpoint inserted after aura recalc (⑥), before the loop driver (⑨) considers the next action — the post-phase twin of ③′, and like ③′ it does **not** renumber the stages. After an action's events are published (⑤) and stats refreshed (⑥), the engine checks for **hand-card post-reactions** whose triggers matched **this action's** events:

1. **Unless `activePlayerId == null`**, the engine opens a Stage-1 decision window for the **non-active** responder — *unconditionally* (§3 "The responder & window delivery"), suppressed only by a prior `skip-all` this action, an exhausted budget (`turn.interventionsUsed == MaxInterventionsPerTurn`), or the responder's `autoSkipAll`. It gathers the responder's reactive hand cards whose triggers matched **this action's** events and that are **affordable** (`effectiveCost ≤ mana`) — each with its computed targets — into `candidates` (**possibly empty**; §1 `InterventionCandidate`), sets `phase → InterventionDecision` with `heldAction = null`, and emits the responder-directed `InterventionPromptEvent` (cause = `causeEventIds[]`, refs to this action's matched events, already on the responder's wire from ⑤; §3 visibility) and the public `InterventionWindowOpenedEvent { stage: Decision }` (§2B). `Intervene` advances to `InterventionExtension`. Each card's offered targets are its matched subjects from this action ∩ its declared `(selector, cardinality)` family.
2. **Deaths are not pre-settled here** — a post-reaction is *about* what just happened, so a mortally-wounded subject must stay on the board to be acted on. ⑦ runs only at the eventual full-drain settle (⑨), **uniformly**: no window or halt force-settles deaths early (G8, 2026-06-24).
3. The player plays **one** card — choosing it and its `cardinality` targets — or **skips**. The response **resolves immediately** (see "Response resolution"). A **play** then **re-offers** the *remaining* matching + affordable cards against the same (frozen) events with the now-current board (a card whose condition lapsed — its subject left — drops out); a **skip** ends the window. Only a play re-opens it (the **re-offer loop**, 2026-06-09).

Auto reactions (inscriptions, board/artifact triggers) do **not** open ⑥′ windows — they fired inline during ⑤ in FIFO order and enqueued their actions. Only player-choice hand cards open a ⑥′ window.

### ⑦ Death Resolution

**Settle stage — runs only when the action queue has drained to empty** (the cascade started by a top-level action has fully settled), never mid-cascade. While the queue is non-empty the engine stays in ①–⑥, so every triggered reaction resolves first and a mortally-wounded minion remains on the board (targetable, countable, keyword-active — and savable by a dying-window **save-from-lethal** intervention — a player responding to lethal damage to rescue it before removal, the "R1" avenue in amendment B) right up to this point. Then `DeathResolutionService` runs the wave loop:

1. **Collect** all minions with `currentHealth ≤ 0` or marked for destruction. Sort: current player `board[0..n]` → opponent `board[0..n]` → neutral zone by index. If none, exit loop. Otherwise publish `DeathWaveStartedEvent { waveIndex }` (0-based, incremented per wave).
2. **Phase 1 — Remove & grieve:** for each in sort order — remove from board, build a `GraveyardMinion` from its death state (`snapshot`; **no card is fabricated here** — the card form is produced lazily at point of use, §1), **route it to a graveyard**, then publish `MinionDiedEvent`. Routing reads exactly two inputs:
   - `ownerId != null` → **that player's** `graveyard` (a controlled minion — including a once-`bornNeutral` body taken over via `TakeControlAction` — grieves to its controller).
   - `bornNeutral && ownerId == null` → the shared `GameState.neutralGraveyard` (a system-spawned neutral that died in the neutral lane).
   - `!bornNeutral && ownerId == null` → **undefined, asserts** — no in-scope path creates a player-originated body in the neutral lane (`TakeControlAction` only ever moves bodies *out*). See Unaddressed Features, "A non-`bornNeutral` body dying in the neutral lane."
3. **Phase 2 — Deathrattles:** for each in sort order — call `ICardHandler.OnDeath`, which enqueues this minion's deathrattle actions. **Process the full action queue: each enqueued action is an ordinary queue citizen running the complete pipeline ①–⑥ — including its ③′/⑥′ interception windows** (so a deathrattle's `DealDamageAction` can itself be countered/redirected, and an enqueue-and-drain action sees fresh aura math; 2026-06-13, #18). New deaths during this phase are deferred to the next wave. Minions summoned during this phase are registered with the `birthEpoch` of their own `SummonMinionAction`, so the creation-epoch filter (§3 `IEventBus`) keeps them from reacting to `MinionDiedEvent`s already published in this wave — they react only to events from subsequent actions.
4. **Phase 3 — Reborns:** for each minion in the current wave with the Reborn keyword **and** `rebornAvailable == true` (in sort order) — enqueue `SummonMinionAction` with `currentHealthOverride = 1` (the "at 1 HP" param, #20). Process fully. Run aura recalculation. The reborn copy **keeps the Reborn keyword** but is summoned with `rebornAvailable = false` (the Phase-3 summon explicitly overrides the default-`true` init for the copy — F5), so it cannot reborn again (if it dies it goes to graveyard normally). The keyword still counts for auras and survives bounce. A *copy* made of that spent minion is a fresh summon → `rebornAvailable` re-initializes to `true` (every fresh summon's unspent charge), so the clone reborns if it dies (it has the keyword **and** a fresh charge).
5. Publish `DeathWaveEndedEvent { waveIndex, minionsResolved }`. Back to step 1 (next wave).

**Termination — iteration cap.** The wave loop is bounded by `GameConstants.MaxDeathWaves = 16`. Reborn cannot drive an unbounded loop (it is self-consuming, step 4), so the only runaway source is Phase 2 deathrattles/summons that keep producing new deaths — almost always a card-definition bug. If the loop is about to begin a wave whose `waveIndex` would reach `MaxDeathWaves`, the engine treats this as a fatal assertion failure, **not** a game event:

1. Publish `StabilizationAbortedEvent { wavesReached, lastWaveMinionIds[] }`.
2. Publish `GameEndedEvent { winnerId: null, reason: NoContest }` and set `phase = Ended`. The match is **voided** — not a draw (a draw would unfairly affect MMR).
3. Emit a `StabilizationAbortReport` to the telemetry sink (below).

The action is **not** rolled back and replayed — rollback would restore the exact state the runaway re-triggers from, so it cannot resolve the match. Aborting the whole match is the only safe escalation. There is no rollback/snapshot dependency: the engine simply tears the match down.

This is a last-line backstop. The primary mitigation is upstream content validation at card-load time (flag obvious self-feeding patterns such as a deathrattle that summons a 0-health token); general termination is undecidable, which is why the runtime cap remains. **`MaxDeathWaves = 16` is provisional (DR6, 2026-06-28)** — a `GameConstant`, hence tunable, set high enough that it should only ever bite a true runaway, not a deep-but-finite *legal* cascade (e.g. many deathrattle-summon-deathrattle bodies across both boards + the neutral lane). It is to be revisited against real cascade-depth telemetry, and the card-load validator (above) is the real guard — the runtime cap must never be the thing that voids a board a player legitimately built toward.

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

A regression scenario built from a report asserts: submitting `triggeringAction` against `preActionState` (seeded with `rngSeed`) terminates at wave `MaxDeathWaves` with `StabilizationAbortedEvent` then `GameEndedEvent { NoContest }` — i.e. it aborts rather than hangs. (Re-running that scenario with an `ITraceSink` attached (§3) upgrades the event-only `cascadeTrace` to the full per-action debug view — diffs, rejections, enqueue attribution.)

### ⑧ Win Condition Check

**Settle stage — runs immediately after ⑦**, at the same queue-empty settle point. ⑧ evaluates the **registered win-condition list** against the freshly-settled state and resolves the outcome:

- The **built-in entry** is the hero-death check (always present, never a registered card effect): if any hero has health ≤ 0 it yields that hero's *opponent* as winner.
- Each **registered `IWinCondition`** (§3 — the aura-shaped declarative win-seam; v1 ships **only** the built-in entry, alternate-win content is v2) contributes `Evaluate(state) → winnerId?` (null = "no win from me"): mill-as-win, a completed quest counter (`ArtifactOnBoard.charges ≥ goal`), an alternate-resource threshold, etc.
- **Resolution:** if exactly one player is yielded as winner, that player wins; if **two different** players are yielded on the same settle (mutual hero-death, or a quest completing on the very settle its holder is killed), it is a **draw** — the general form of the mutual-lethal rule. `GameEndedEvent { winnerId?, reason }` is published and `phase` is set to `Ended`.

**Why a list checked *here*, not a flag pushed mid-cascade (DR5, 2026-06-28).** Win conditions register/deregister zone-scoped exactly like `IAura`/triggers, and ⑧ **re-evaluates them fresh at this settle** — none is marked mid-cascade. This inherits the correctness the hero check already relies on: a condition momentarily true mid-cascade but **undone by a post-reaction save before the settle** (the dying holder healed, the quest counter decremented, the alt-win artifact destroyed) correctly does **not** win, because ⑧ reads the settled board, not a stale snapshot. (A "push a `pendingWin` flag when the triggering event fires" design would have to re-verify the condition at this settle anyway — collapsing back into exactly this predicate; pull-at-settle is the form that composes with saves and mutual-lethal.)

Mirroring ⑦'s pending-death window: a hero that crosses to ≤0 **mid-cascade** publishes **`HeroMortallyWoundedEvent`** at that moment (from the damaging action's ④), but the loss is **not** finalized until this settle — so a post-reaction save (heal-above-0 / immune, §3) enqueued onto the still-draining queue resolves first, and ⑧ then re-reads actual hero health and finds it survived. This gives heroes the same dying-window the death wave gives minions; simultaneous mutual lethality still settles here as a draw — the built-in instance of the resolution rule above.

### ⑨ Dequeue / Drive Loop

The loop driver. **If the action queue is non-empty**, pull the next action and return to ① (continue draining the cascade through ①–⑥, including ⑥′). **If the queue is empty**, run the settle stages ⑦ (death wave) then ⑧ (win check); the death wave may enqueue actions (deathrattles, reborns), so if the queue is non-empty again, resume draining — otherwise the cascade is fully settled and the engine awaits the next player action. (A player-submitted action is just the next thing dequeued, starting a fresh cascade.)

Post-reaction windows are **not** opened here at the settle; they open **per action at ⑥′**, immediately after each triggering action. A save therefore resolves right after its cause — well before the eventual ⑦/⑧ — so it can lift a mortally-wounded minion back above 0 before any death wave collects it, without waiting for or racing against the full-cascade settle.

---

### Match Setup

Runs **once**, before the first turn, when the second player connects (the `WaitingForPlayers` → `Mulligan` transition — ② phase-guard). All randomness is seeded from `rngSeed` via `IRandom` (§3), so setup is replay-exact. Bracketed values are `GameConstants` (the single source).

1. **Players & heroes.** Both `PlayerState`s are built from their decklists. Each hero starts at `StartingHeroHealth` (30) health **and** max (the `HealAction` cap, #12), `armor = 0`, `mana = maxMana = 0`, `fatigueCounter = 0`. Each player is equipped the shared **starter artifact** via `EquipArtifactAction` (§2A); any triggers/auras it carries register at **`birthEpoch = 0`** (the match-start epoch — identical to the opening-hand sigil rule, step 7; setup runs outside the turn pipeline, so this is *not* the normal ④ epoch stamp), so they are live from turn 1 and do not react to setup's own events. *(Moot for the v1 active-only starter — "pay mana, draw" registers no triggers/auras — but pins the rule for a future passive/hero artifact; G7, 2026-06-22.)*
2. **Deck assertion & shuffle.** The engine asserts each decklist is exactly `DeckSize` (30) cards within `MaxCopiesPerCard` (2; 1 per Legendary) — primarily server-side deck validation, asserted here as a guard — then Fisher-Yates-shuffles each deck over `IRandom`.
3. **First player.** A seeded coin flip picks who goes first.
4. **Opening hands.** Deal `StartingHandFirstPlayer` (3) to the first player, `StartingHandSecondPlayer` (4) to the second. **Guaranteed opening card (deck-build pin):** a decklist may flag **one** `guaranteedOpeningCardKey`; after the shuffle, if no copy of that `definitionKey` already lies within the player's opening-hand slice, the engine **swaps the top-most copy into the opening hand** — a pure post-shuffle position swap that consumes **no** `IRandom`, so the remaining deal is unbiased and replay stays exact. The pin guarantees only the initial deal — the card carries **no** mulligan protection (it can be replaced like any other). Optional (a deck may have none).
5. **Mulligan (HS-faithful).** `phase → Mulligan`; each player sees their own opening hand (directed `MulliganStartedEvent`, own options only) and may replace **any subset** (`SubmitMulliganAction { cardIdsToKeep[] }` — the complement is replaced). The **replacement cards are drawn first**, *then* the replaced cards are shuffled back into the deck — so a replaced card cannot be immediately redrawn. Both `MulliganCompletedEvent`s in (`mulliganState` both-complete) ends the phase. *(Sigil hand-trigger registration is deferred to step 7, after the hand settles — so a mulligan swap needs no per-card trigger bookkeeping; F7.)*
6. **The Coin.** Added **after** the mulligan (so it cannot be mulliganed away — F6): the second player additionally receives **The Coin** — a 0-cost card whose effect enqueues `ModifyManaAction { delta = +1 }` (the going-second equalizer; it may push `mana` above `maxMana`, §2A #22). This stands in for HS's temporary mana crystal, which this model does not have.
7. **Opening-hand sigil registration.** The opening hand is **settled** here (post-mulligan, incl. The Coin), outside the action pipeline, so any reactive (sigil) cards in the **final** hand register their hand-zone triggers **at setup with `birthEpoch = 0`** (the match-start epoch) — *not* in the `DrawCardAction`/`GiveCardAction` handler, which covers only in-game additions. Running this **after** the mulligan (F6/F7) means a mulligan swap needs no unregister/re-register dance — only the cards actually in hand get registered, once. They are live for invocation from turn 1. *(Nothing reactive fires during the Mulligan phase, so sigils need not be live earlier.)*
8. **Turn 1.** Seed **both** players' `mana = maxMana = StartingMana` (1) **directly** — each player's *opening* turn has no preceding turn-end refresh, so setup provides the turn-1 crystal for both (seeding only the first player would strand the going-second player at 0 mana on their first turn, since the turn-end refresh only ever refreshes the *ending* player — F1). The going-second seed doubles as that player's off-turn intervention pool during the first player's turn 1 (the carry model — Turn Lifecycle step 4). `phase → InProgress`; seed `turn.number = 1` and `turn.activePlayerId = ` the first player (the lifecycle step-6 increment carries it from turn 2 onward — F13), then begin turn 1. The neutral lane is **empty** until its first Interturn handover (end of turn 3 — `NeutralZoneConfig.firstSpawnTurn`).

---

### Turn Lifecycle

The `EndTurnAction` handler and subsequent system actions implement the full turn transition:

1. Publish `TurnEndedEvent`
2. Fire End-of-Turn triggers (current player L→R, then opponent L→R) — **neutral minions are intentionally excluded** (turn-scoped; a neutral minion has no turn of its own — see §3 `IEventBus`)
3. **Unfreeze sweep — for the player whose turn is ENDING** (before the swap, so the freeze actually costs an attack rather than thawing at the start of the controller's turn): thaw each frozen minion they control **unless** it was frozen *this* turn while exhausted — i.e. keep frozen iff `frozenOnTurn == turn.number && attacksUsedThisTurn >= budget` (the same exhaustion test as `AttackerExhausted`, §4 ③); otherwise set `isFrozen = false`, clear `frozenOnTurn`, emit `MinionThawedEvent`. This makes Freeze cost exactly one attack (a minion frozen after swinging stays frozen through its next turn; one frozen on the opponent's turn, or this turn before swinging, thaws at the end of the turn it was unable to act). Minions only — heroes have no frozen state (§1).
4. **Mana refresh — for the player whose turn is ENDING** (moved from turn-start to turn-**end**, 2026-06-09): increment *their* `maxMana` (cap 10) and restore *their* `mana` to `maxMana`. Refreshing here pre-loads the full pool the player then carries **through the opponent's turn**, so an off-turn **intervention** spends straight from `mana` (the ordinary `effectiveCost ≤ mana` check) and the player's own next turn simply begins with whatever interventions didn't consume — **no `reservedMana` field, no separate ceiling**; the affordability ceiling is naturally their *full ramped* next-turn pool. (The game's first turn has no preceding turn-end, so game setup seeds each player's turn-1 `mana`/`maxMana` directly. A future overload-style "less mana next turn" effect would apply its reduction here.)
5. **Interturn — the neutral lane's "turn" (`activePlayerId = none`).** Runs after the ending player's cleanup and before the swap, owned by **neither** player. For its duration `turn.activePlayerId` is **none** (null): with no active player, neither player's trigger groups fire — **only the neutral group does** (§3 `IEventBus`), the exact dual of neutral minions being excluded from player turns. **Repopulation:** if `neutralZoneConfig != null` **and** `turn.number ≥ neutralZoneConfig.firstSpawnTurn` (default 3 — the end of P1's second turn), **refill the neutral lane to `maxCapacity`** by enqueuing `SpawnNeutralMinionAction { definitionKey = null }` × (`maxCapacity − currentCount`) — each draws its definition from `spawnPool` at its own ④ `IRandom` and emits `MinionSummonedEvent { ownerId = null }`; the batch is bracketed by `NeutralZoneRepopulatedEvent` (renderer marker). The first qualifying handover finds an empty lane and **seeds** full capacity; every later one **tops it up** — initial seed and recurring refill are one rule, no special branch. **Consequence:** a player's board-wide on-summon (or other) reaction does **not** fire on a handover spawn — the neutral lane's turn is walled off from both players (a neutral *card-spawned on a player's turn* still triggers player reactions normally; only the handover is walled). No-op before `firstSpawnTurn` or when the lane is already full. **Reserved seam:** the Interturn step is the *only* owned-by-neither, fair-dispatch, once-per-handover point in the loop; neutral repopulation is v1's sole inhabitant (other candidates — neutral-minion upkeep, environmental/round effects, shared-resource ticks — recorded in the borrow-list note).
6. **Advance the active player** to the incoming player (out of the Interturn `none` state — the one who did *not* just end their turn); **increment `turn.number`** (F13 — the global turn counter; steps 1–5 above ran with the *ending* turn's number, which is why Freeze's `frozenOnTurn == turn.number` test in step 3 and the Interturn `turn.number ≥ firstSpawnTurn` gate in step 5 both read the just-ended turn; the increment lands here so the new turn carries the next number); **reset `turn.interventionsUsed = 0`** — the fresh per-turn intervene-press budget (`MaxInterventionsPerTurn`) for the new non-active player
7. Clear `summoningSick` (wake) and reset `attacksUsedThisTurn` on all of the new active player's minions
8. Reset **the new active player's** `cardsPlayedThisTurn` (#23 — Combo is per-player and on-turn; an off-turn invocation never incremented it); reset `usesThisTurn` on each of the new active player's artifacts (re-arms escalating activation costs — §1 `ArtifactOnBoard`)
9. Enqueue `DrawCardAction`
10. Publish `TurnStartedEvent`
11. Fire Start-of-Turn triggers (new active player L→R, then opponent L→R) — neutral minions excluded (turn-scoped; step 2)
12. Start turn timer

Each of these produces actions that go through the full pipeline.

---

### Response resolution (both window kinds)

Whenever a window's response is submitted, the response's effects **resolve to completion immediately** — the engine processes the response's enqueued actions (and any inline board/auto reactions they chain, FIFO) **fully, before returning to drain the pre-existing queue.** Then the window **loops**: a declaration-hold resumes its `heldAction` (re-validated ②③) and, if still legal, **re-declares (③′)**; a post-reaction **re-offers** its remaining matched set at ⑥′. The loop re-opens **iff the response played a card** — which shrinks the offer and may change the board; a **skip** changes nothing and so **terminates** the loop (a declaration-hold's `heldAction` proceeds to ④; a post-reaction's cascade resumes). Each played card pays `effectiveCost` from the responder's `mana` (their next-turn pool, pre-loaded at turn-end — Turn Lifecycle); when nothing affordable remains, the loop ends.

Immediate resolution is **load-bearing, not an optimization**: were the response merely FIFO-appended behind the rest of the cascade, the cascade would drain to queue-empty and the settle (⑦) would remove the very minion a save just targeted before the heal ran. Resolving the response immediately — inside the window, ahead of the pre-existing queue — lands the save before any death wave collects its subject. The **depth-1 cap** applies **within** a response — while it resolves, its own sub-actions open no further *player* windows (③′ and ⑥′ are suppressed); **board/auto triggers still fire inline**. The cap bars only **nesting** (a further *player window* on a response), never **iteration** (the loop re-opening the *same* responder's window after the response completes) and never inline reactions — an auto host may even *intercept* a response's own sub-action inline (e.g. a pre-damage inscription on damage the response deals).

### PendingChoice Interruption

When an effect issues `StartChoiceAction`:
- **No early death settle — uniform cadence (G8, 2026-06-24).** A choice halts the pipeline *where it is*; the settle stages **⑦ death / ⑧ win still run only at the eventual queue-empty (⑨)** — no halt force-settles deaths early. So the chooser decides against the **live** board, mortally-wounded (≤0-health, still-on-board) minions included, exactly as every other action and every ⑥′ post-reaction sees them; whether a choice's selector *offers* a dying minion is the `All…/Alive…/Dead…` decision (§3 `ITargetSelector`), not a pipeline rule. This keeps a choice's board identical to the equivalent battlecry's, and it is what lets a **save reach a dying minion through a Discover/Target chain** — the minion lingers across the choice(s), an inclusive selector can target it, and the heal lands before the death wave. *(Removes the former "pre-halt death rule / amendment B" that force-settled ⑦ before a `PendingChoice` or declaration-hold: it gave choices a privileged board no other action gets, contradicted the still-on-board targeting rule of §4 ③ at :1044, and silently broke save-via-Discover.)*
- `GamePhase` → `PendingChoice`
- Remaining effect actions (the continuation) are serialised into `PendingChoice.context`
- Pipeline halts — no further actions are processed until the choice resolves
- `SubmitChoiceAction` arrives → selected option is filled into the continuation → continuation actions re-enqueued → `GamePhase` → `InProgress`

**A response may open a `PendingChoice` (G8, 2026-06-24).** A responder's invoked sigil whose effect Discovers / targets / picks a mode issues `StartChoiceAction` mid-response — an ordinary `PendingChoice`, the **responder's own** choice, **not** a nested intervention. The depth-1 cap bars only a further *player intervention window* on a response, so it does not bar this; `pendingChoice` and `pendingIntervention` coexist as orthogonal §1 fields (`phase → PendingChoice` with the parked intervention state retained). The choice resolves, the response continuation completes ("Response resolution"), then the intervention loop resumes. **Chained choices** (Discover an effect → pick its target — §1 "Chained choices") are each their own `PendingChoice`; with no early settle, a mortally-wounded subject the response is acting on stays on board across the whole chain, so a Discovered/targeted save can reach it.

### PendingIntervention Interruption

A **two-stage** window the engine opens for the **non-active player** at every ③′ and ⑥′ (§3 "The responder & window delivery") — **unconditional** (it opens even with an empty `candidates` offer), suppressed only by a prior `skip-all` this action, an exhausted budget (`turn.interventionsUsed == MaxInterventionsPerTurn`), or the responder's `autoSkipAll`. **Stage 1** (`phase = InterventionDecision`) takes `RespondInterventionAction` (skip / skip-all / intervene); pressing **intervene** (`turn.interventionsUsed++`) advances to **Stage 2** (`phase = InterventionExtension`), which takes one `SubmitInterventionAction` (a card + targets, or back out). Stage 2 is **set-valued** (offers *all* the responder's matching + affordable cards in `candidates`, each with its engine-computed targets — §1 `InterventionCandidate`) and the window **loops** by re-opening a fresh Stage-1 window after each play; it resolves one of two ways, by which phase the triggers hooked. (Throughout: the responder is the **non-active** player; the responder-directed `InterventionPromptEvent` + public `InterventionWindowOpenedEvent` go out (§2B; `stage: Decision`, then `stage: Extension` on escalation); the stage's timer starts (`InterventionDecisionWindow` / `InterventionExtensionWindow`); `RespondInterventionAction`/`SubmitInterventionAction` are accepted only from the responding player and **exempt** from the ③ turn-ownership check, a played card's cost paid from the responder's `mana` (their next-turn pool, pre-loaded at turn-end — Turn Lifecycle); a **play** re-opens a fresh decision window over the remaining offer, a **skip / skip-all / timeout** ends the loop; timeout = an injected skip, logged as an input per Item 13; `phase → InProgress` at the end.)

**Declaration-hold (pre-phase — `heldAction` present).** Opened at ③′ when hand triggers fire on `ActionDeclaredEvent`. The triggering action is suspended into `pendingIntervention.heldAction`.
- On **play**: the chosen card resolves first (full pipeline, paying `effectiveCost` from `mana`); it runs ordinary effects (summon, grant keyword, gain armor, …) and may `cancel` or `retarget` the held action. The held action then **resumes, re-validated (②③)**: if `cancel`led or now illegal (attacker dead/returned, target gone, forced to a Taunt) it **fizzles** — dropped, not executed (no `ActionRejection`; no client awaits it) — and the loop ends; if still legal it **re-declares (③′)**, re-opening the **set-valued** window over the responder's *remaining* matching + affordable cards (the **re-declare loop**), or, if none remain, proceeding to ④. The responder may thus chain several of their *own* held cards (each chained play is a fresh intervene-press, so the chain is bounded by **both** the per-turn press budget `MaxInterventionsPerTurn` **and** next-turn mana); a response is never intervened on *itself* (depth-1).
- On **skip / timeout**: the loop ends — the held action resumes, re-validates, and executes (④) or fizzles. The window does **not** re-open (a skip changes nothing).

**Post-reaction (post-phase — `heldAction` null).** Hand triggers that fire on regular events (including the cadence-deferred `MinionMortallyWoundedEvent` / `HeroMortallyWoundedEvent`) do **not** halt per-event. The matching events of **one action** are gathered and a **set-valued** window opens at **⑥′** — immediately after that action's ⑤/⑥ (see "Batching"), *not* deferred to the full-cascade settle. Because ⑦ runs only at the eventual settle, the mortally-wounded subject is still on the board when the window opens, so a save (heal a mortally-wounded minion or hero back above 0) lands before any death wave collects it. **Post-reaction windows never pre-settle deaths** — the subject must remain on board for the response to act on, and the wave simply does not collect a minion the response lifted back above 0. The response resolves immediately (see "Response resolution"); a **play re-offers** the remaining matching + affordable set (the **re-offer loop**); a **skip / timeout** ends the window and the cascade resumes. There is no held action to resume.

**Batching — two granularities: the offered card set, and each card's target set.** (1) *Across cards* — the window is **set-valued**: it offers every reactive hand card of the responder that matched **this action** and is affordable; the player plays one or skips, and a play re-offers the rest (above). (2) *Within a card* — a single AoE produces many sub-events (one selector-carrying `DealDamageAction` → many `DamageTakenEvent`s, one per target), so a card's offered **targets** are its matched subjects from this action **∩ its declared `(selector, cardinality)` family** (the family's `All…/Alive…/Dead…` choice decides whether a mortally-wounded match is offered — inclusive for a "save" card). The triggering unit is therefore **one action's events** — never one event, never the whole cascade. `SubmitInterventionAction` carries `{chosenCardId, targetIds[]}`, validated by the ordinary §4 ③ membership check (`chosenCardId ∈ candidates`, `targetIds ⊆` that candidate's `targetIds`).

- **AoE, one save card → one card in the offer, all N subjects as its targets.** An AoE damaging **7** friendly minions while the responder holds a single "heal one minion +2 on `DamageTakenEvent`" card → the window offers that one card over all 7 (the dying ones included); pick one to heal/save.
- **Two separate damage instances (two actions) → two windows.** A card dealing 1 twice via two `DealDamageAction`s opens a window after each; a single-use reactive may be spent on **either** — **skip** the first (which ends *that* window's loop but does **not** consume or discard the card — it stays in hand, trigger registered) to hold for the second, or spend immediately. Distinct instances, not the AoE case.
- **Multiple reactive cards on one action → one set-valued window**, re-offered on each play; the responder chooses order by which card they pick each pass. (This replaces the former "one window per matched card in hand order"; replay determinism comes from the logged `SubmitInterventionAction` choices, not an engine-imposed card order.)

Declaration-hold windows are likewise **set-valued + looped**: each declared action is a single `ActionDeclaredEvent`, so a "counter the spell" card hooks the one `ResolveCardAction` (the resolution — true Counterspell; a "prevent the cast" card hooks the earlier `PlayCardAction` instead, §2A #34), a "redirect the spell" card hooks whichever it targets, and a **pre-damage** card hooks the AoE's **single selector-carrying `DealDamageAction`** (§3 `ITargetSelector` auto-hit) — one window for the whole AoE either way, fired at ③′ *before* the hits rather than at ⑥′ *after* them, and re-declared after each play.

Only **player-choice windows** open this way; **auto reactions** (inscriptions, board/artifact triggers) fire inline per-event in FIFO during ⑤ (and during a response — the depth-1 cap suppresses only player windows). A deterministic reaction with no decision (e.g. "heal *it* for 2", no target choice) is the natural **inscription** (or board-hosted trigger), enqueuing one effect per match and opening no window at all.

---

## Unaddressed Features

Features the current architecture **deliberately does not support** and that are deferred **indefinitely** (no planned epic/ticket, unlike the "deferred to a future epic" items tracked in the borrow-list/plan). Each entry records the limitation, why it exists, and what enabling it would cost — so a future card requirement can reopen it with full context rather than rediscovering the constraint.

> **Note (keyword model collapse):** an earlier entry here — *"aura-granted active keywords"* — has been **removed**, not deferred. It existed because keywords were split into declarative (queried) and active (event-bus listeners), and an aura granting a listener-backed keyword would have had to subscribe during stage-⑥ recalc, violating the Item 3 invariant. The active/listener category was eliminated (§3 `IKeyword`): all keywords are declarative, pulled from the effective view at the minion's own action moments, with follow-on actions expressed as role-interface hooks. Aura-granted keywords now behave identically to intrinsic ones with no special mechanism, so the limitation no longer exists.

### A minion triggers off its own Battlecry (HS divergence)

A card-played minion's persistent `ITrigger`s are **live during its own Battlecry** (registered at summon, `ResolveCardAction` ④, before the Battlecry is enqueued — §2A / §3 trigger-timing note). So a minion whose own trigger watches its Battlecry's event type reacts to it — e.g. *"Battlecry: deal 1 to all minions. Whenever a minion is damaged, gain +1/+1"* buffs itself off its own Battlecry. **Hearthstone does the opposite** — a minion's triggered effects stay inactive until its Battlecry fully resolves, so an HS minion never triggers off its own Battlecry (verified against the HS Advanced Rulebook, 2026-06-21).

- **Why (deliberate):** honoring the HS rule would require a "summoned-but-triggers-dormant" sub-state — `OnSummon` deferred to a *post-Battlecry* epoch — which is the **only** place in the engine that would need one. Every other mechanic is uniform: keywords pulled not stored, all damage one channel, triggers live by board zone. The simplification (register at summon, enqueue the Battlecry after) preserves that uniformity and **deletes** a whole mechanism (the rejected sub-drain / `FinalizeSummon` + DFS machinery). The divergence is **narrow**: only a minion's own `ITrigger`s reacting to its own Battlecry — the own-summon-event exclusion (equal-epoch), keyword hooks (pulled), and Deathrattle (snapshot) are all unaffected, and the *other*-minion summon-reaction ordering stays HS-faithful.
- **Cost to honor the HS rule / the reversible path:** make `OnSummon` registration its own queued action (`FinalizeSummonAction`, no event, self-skips if the minion already left the board) and let the Battlecry sub-tree resolve **depth-first** (front-enqueue, generalizing the currently-death-only `EnqueueFront`) so the finalizer registers *after* the whole Battlecry cascade — giving the minion a `birthEpoch` past every Battlecry epoch. Purely **additive** and localized to the summon path; no other subsystem changes.
- **Design note:** the self-trigger is as much an **opportunity** as a footgun — intentional "synergizes with its own Battlecry" cards become expressible. v1's own card set should treat it as a known interaction (and keep it legible on the card) rather than author an overlapping Battlecry+trigger minion that silently *assumes* the HS rule.
- **Status:** deliberate v1 choice (2026-06-21). Revisit only if a card genuinely needs HS's "Battlecry before triggers" behavior — the path above slots in without disturbing anything else.

### Intervention decision-window real-time budget

The per-turn press cap (`MaxInterventionsPerTurn`) bounds the **Stage-2 extension** windows but **not** the **Stage-1 decision** windows: a non-active player who never sets `autoSkipAll` can let each unconditional decision window time out, adding up to `InterventionDecisionWindow` of real time per active-player action to the opponent's turn (charged to neither player's turn clock — it is paused while a window is open). This is **inherent to the structural-window + hidden-information design**: a no-card decision window cannot be auto-fast-resolved without re-leaking "this player holds nothing," defeating the unconditional window's whole purpose.

- **Why deferred:** the waste is bounded (one short window per action, finite cascades) and may be negligible — it depends on the average number of windows (actions) per turn.
- **Cost to enable:** a per-turn cumulative real-time budget on a player's base (decision) windows — once exhausted, the turn's remaining decision windows are elided (auto skip-all), parallel to the press cap on extensions. One counter + a `GameConstant`.
- **Status:** **not built in v1** (decided 2026-06-16). Revisit only if telemetry (avg windows/turn × `InterventionDecisionWindow`) shows the wasted real-time per turn crosses a threshold worth a fix.

### Inscription counterplay (no answer cards)

Inscriptions have **zero counterplay surface (#31).** No `ITargetSelector` reaches the `inscriptions` zone, no action destroys / reveals / steals an inscription, and `SilenceMinionAction` does not apply (an inscription is a card, not a minion). HS ships Flare / Eater-of-Secrets / Kezan-Mystic-style answers; v1 **cannot express one.**

- **Why:** the sigil model deliberately kept the zone opaque (identity hidden, existence public) and added no zone-reaching primitives — the minimum needed for the invoke/inscribe duality, nothing more.
- **Cost to enable:** an **`InscriptionSelector`** surface (a selector that ranges over a player's inscriptions) plus a **`DestroyInscriptionAction`** (and optionally reveal/steal variants) — both purely **additive**: new selector + new action, no change to the existing hosting/firing model.
- **Related cap interaction (#33):** because inscribing frees a hand slot and the zone caps at `GameConstants.MaxInscriptions` (3), a player *can* inscribe-lock — hold 3 inscriptions down with more copies of other sigils in hand, making the zone the sole pressure valve. This is fine (the cap is intentional) and only matters once `MaxInscriptions`/`HandCap` are tuned together; noted here so a future answer-card pass weighs it.
- **Status:** **not built in v1** — deliberate. Revisit when a card requires an inscription answer.

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

### Hero combat (RESOLVED 2026-06-11 — concept dropped)

The former "Freezing a hero" and "Hero armor" entries lived here. Both closed by the hero-kit decision (2026-06-11): **heroes never attack** — no `heroAttack`, no weapons (the `Weapon` card type is gone), no hero frozen state (freeze has nothing to deny a hero — `FreezeTargetAction` is minion-only); the hero's kit is the **artifact row** (§1). Hero **armor** was kept and landed: `PlayerState.armor`, `GainArmorAction`/`ArmorGainedEvent`, and the §4 ④ absorb slot (`DamageTakenEvent.armorAbsorbed`). Heroes remain attackable *defenders* and never retaliate — structurally, since retaliation is gated on the defender's attack (> 0) and a hero has none.
