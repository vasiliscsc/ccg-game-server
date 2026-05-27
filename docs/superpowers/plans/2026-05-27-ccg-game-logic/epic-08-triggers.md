# Epic 08 — Triggers & Card Handlers

**Goal:** The card-behaviour subsystem. Introduce `ICardHandler` + `ITrigger` + `DefaultCardHandler`, then implement the trigger catalog: Battlecry, Deathrattle (death-wave Phase 2), Reborn (Phase 3), Start/End of Turn, On Friendly Minion Death, On Damage Taken (+ Enrage). This completes the death wave (Phases 2 & 3) left as hooks in Epic 04.

**Depends on:** Epic 04 (death wave hooks, summon), Epic 07 (bus listener lifecycle), Epic 03 (definition-driven play; replaces the inline interpreter here).

---

## T8.1 — ICardHandler + ITrigger infra + DefaultCardHandler

**Goal:** Generalise card behaviour; retire the Epic 03 inline effect interpreter.

**Files (create):** `Interfaces/ICardHandler.cs` (`string HandlerKey`, `OnPlay(CardPlayContext, IEventBus, IActionQueue)`, `OnSummon(MinionOnBoard, IEventBus, IActionQueue)`, `OnDeath(MinionOnBoard, IEventBus)`), `Interfaces/ITrigger.cs` (`string TriggerType`, `Type ListensTo`, `bool ShouldFire(GameEvent, GameState)`, `OnFire(GameEvent, GameState, IActionQueue)`), `Interfaces/IEffect.cs` (`Execute(EffectContext, IActionQueue)`), `State/CardPlayContext.cs` + `State/EffectContext.cs`, `Cards/DefaultCardHandler.cs`, `Cards/CardHandlerRegistry.cs`, `Effects/EffectRegistry.cs`, `Effects/DealDamageEffect.cs`, `Effects/SummonMinionEffect.cs`. **Modify:** `PlayCardHandler` (delegate to `ICardHandler` resolved by `HandlerKey`, default = `DefaultCardHandler`), `SummonMinionHandler` (call `OnSummon`). **Test:** `Scenarios/CardHandlerScenarios.cs`.

**`DefaultCardHandler`:** reads `Definition.effects[]`, maps each `{type,...}` to an `IEffect` from `EffectRegistry`, executes (which enqueues actions). Targeting modes: `chosen` (uses `PlayCardAction.TargetId`), later `all-enemies`, `random`, etc. Port the Epic 03 damage + summon effects into `DealDamageEffect`/`SummonMinionEffect`.

**Scenario tests:** re-run Epic 03/04 damage-spell and summon scenarios through `DefaultCardHandler` (regression: behaviour identical, inline interpreter removed). Custom `handlerKey` resolves to a registered `ICardHandler`.

**Notes:** registries are injected (constructed in `GameEngineFactory`), enabling tests to register bespoke handlers/effects. Remove the `// TODO Epic 08` markers from Epic 03/04.

---

## T8.2 — Battlecry

**Goal:** "When you play this minion" effects.

**Files:** a battlecry `ICardHandler` (or `DefaultCardHandler` reading `definition.battlecry`), `Events/EffectEvents.cs` add `BattlecryTriggeredEvent { string MinionId }`. **Test:** `Scenarios/BattlecryScenarios.cs`.

**Mechanism:** on play of a minion with a battlecry, after `MinionSummonedEvent`, emit `BattlecryTriggeredEvent` and enqueue the battlecry's effect actions (honouring `PlayCardAction.TargetId` for targeted battlecries).

**Scenario tests:**
- *Damage battlecry:* play "Battlecry: deal 2 to a minion" targeting enemy 3/3 → 3/3 → 1 health; events `[...MinionSummoned, BattlecryTriggered, DamageTaken]`.
- *Summon battlecry:* battlecry summons a 1/1 → two minions on board.
- *Targeted battlecry validation:* requires a legal target if `definition` says target-required (validate at play).

---

## T8.3 — Deathrattle + death wave Phase 2

**Goal:** "When this dies" effects; implement Phase 2 of the wave.

**Depends on:** Epic 04 T4.5 wave loop, T8.1.

**Files (modify):** `DeathResolutionService` (Phase 2: for each dead minion in sort order, call its `ICardHandler.OnDeath`, which enqueues deathrattle actions; process the queue fully; **new deaths during Phase 2 deferred to next wave**), `Events/EffectEvents.cs` add `DeathrattleTriggeredEvent { string MinionId }`. **Test:** `Scenarios/DeathrattleScenarios.cs`.

**Mechanism:** `OnDeath` emits `DeathrattleTriggeredEvent` + enqueues effects. The wave's Phase 2 drains the queue (each action goes through the full pipeline incl. nested death checks deferred). Reborn handled in T8.5 Phase 3.

**Scenario tests:**
- *Deathrattle summons:* "Deathrattle: summon a 1/1" 2/2 dies → 1/1 appears; events `[MinionDied, DeathrattleTriggered, MinionSummoned]`.
- *Deathrattle deals damage that kills another → next wave:* DR deals 5 to an adjacent 2/2 → that death is a *second* wave (its own DR, if any, fires after).
- *Snapshot integrity:* DR reads the dead minion's snapshot stats correctly.

---

## T8.4 — Multiple-deathrattle batch ordering

**Goal:** Lock the spec rule: simultaneous deaths enqueue their deathrattles, processed as a batch in sort order.

**Depends on:** T8.3.

**Files:** test-only (`DeathrattleScenarios.cs`) + any fix needed in `DeathResolutionService`.

**Scenario tests:**
- *Batch order:* current board `[DR-A, DR-B]`, opponent `[DR-C]` all die at once → deathrattles fire A, B, C (each `DeathrattleTriggered` + its effects) before any second-wave deaths.
- *Interleaving guard:* a DR that would kill a still-living minion does not interrupt the current batch; that death is the next wave.

---

## T8.5 — Reborn + death wave Phase 3

**Goal:** Reborn re-summons at 1 HP after all deathrattles of the wave.

**Depends on:** T8.3.

**Files (create):** `Keywords/RebornKeyword.cs` (declarative marker — no listener). **Modify:** `DeathResolutionService` (Phase 3: for each dead minion of the wave with `reborn`, in sort order, enqueue `SummonMinionAction` with the card but `CurrentHealth = 1` and reborn removed; process; recalc auras), `Events` if a dedicated event is wanted (reuse `MinionSummonedEvent`). **Test:** `Scenarios/RebornScenarios.cs`.

**Scenario tests:**
- *Reborn returns at 1 HP:* Reborn 3/3 dies → re-summoned 3/1 without reborn; ordering: `MinionDied` (Phase1) → (no DR) → `MinionSummoned` (Phase3).
- *DR before Reborn:* a minion with both Deathrattle and Reborn → deathrattle fires (Phase 2) before the reborn summon (Phase 3).
- *Two reborns order:* two reborn minions die together → both re-summon in sort order, after all deathrattles.

**Notes:** `DeathResolutionService` owns Phase 3 — `ICardHandler.OnDeath` must **not** enqueue reborn (spec invariant). Verify by test that a handler doing so isn't required.

---

## T8.6 — Start-of-turn trigger

**Goal:** Triggers that fire at the active player's turn start.

**Files (create):** a sample start-of-turn `ITrigger` (via `ICardHandler.OnSummon` subscribing to `TurnStartedEvent` filtered to owner being active). **Test:** `Scenarios/TriggerOrderingScenarios.cs`.

**Mechanism:** minion's `OnSummon` registers an `ITrigger` on `TurnStartedEvent`; `ShouldFire` = `evt.ActivePlayerId == owner`; `OnFire` enqueues the effect. Fire order follows `IEventBus` (current player's minions by board index, then opponent's).

**Scenario tests:**
- *Fires on owner turn only:* "Start of turn: deal 1 to enemy hero" → fires when owner's turn starts, not opponent's.
- *Order by board index:* two such minions → fire in board order (lock the spec ordering rule here).

---

## T8.7 — End-of-turn trigger

**Goal:** Symmetric to T8.6 on `TurnEndedEvent`.

**Files:** sample end-of-turn `ITrigger`. **Test:** extend `TriggerOrderingScenarios.cs`.

**Scenario tests:** "End of turn: gain +1/+1" fires for the active player's minions at end of their turn, in board order; opponent's end-of-turn triggers (if any) fire after per bus ordering.

**Notes:** confirm end-of-turn triggers fire *before* the active-player swap (spec turn lifecycle step 2) so "current player first" ordering is well-defined.

---

## T8.8 — On Friendly Minion Death trigger

**Goal:** React to a friendly minion dying.

**Files:** sample `ITrigger` on `MinionDiedEvent` filtered to `snapshot.OwnerId == owner` and `MinionId != self`. **Test:** extend `TriggerOrderingScenarios.cs`.

**Scenario tests:**
- *Fires on ally death:* "When a friendly minion dies, gain +1 attack" → ally dies → buff applied; firing happens during the death wave (the `MinionDiedEvent` publish) and its enqueued buff resolves in-wave.
- *Not on enemy death:* enemy minion dying doesn't trigger it.
- *Not on self death:* the dying minion's own such trigger doesn't fire for itself.

**Notes:** careful interplay with the death wave — the trigger fires when `MinionDiedEvent` is published in Phase 1; its enqueued actions process within the wave. Test the resulting ordering explicitly.

---

## T8.9 — On Damage Taken trigger + Enrage

**Goal:** React to taking damage; Enrage as the canonical case.

**Files (create):** `Keywords/EnrageKeyword.cs` (or an `ITrigger`), `Events/StatusEvents.cs` add `EnrageStateChangedEvent { string MinionId; bool IsEnraged }`. **Modify:** damage/heal handlers to surface `IsDamaged` changes. **Test:** `Scenarios/EnrageScenarios.cs`.

**Mechanism:** subscribe to `DamageTakenEvent` (and `HealedEvent`) on self; when `IsDamaged` flips, apply/remove the enrage bonus (via an enchantment or aura-style bonus) and emit `EnrageStateChangedEvent`. Decide representation: recommend enrage as a temporary enchantment added when damaged, removed when healed to full.

**Scenario tests:**
- *Enrage on:* "Enrage: +3 attack" 3/5 takes 2 → attack becomes 6, `EnrageStateChanged(true)`.
- *Enrage off on heal:* heal back to full → attack 3, `EnrageStateChanged(false)`.
- *Generic on-damage trigger:* "When this takes damage, draw a card" → fires per damage instance.

**Notes:** if enrage is modeled via enchantment, Silence (Epic 16) clearing enchantments must also clear the enrage bonus — verify in Epic 16.

---

## Epic 08 done-when

- `ICardHandler`/`ITrigger`/`IEffect` infra is the single path for card behaviour; the Epic 03 inline interpreter is gone.
- Death wave is complete: Phase 1 (Epic 04) + Phase 2 deathrattles + Phase 3 reborn, with batch ordering and DR-before-Reborn invariant locked by tests.
- Start/End-of-turn, On-Friendly-Death, On-Damage-Taken/Enrage triggers work with verified bus ordering.
