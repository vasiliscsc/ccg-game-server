# CCG.GameLogic — Implementation Plan (Epics & Tickets)

> **Source spec:** `docs/superpowers/specs/2026-05-26-game-mechanics.md`
> **Target:** `CCG.GameLogic` — standalone C# class library, no framework deps. Authoritative game rules for the CCG. Referenced by the Game Server; optionally by the Unity/Godot client.

## How this plan is structured

The project is too large for a single plan file, so it is broken into **epics** (one file each) containing **tickets**. Each ticket is a small, independently testable increment that builds toward a working engine. Tickets are picked up one at a time and implemented collaboratively.

**Ticket format** (in each epic file): goal, dependencies, files touched, in/out scope, scenario tests (Given/When/Then over initial state → action → expected events + state), and implementation notes. Tickets are intentionally *not* full pre-written code — code is produced when the ticket is picked up.

## Tech stack & conventions

- **.NET 10** (current LTS), C# records for `GameAction`/`GameEvent` (immutable), classes for state types (mutable).
- **Testing:** xUnit + FluentAssertions. Every feature ships with **scenario tests**: build an initial `GameState`, submit an action (or sequence), assert the ordered list of published events *and* the resulting state. This gives regression coverage for free — future bugs become new scenarios.
- **State mutation** happens only inside `IActionHandler` implementations.
- **Effects/triggers/keywords** enqueue actions; they never mutate state directly.
- `GameEngine.Submit(action)` returns `IReadOnlyList<GameEvent>` (all events from the action and its cascade) and the `EventBus` is independently inspectable.

## Project layout (target)

```
CCG.sln
CCG.GameLogic/            # the library
  State/  Actions/  Events/  Interfaces/
  Engine/ Infrastructure/ Handlers/
  Keywords/ Effects/ Cards/
CCG.GameLogic.Tests/      # xUnit scenario tests
  Helpers/  Scenarios/
```

---

## Epic index

| # | Epic | Status | Tickets |
|---|------|--------|---------|
| 01 | [Foundation & Core Engine](epic-01-foundation.md) | ✅ written | T1.1–T1.8 |
| 02 | [Mana & Card Flow](epic-02-mana-and-card-flow.md) | ✅ written | T2.1–T2.6 |
| 03 | [Spells & Win Condition](epic-03-spells-and-win.md) | ✅ written | T3.1–T3.6 |
| 04 | [Minions & Death Resolution](epic-04-minions.md) | ✅ written | T4.1–T4.7 |
| 05 | [Combat](epic-05-combat.md) | ✅ written | T5.1–T5.6 |
| 06 | [Declarative Keywords](epic-06-declarative-keywords.md) | ✅ written | T6.1–T6.5 |
| 07 | [Active Keywords](epic-07-active-keywords.md) | ✅ written | T7.1–T7.4 |
| 08 | [Triggers & Card Handlers](epic-08-triggers.md) | ✅ written | T8.1–T8.9 |
| 09 | [Auras](epic-09-auras.md) | ✅ written | T9.1–T9.4 |
| 10 | [Hero Power & Weapons](epic-10-hero-power-weapons.md) | ✅ written | T10.1–T10.6 |
| 11 | [Spell Damage, Combo & Synergies](epic-11-synergies.md) | ✅ written | T11.1–T11.3 |
| 12 | [Mulligan & Choices](epic-12-mulligan-choices.md) | ✅ written | T12.1–T12.6 |
| 13 | [Intervention System](epic-13-intervention.md) | ✅ written | T13.1–T13.3 |
| 14 | [Inversion Mechanic](epic-14-inversion.md) | ⛔ deferred to v2 | — (parked 2026-06-14; see `notes/2026-06-14-inversion-v2.md`) |
| 15 | [Neutral Zone](epic-15-neutral-zone.md) | ✅ written | T15.1–T15.4 |
| 16 | [Remaining Effects](epic-16-remaining-effects.md) | ✅ written | T16.1–T16.6 |

**Legend:** ✅ written · 🚧 in progress · ⬜ not written

---

## Full ticket outline

This captures the intended breakdown for *every* epic, so the structure survives even if detailed epic files are written incrementally. Detailed files expand each ticket with tests and notes.

### Epic 01 — Foundation & Core Engine
- **T1.1** Solution & project scaffold (.NET 10 sln, lib + test projects, xUnit + FluentAssertions, green build)
- **T1.2** Minimal data model — `GameState`, `PlayerState`, `TurnState`, `GamePhase`
- **T1.3** Action/Event base types — `GameAction`, `GameEvent`, `EndTurnAction`, `TurnStartedEvent`, `TurnEndedEvent`
- **T1.4** Core interfaces — `IActionHandler<T>`, `IEventBus`, `IActionQueue`
- **T1.5** `EventBus` — inspectable, publish ordering (current player first, then board index at publish time)
- **T1.6** `ActionQueue` — FIFO + `EnqueueFront`
- **T1.7** `ScenarioBuilder` + event-assertion helpers (`ShouldMatchInOrder`)
- **T1.8** `GameEngine` pipeline + `PhaseGuard` + `ActionValidator` + `EndTurnHandler` + scenario tests

### Epic 02 — Mana & Card Flow
- **T2.1** Mana fields on `PlayerState` + `ManaChangedEvent`
- **T2.2** Turn-start init: mana increment (cap 10) + restore (extend turn transition)
- **T2.3** Deck & hand on `PlayerState` + minimal `Card` type
- **T2.4** `DrawCardAction` + `CardDrawnEvent` + draw-on-turn-start
- **T2.5** Hand overflow / burn → `HandOverflowEvent`
- **T2.6** Empty-deck draw policy (no-op vs fatigue — decide & test)

### Epic 03 — Spells & Win Condition
- **T3.1** `Card` with `CardType`/`effectiveCost`; `PlayCardAction`
- **T3.2** `PlayCardHandler` — mana deduction, remove from hand, `CardPlayedEvent` + `SpellCastEvent`
- **T3.3** `DealDamageAction` + `DealDamageHandler` (hero target) + `DamageTakenEvent`
- **T3.4** First concrete damage spell wired via card definition
- **T3.5** Win-condition check (hero ≤ 0 → `GameEndedEvent`, phase `Ended`)
- **T3.6** `SurrenderAction`

### Epic 04 — Minions & Death Resolution
- **T4.1** `MinionOnBoard` type + `board` on `PlayerState`
- **T4.2** `SummonMinionAction` + handler + `MinionSummonedEvent` + board cap (7)
- **T4.3** A summon-minion spell
- **T4.4** Spell targeting a minion (`DealDamage` to minion) + target validation
- **T4.5** `DeathResolutionService` Phase 1 (remove, graveyard, `MinionDiedEvent`) + `GraveyardEntry`
- **T4.6** Death sort order across boards (current L→R, opponent L→R, neutral)
- **T4.7** `DestroyMinionAction` (instant kill)

### Epic 05 — Combat
- **T5.1** `AttackAction` + `AttackDeclaredEvent`
- **T5.2** `AttackHandler` — mutual damage, `AttackResolvedEvent`, `attacksUsedThisTurn`
- **T5.3** Minion attacking hero
- **T5.4** Attack validation (canAttack, frozen, attacks remaining, summoning sickness)
- **T5.5** Simultaneous death from combat
- **T5.6** `attacksAllowedThisTurn` field + reset on turn start

### Epic 06 — Declarative Keywords
- **T6.1** `IKeyword` infra + resolution + registration hook on summon
- **T6.2** Taunt (attack validation)
- **T6.3** Divine Shield (damage handler + `DivineShieldBrokenEvent`)
- **T6.4** Charge & Rush (canAttack-on-summon semantics)
- **T6.5** Stealth (targeting validation + break on attack)

### Epic 07 — Active Keywords
- **T7.1** Lifesteal (`DamageTakenEvent` → `HealAction`)
- **T7.2** Poisonous (`DamageTakenEvent` → `DestroyMinionAction`)
- **T7.3** Windfury (`attacksAllowedThisTurn = 2`)
- **T7.4** Freeze (`FreezeTargetAction` + `isFrozen` + end-of-turn unfreeze)

### Epic 08 — Triggers & Card Handlers
- **T8.1** `ICardHandler` + `ITrigger` infra + `DefaultCardHandler`
- **T8.2** Battlecry (`OnPlay`)
- **T8.3** Deathrattle (`OnDeath` → Phase 2) + `DeathrattleTriggeredEvent`
- **T8.4** Multiple-deathrattle batch ordering
- **T8.5** Reborn (Phase 3) + reborn marker keyword
- **T8.6** Start-of-turn trigger
- **T8.7** End-of-turn trigger
- **T8.8** On Friendly Minion Death trigger
- **T8.9** On Damage Taken trigger + Enrage + `EnrageStateChangedEvent`

### Epic 09 — Auras
- **T9.1** `IAura` infra + `AuraService` recalculation pass
- **T9.2** Basic stat aura (buff friendly minions)
- **T9.3** Recalculation triggers wired into pipeline (after board-affecting actions)
- **T9.4** Aura death (minion dies when its aura source is removed)

### Epic 10 — Hero Power & Weapons
- **T10.1** `HeroPower` + `UseHeroPowerAction` + `HeroPowerUsedEvent` + `heroPowerUsedThisTurn`
- **T10.2** Hero-power reset on turn start
- **T10.3** `WeaponOnHero` + `EquipWeaponAction` + `WeaponEquippedEvent` + `heroAttack`
- **T10.4** Hero attacking with weapon, durability loss, `WeaponDurabilityLostEvent`
- **T10.5** Weapon destroy (durability 0 / `DestroyWeaponAction`) + `WeaponDestroyedEvent`
- **T10.6** Inspire trigger (fires on hero-power use)

### Epic 11 — Spell Damage, Combo & Synergies
- **T11.1** Spell Damage +X keyword (bumps spell damage)
- **T11.2** Combo (`cardsPlayedThisTurn` + `ComboTriggeredEvent`)
- **T11.3** On Spell Cast trigger

### Epic 12 — Mulligan & Choices
- **T12.1** Mulligan phase + `MulliganStartedEvent` (per player) + `MulliganState`
- **T12.2** `SubmitMulliganAction` + replacement + `MulliganCompletedEvent` + → `InProgress`
- **T12.3** `PendingChoice` infra + `StartChoiceAction` + `ChoiceStartedEvent`
- **T12.4** `SubmitChoiceAction` + continuation resume + `ChoiceResolvedEvent`
- **T12.5** Discover effect (3 options, pick 1)
- **T12.6** Mid-effect targeting

### Epic 13 — Intervention System
- **T13.1** `PendingIntervention` + `StartInterventionAction` + `InterventionWindowOpenedEvent` + held action
- **T13.2** `SubmitInterventionAction` (play card → process card then held action) + `InterventionPlayedEvent`
- **T13.3** Skip / timeout path + `InterventionWindowClosedEvent`

### Epic 14 — Inversion Mechanic ⛔ DEFERRED TO V2 (parked 2026-06-14)
Builds nothing in v1; full v2 design seed in `notes/2026-06-14-inversion-v2.md` + the retained `epic-14-inversion.md`.
- ~~**T14.1** Invert/UnInvert on minions + stat flip~~ (v2)
- ~~**T14.2** Inversion on cards in hand~~ (v2)
- ~~**T14.3** Trigger-type change on inversion~~ (v2)
- ~~**T14.4** On Invert trigger~~ (v2)

### Epic 15 — Neutral Zone
- **T15.1** `NeutralZoneConfig` + `neutralZone` + `SpawnNeutralMinionAction` + `NeutralMinionSpawnedEvent`
- **T15.2** Attacking / commanding neutral minions
- **T15.3** Taunt ignored for neutral + player auras don't apply to neutral
- **T15.4** Repopulation on turn start + `NeutralZoneRepopulatedEvent`

### Epic 16 — Remaining Effects
- **T16.1** Silence (clear enchantments + keywords) + `MinionSilencedEvent`
- **T16.2** `TransformMinionAction` + `MinionTransformedEvent`
- **T16.3** `ReturnToHandAction` (bounce) + `CardReturnedToHandEvent`
- **T16.4** Buff via enchantments + `StatModifier`
- **T16.5** `ShuffleDeckAction` / `DiscardCardAction` / `GiveCardAction`
- **T16.6** Card crafting (multi-step interactive sequence)

---

## Writing progress

> Tracks how far the **detailed epic files** have been written, so this can be resumed across sessions. Update the status column in the Epic index table above to match.

- [x] README / index (this file)
- [x] Epic 01 detail
- [x] Epic 02 detail
- [x] Epic 03 detail
- [x] Epic 04 detail
- [x] Epic 05 detail
- [x] Epic 06 detail
- [x] Epic 07 detail
- [x] Epic 08 detail
- [x] Epic 09 detail
- [x] Epic 10 detail
- [x] Epic 11 detail
- [x] Epic 12 detail
- [x] Epic 13 detail
- [x] Epic 14 detail
- [x] Epic 15 detail
- [x] Epic 16 detail

**Resume point:** ✅ All 16 epic detail files written. Plan is complete — ready to implement, starting with Epic 01 / T1.1.
