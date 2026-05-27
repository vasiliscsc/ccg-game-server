# Epic 02 — Mana & Card Flow

**Goal:** Add the resource and card-movement substrate that spells/minions depend on: a mana pool that grows and refills each turn, a deck and hand, drawing (including on turn start), and the hand-overflow burn rule. No cards *do* anything yet — this epic is purely about resources and zones.

**Outcome demo:** Starting a new turn increments the active player's max mana (capped at 10), refills current mana, and draws the top card of their deck into hand. Drawing with a full hand burns the card.

**Depends on:** Epic 01 (engine pipeline, turn transition).

---

## T2.1 — Mana fields + ManaChangedEvent

**Goal:** Represent mana on players and a fact for changes.

**Files (modify):** `State/PlayerState.cs` (add `Mana`, `MaxMana`, both default 0). **Create:** `Events/ManaEvents.cs` (`ManaChangedEvent { required string PlayerId; required int Mana; required int MaxMana }`). **Modify:** `Helpers/ScenarioBuilder.cs` (add `WithMana(playerId, mana, maxMana)`).

**Scope out:** spending mana (Epic 03), mana-crystal effects (Epic 16 `ModifyManaAction`).

**Verify:** builds; `ScenarioBuilder.WithMana` covered indirectly by T2.2 tests. Commit.

---

## T2.2 — Turn-start mana increment & restore

**Goal:** Extend the turn transition so the new active player gains/refills mana.

**Depends on:** T2.1, Epic 01 `EndTurnHandler`.

**Files (modify):** `Handlers/EndTurnHandler.cs` (after flipping active player, for the **new** active player: `MaxMana = Min(10, MaxMana + 1)`, `Mana = MaxMana`, emit `ManaChangedEvent`). **Test:** `Scenarios/ManaScenarios.cs`.

**Design note — event order:** within the turn transition the event sequence becomes `[TurnEnded, TurnStarted, ManaChanged]`. Confirm `TurnStarted` precedes `ManaChanged` (turn context first, then resources). Document this ordering in the handler.

**Scenario tests:**
- *First end-turn grants 1 crystal:* p1 (max 1) ends turn → p2 max becomes 1, mana 1, `ManaChanged(p2,1,1)` present after `TurnStarted`.
- *Cap at 10:* p2 at max 10 → ends back to them later → stays 10.
- *Refill:* p2 max 5, mana 0 → their turn starts → mana 5.

**Notes:** The very first turn's mana is normally set at game start (Epic 12 mulligan / game init). For now `ScenarioBuilder` seeds starting mana; the handler only covers transitions. Note this seam so Epic 12 wires real game-start mana.

---

## T2.3 — Deck & hand + minimal Card

**Goal:** Add card zones and a minimal card type to move between them.

**Depends on:** Epic 01.

**Files (modify):** `State/PlayerState.cs` (add `List<Card> Hand`, `List<Card> Deck`, both default empty). **Create:** `State/Card.cs`. **Modify:** `State/Enums.cs` (add `CardType { Minion, Spell, Weapon, HeroPower }`, `CardRarity { Common, Rare, Epic, Legendary }`). **Modify:** `ScenarioBuilder` (`WithCardInHand(playerId, Card)`, `WithDeck(playerId, params Card[])`).

**`Card` minimal shape (grows in Epic 03/04):** `Id`, `Name`, `CardType Type`, `CardRarity Rarity`, `int BaseManaCost`. Add `EffectiveCost => Max(0, BaseManaCost + Σ modifiers)` later; for now `EffectiveCost = BaseManaCost`.

**Scope out:** `definition` JSON, `handlerKey`, stat fields, modifiers — added when spells (Epic 03) and minions (Epic 04) need them.

**Verify:** builds. Commit. (Behaviour tested via T2.4.)

**Notes:** Decide max hand size constant now (`GameConstants.MaxHandSize = 10`, `MaxDeckSize` etc. in a `State/GameConstants.cs`). T2.5 uses it.

---

## T2.4 — DrawCardAction + draw on turn start

**Goal:** Implement drawing as a first-class action, then trigger it at turn start.

**Depends on:** T2.3, T2.2.

**Files (create):** `Actions/DrawCardAction.cs` (`{ required string PlayerId; string? SourceId }`), `Events/CardEvents.cs` (`CardDrawnEvent { required string PlayerId; required Card Card }`), `Handlers/DrawCardHandler.cs`. **Modify:** `GameEngineFactory` (register handler), `EndTurnHandler` (enqueue `DrawCardAction` for new active player), **Test:** `Scenarios/CardDrawScenarios.cs`.

**`DrawCardHandler`:** take top of `Deck` (index 0), move to `Hand`, emit `CardDrawnEvent`. Empty-deck and overflow handled in T2.5/T2.6.

**Design note:** turn-start now enqueues `DrawCardAction` rather than drawing inline — this exercises the action queue cascade (a handler enqueuing another action) for the first time. Verify the returned event list from `Submit(EndTurn)` includes the draw event in correct position: `[TurnEnded, TurnStarted, ManaChanged, CardDrawn]`.

**Scenario tests:**
- *Direct draw:* p1 deck `[A,B]`, hand empty → `Submit(DrawCardAction p1)` → hand `[A]`, deck `[B]`, event `CardDrawn(p1,A)`.
- *Draw on turn start:* p1 ends turn, p2 deck `[X]` → events include `CardDrawn(p2,X)` after `ManaChanged`; p2 hand has X.

---

## T2.5 — Hand overflow (burn)

**Goal:** Drawing into a full hand discards the drawn card.

**Depends on:** T2.4.

**Files (modify):** `Events/CardEvents.cs` (add `HandOverflowEvent { required string PlayerId; required Card BurnedCard }`), `Handlers/DrawCardHandler.cs`. **Test:** extend `CardDrawScenarios.cs`.

**Behaviour:** if `Hand.Count >= GameConstants.MaxHandSize`, remove top of deck, do **not** add to hand, emit `HandOverflowEvent` (no `CardDrawnEvent`).

**Scenario tests:**
- *Burn at full hand:* hand has 10, deck `[A]` → draw → hand still 10, deck empty, event `HandOverflow(p,A)`, no `CardDrawn`.
- *Draw when one slot free:* hand 9, deck `[A]` → draw → hand 10 with A, `CardDrawn`.

---

## T2.6 — Empty-deck draw policy

**Goal:** Define and test what happens drawing from an empty deck.

**Depends on:** T2.4.

**Decision required at pickup:** the spec does not mandate fatigue. Two options — pick one and record it in this file before implementing:
- **(a) No-op:** drawing from empty deck emits nothing (or a `DeckEmptyEvent`), no damage.
- **(b) Fatigue:** emit escalating self-damage (`DealDamageAction` to own hero, increasing each empty draw) — requires `fatigueCounter` on `PlayerState` and depends on damage (Epic 03). If chosen, **defer this ticket until after Epic 03**.

**Recommendation:** ship **(a) no-op + `DeckEmptyEvent`** now to keep Epic 02 self-contained; revisit fatigue as an optional ticket after Epic 03 if desired.

**Files (if option a):** `Events/CardEvents.cs` (`DeckEmptyEvent { required string PlayerId }`), modify `DrawCardHandler`. **Test:** extend `CardDrawScenarios.cs`.

**Scenario tests (option a):** empty deck → draw → hand unchanged, event `DeckEmpty(p)`, no `CardDrawn`.

---

## Epic 02 done-when

- Turn start grants/refills mana (capped) and draws, in the verified event order `[TurnEnded, TurnStarted, ManaChanged, CardDrawn]`.
- Full-hand draw burns; empty-deck draw follows the chosen policy.
- `ScenarioBuilder` now supports `WithMana`, `WithCardInHand`, `WithDeck`.
- All new scenario tests green.
