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

**▶ BORROW-LIST PASS COMPLETE — all 13 items resolved (✅ below). RESUME AT THE END-OF-PASS PLAN RECONCILIATION.** Session of 2026-06-02 walked Items 12 and 13 and closed here. The spec is now fully refined (plus post-pass Fireplace-comparison amendments — see below); the next task is purely mechanical plan-side reconciliation, then implementation.

**▶▶ PENDING SPEC EDIT (carry to next session) — Item 12 Fork-A tightening.** The random-K targeting path (§3 `ITargetSelector`, "random-K" consumption mode) currently has the effect enqueue an action carrying a *frozen pool* (`EntityId[]`). Change it to **carry the `ITargetSelector` reference and evaluate it at stage ④** (then draw K), so selection stays pure *and* current and is co-located with the draw — avoids a staleness bug where a pooled entity dies between enqueue and the draw action's ④. Small edit: update the §3 `ITargetSelector` "random-K" row + the Item 12 decision note in the borrow-list file. Identified during the 2026-06-02 Fireplace comparison; deferred by user to next session.

**Post-pass amendments (Fireplace comparison, 2026-06-02):** the spec is no longer frozen at the 13-item state. A comparison against jleclanche/fireplace (open-source HS clone) is driving refinements on shared subsystems. Applied so far: **keyword-model collapse** — eliminated the active/declarative keyword split; all keywords declarative, pulled at the minion's own action moments via role-interface hooks (`IOnDealtDamage`), `IKeyword` loses `OnApplied`/`OnRemoved` (no bus subscriptions; bus = `ITrigger` only); dissolved both Unaddressed-Features walls. Right cut = keyword (own-action-moment) vs trigger (board-wide event). See the borrow-list note's "second follow-on amendment (2026-06-02)" under Item 8. The user may have further Fireplace-driven points.

**▶ Next task — Plan reconciliation** (full instructions under "Plan reconciliation — end-of-pass task" in the borrow-list note). Walk every `Plan impact:` list in the borrow-list note and apply the edits to the epic/ticket files under `docs/superpowers/plans/2026-05-27-ccg-game-logic/`, **creating new tickets/epics where flagged**. Known new work (consolidated in the note's reconciliation list):
- `StabilizationAbortReport` telemetry → new Epic 04 **T4.8** (Item 1)
- `ITriggerCondition` library → new Epic 08 **T8.10** (Item 2)
- **NEW EPIC: Bot Support** → `GetLegalActions` + bot logic (Item 5)
- `IRandom` foundation → Epic 01 (Item 6)
- Structured error codes / `SubmitResult` → Epic 01 validator (Item 7)
- Tribe/keyword 4-field model → data-model + auras tickets (Item 8 + follow-on)
- **`ITargetSelector` library → new Epic 08 ticket** beside T8.10 (Item 12); validator target-check becomes selector-membership; cards carry `(selector, cardinality)`
- Replay artifacts → Epic 16: keep event-log replay (client path) + add command/input-log canonical replay (Item 13); ruleset-version stamp flagged for Game Server spec
- **Keyword-model collapse → Epic 01 `IKeyword` + Epic 07 re-scope** (post-pass amendment 2026-06-02, from the Fireplace comparison): `IKeyword { KeywordId }` + role hooks (`IOnDealtDamage`), NO `OnApplied`/`OnRemoved`/subscriptions; Lifesteal/Poisonous = hooks, Windfury = declarative read, Enrage = stage-⑥ recompute, Freeze = turn sweep; aura grants accept any keyword; bus carries only `ITrigger`. Dissolved both Unaddressed-Features walls (aura-granted active keywords gone; dead-source reframed). Spec §3 `IKeyword`/`IEventBus`/Source-Attribution + Unaddressed Features amended.

Then re-verify the README index/outline/progress tracker and resume line, and implementation can begin at **Epic 01 / T1.1**. (Workflow note: reconciliation edits the *plan*, not the spec — the spec is done.)

**Discuss the borrow list from the old Unity project** — `docs/superpowers/notes/2026-05-28-old-project-borrow-list.md` contains 13 discrete spec-refinement candidates harvested from the old proof-of-concept. Each has: what's there, why it matters for our spec, recommendation, and open questions. The user picks one item at a time; each decision either amends the spec/plan or gets rejected, then the notes file records the outcome.

Items, ordered by impact (✅ = resolved this pass):
1. ✅ Stabilization loop iteration cap & wave events — ADOPTED (cap=16, abort-as-NoContest, wave markers, scenario-repro telemetry). Also produced a related amendment: Reborn keyword/charge split (`MinionOnBoard.rebornAvailable`). See borrow-list note for full decision.
2. ✅ Pre-built `ITriggerCondition` singletons — ADOPTED in spec §3 (2026-05-30): condition interface + single-condition library (parameterless singletons + parameterized factories) + `All`/`Any` combinators (`Not` deferred). Made multiplicity explicit (card = handler = N triggers via `definition.triggers`); clarified conditions *gate* while trigger `type` *routes* resolution (Battlecry synchronous in play pipeline, Deathrattle in death-wave Phase 2). JSON encoding left as `DefaultCardHandler` detail. See borrow-list note for full decision.
3. ✅ Snapshot triggers before processing (registration safety) — ADOPTED in spec (2026-05-31) as **per-event snapshot at publish + creation-epoch filter**: `GameEngine.currentActionEpoch` (incremented per action ④); subscribers stamped `birthEpoch`, events stamped `originEpoch`; `Publish` dispatches iff `birthEpoch < originEpoch` so a listener never reacts to the event from the action that created it (incl. its own `MinionSummonedEvent`). Chosen over per-batch threading (heavy: registry copy, contract leak, temporal coupling, boundary definition) — epoch is lighter + more robust + uniform across all entity-introducing events. Invariant added: all subscriptions happen inside action handlers. See borrow-list note for full decision.
4. ✅ Effect-op result chaining + Spell Damage +X — ADAPTED (2026-05-31). Split in two: **(A)** Spell Damage +X adopted as a read-only *pulled* `EffectContext.SpellDamageBonus` snapshotted once at cast; Spell Damage **demoted from active keyword to declarative** (no listener), magnitude in the keyword string (`spell_damage:X`), aura-granted via `grantedKeywords`; v1 sources = intrinsic + aura keyword. **(B)** `Results` ledger / op-chaining **rejected** — the epoch-filtered event bus already does outcome-chaining (consistent with deferred Item 10). Player-scoped modifiers ("+1 all spells this turn", "+3 next spell") **deferred to a new future Modifier System** (`docs/superpowers/notes/2026-05-31-modifier-system.md`) — same shape as **mana-cost reduction**, design once for both; no reserved field now. See borrow-list note for full decision + Plan impact.
5. ✅ `GetLegalActions(playerId)` API — ADOPTED (2026-05-31), seam now / enumerator deferred. **Locked in spec (§4 ③):** validation (②③) is a pure, side-effect-free, standalone-invokable predicate `(action, state) → Ok | Rejection` = single source of truth for legality; `Submit` and any "what's legal" query call the same predicate. **`GetLegalActions` itself deferred to a NEW future "Bot Support" epic** (lives in `CCG.GameLogic`, brute-force-then-filter, phase-aware, decoupled from Item 12) — bots are confirmed-coming, so deferred not rejected. See borrow-list note for full decision + Plan impact.
6. ✅ Deterministic `IRandom` interface — ADOPTED (2026-06-01) as **counter-based per-action reseed**: one match seed `GameState.rngSeed`, no long-lived PRNG; each action derives a fresh `IRandom` from `mix(rngSeed, currentActionEpoch)` (reuses Item 3's epoch, now moved into `GameState`). Makes any single action reproducible from `(seed, epoch, preActionState)` with no PRNG-state serialization → `StabilizationAbortReport` self-contained. Seed hidden by architecture (client never sees `GameState`); one-per-game (per-player moot). **Deck shuffle at init via Fisher-Yates over `IRandom`** — decklists stored, shuffled order regenerates. Replay = seed + decklists + player-action log (system actions/rolls/epoch regenerate). See borrow-list note for full decision + Plan impact (new Epic 01 `IRandom` ticket).
7. ✅ Structured error codes — ADOPTED (2026-06-01). `ActionRejectionCode` (closed enum) + `record ActionRejection(Code, string? Detail = null)`; `Detail` is free-form, **logs-only, non-contractual** (tests assert on `Code`, clients localize from `Code`). **Typed per-code payloads rejected as premature** (reconstructable from `(action, state)`; no consumer blocked; keeps upgrade path). **Delivery = return value only** — never a `GameEvent`/on the bus/in the log (rejected actions re-reject on replay). `Submit` now returns `SubmitResult = Accepted(events) | Rejected(rejection)`. Completes the Item 5 predicate's `Ok | Rejection`. See borrow-list note for full decision + Plan impact.
8. ✅ Card definition hooks — ADOPTED SELECTIVELY (2026-06-01) on the line "does logic read it, or only the client?" **Tribes → `[Flags] enum Tribe`** (not string[]: keywords are strings because they resolve to `IKeyword` behavior; a tribe is an inert label → closed designer taxonomy → enum; `[Flags]` gives cheap multi-tribe + granting via `|`/`&`). **Tribeless allowed** (`Tribe.None = 0`, no special-casing). **Granting** = three minion fields `intrinsicTribes`(survives Silence) / `grantedTribes`(Silence→None) / `auraTribes`(recomputed each pass) → `effectiveTribes` (OR). `AuraEffect` gains `GrantedTribes`. **Presentation (art/description/sfx) OUT of the library** — client resolves from Platform-API catalog by `definitionKey`; `rarity` stays (gameplay-relevant). See borrow-list note for full decision + Plan impact.
   - **Follow-on (2026-06-01): keywords expanded to the same 4-field model.** Triggered by a consistency question; exposed an Item 4 bug (aura `spell_damage` was routed into bounce-retained `grantedKeywords`). Now `MinionOnBoard`: `intrinsicKeywords`/`grantedKeywords`(permanent, bounce-retained)/`auraKeywords`(recomputed, declarative-only) → effective `keywords`. **Silence asymmetry vs tribes:** keywords Silence clears intrinsic+granted; tribes clears granted only. `AuraEffect` gained `GrantedKeywords`. Item 4 corrected (aura spell_damage → `auraKeywords`). **Active keywords can't be aura-granted** (would subscribe during ⑥, violating Item 3) → recorded in new **Unaddressed Features** spec section (registry for *indefinitely*-deferred features, distinct from future-epic deferrals).
9. ✅ Nested `EffectContext` source-attribution — ADOPTED as clarification (2026-06-01). Plumbing already existed (actions + key events carry `sourceId`; Item 2 conditions key off it). Four locks: (1) **`sourceId` = the entity whose effect this is, set by the enqueuer, never inherited from the upstream cause** — Yeti's deathrattle damage is sourced to Yeti, not the spell that killed it; trigger `OnFire` stamps host as source; `sourcePlayerId` = source's controller (death-snapshot if dead). (2) **`EffectContext.sourceCardId` → `sourceId`** (unified ref; old field couldn't express a minion source). (3) **Lifesteal listener corrected** — fires on damage *caused by* self (`evt.sourceId == me`), not "on self" (was backwards). (4) **Dead-source active keywords don't fire** — attribution (event data) ≠ keyword application (live listeners, gone at Phase 1); → **Unaddressed Features**. See borrow-list note for full decision + Plan impact.
10. ✅ Op `Results` ledger — **REJECTED for v1** (2026-06-01), both halves. Gameplay op-chaining → event bus (Item 4). Observability/tooling ledger → need served by deterministic replay-with-tracer (Item 6) + event log; not withheld, not blocked, trivially additive later → **not** an Unaddressed Feature. `IEffect.Execute` stays `void`. No spec change. (Established the disposition test: reject vs Unaddressed-Features vs future-epic-deferral.)
11. ✅ Game-engine builder pattern — **NOT A SPEC CONCERN** (2026-06-01); implementation detail, home = plan scenario builder T1.7 (production init = seed + setup actions per Item 6; tests = direct `GameState` construction). No spec change. Plan impact: T1.7 must now cover `rngSeed`/`currentActionEpoch`, seed injection, and the new tribe/keyword fields from Items 6–9.
12. ✅ Targeting strategy registry — ADOPTED (2026-06-02) as **one pure `ITargetSelector` primitive**, dual of `ITriggerCondition` (singletons + factories + `Filter` reusing conditions). Pure function of `GameState`, ordered by §4 ⑦ board order. **Three consumption modes** from one candidate set: auto-hit (AoE), random-K (pool-carrying action drawn at stage ④), player-choice (feeds existing `StartChoiceAction`/`PendingChoice`). **Player-target legality unified** into the same primitive — §4 ③ validity = `chosen ∈ selector.Select(ctx)` (+ cardinality); no separate `TargetRequirement` taxonomy. **RNG (Item-6 parked Q) → Fork A:** selectors are pure, do NOT get `IRandom`; random draw stays a stage-④ concern (Item-6 invariant verbatim). Note's `RequiresChoice` selector-return variant rejected (keeps compute-targets vs interrupt-for-input separate). Spec §3 amended. See borrow-list note for full decision + Plan impact (new Epic 08 `ITargetSelector` ticket).
13. ✅ Command-log vs event-log replay — ADOPTED (2026-06-02): confirms Item 6's model and **tightens the spec §3 Replay paragraph** with three gaps — (1) an *input* = any `Submit`-ed action **incl. timeout/forfeit/disconnect-injected** ones (external nondeterminism, must be logged; timer is a Game Server concern); (2) **ruleset-version pinning** (command-log replay valid only vs same engine + card-def version); (3) **command log = canonical / event log = client wire format + archive**, event-log derivable from command-log+seed but not vice versa; single total-order input stream (not per-player). Disposition: SPEC (tighten) + Plan-impact (Epic 16). See borrow-list note for full decision.

**All 13 borrow-list items are resolved.** The spec is fully refined. Remaining before implementation: the **end-of-pass plan reconciliation** (see resume point above + the borrow-list note). Then implementation begins at **Epic 01 / T1.1**.

**Spec-first workflow (2026-05-31):** the borrow-list pass amends the **spec only**; epic/ticket files are reconciled in one pass at the end. Each `✅ DECISION` in the borrow-list note carries a **Plan impact:** list (affected epics/tickets + flagged new tickets/epics) so the end reconciliation is mechanical. The end-of-pass reconciliation task (apply Plan-impact edits; **create new tickets/epics where flagged** — e.g. `StabilizationAbortReport` telemetry, `ITriggerCondition` library; update README index/outline/progress) is written up under "Plan reconciliation" in the borrow-list note. New epics/tickets may be created freely so each finalized amendment has a home.

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
