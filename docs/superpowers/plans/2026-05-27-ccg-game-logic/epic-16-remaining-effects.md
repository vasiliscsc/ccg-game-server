# Epic 16 — Remaining Effects

**Goal:** Fill in the remaining effect/action surface from the spec that earlier epics referenced but deferred: Silence, Transform, Return-to-hand (bounce), enchantment buffs, deck manipulation (shuffle/discard/give), mana modification, and the multi-step Card Crafting sequence. After this epic the action/event catalogs from spec §2 are fully implemented.

**Depends on:** Epic 08 (effects/handlers), Epic 09 (auras — Silence clears aura registrations), Epic 12 (PendingChoice — crafting is a multi-step choice), Epic 04 (minions/board).

---

## T16.1 — Silence

**Goal:** Remove all enchantments and keywords from a minion.

**Files (create):** `Actions/SilenceMinionAction.cs` (`{ required string MinionId; string? SourceId }`), `Handlers/SilenceMinionHandler.cs`, `Events/StatusEvents.cs` add `MinionSilencedEvent { string MinionId }`. **Test:** `Scenarios/SilenceScenarios.cs`.

**Handler:** clear `Enchantments`; for each keyword call `IKeyword.OnRemoved` (unsubscribing active-keyword listeners) then clear `Keywords`; unregister any triggers/auras the minion's `ICardHandler` registered (call a silence hook on the handler, or unsubscribe by the minion's listener-id prefix); recompute stats; emit `MinionSilencedEvent`. Aura bonuses already on *other* minions recompute away on the next aura pass once this minion's aura is unregistered.

**Scenario tests:**
- *Silence clears buff:* a 2/2 enchanted to 4/4 → silence → back to 2/2 base.
- *Silence removes keyword:* a Taunt/Divine Shield minion → silence → keyword gone (Taunt no longer constrains; DS won't absorb).
- *Silence kills via lost health enchant:* a 1/1 buffed to 1/3 and damaged to 1 (current 1, max 3) → silence → max back to 1, current clamps to 1, survives; but a 1/1 buffed to 1/5 and damaged to 2 health remaining (current 2 < max 5)... craft a case where removing the health enchant drops max below current damage so it dies. Lock this regression.
- *Silence removes registered aura:* silencing an aura source → dependents lose the bonus next recalc.
- *Silence removes deathrattle:* silenced minion's deathrattle does not fire on death.

**Notes:** Silence is the canonical "clear everything added" — verify it interacts correctly with Enrage (enchantment-based, Epic 08) and auras. This ticket is a major regression magnet; test broadly.

---

## T16.2 — Transform

**Goal:** Replace a minion with a different one in place.

**Files (create):** `Actions/TransformMinionAction.cs` (`{ required string MinionId; required string NewCardId; string? SourceId }`), `Handlers/TransformMinionHandler.cs`, `Events/BoardEvents.cs` add `MinionTransformedEvent { string MinionId; Card NewCard }`. **Test:** `Scenarios/TransformScenarios.cs`.

**Handler:** unregister old minion's keywords/triggers/auras (like silence); replace its identity/stats with the new card's fresh stats (new `MinionId`? or keep slot id? — decide: keep board *position*, assign new minion identity; transform is "becomes a different minion", not a buff); register new minion's behaviour (`OnSummon`-style); emit `MinionTransformedEvent`. Transformed minion typically has summoning sickness reset per rule — document.

**Scenario tests:**
- *Sheep transform:* a 7/7 with keywords → transform into a 1/1 vanilla → stats 1/1, keywords gone, no deathrattle from original.
- *Position preserved:* transform middle minion → stays in same board slot.
- *No deathrattle of original:* original's deathrattle does not fire (it didn't die, it transformed).

---

## T16.3 — Return to hand (bounce)

**Goal:** Move a board minion back to its owner's hand as a card, under one of two policies — `Stripped` (Sap-style, fresh card) or `RetainEnchantments` (Recall-style, stat enchantments and granted keywords carry over). See spec §1 "Minion → Card Transitions" for the full retain/strip table.

**Files (create):** `Actions/ReturnToHandAction.cs` (`{ required string MinionId; required MinionToCardPolicy Policy; string? SourceId }`), `State/Enums.cs` add `MinionToCardPolicy { Stripped, RetainEnchantments }`, `Handlers/ReturnToHandHandler.cs`, `Events/CardEvents.cs` add `CardReturnedToHandEvent { string PlayerId; Card Card }`. **Test:** `Scenarios/BounceScenarios.cs`.

**Handler:** remove minion from board (unregister behaviour); fabricate a new `Card` with a freshly allocated `Card.id` and `definitionKey` from the minion; copy `isInverted` from the minion. If `Policy == RetainEnchantments`: copy `enchantments` → `modifiers` (zero out `costDelta`, preserve attack/healthDelta) and copy `grantedKeywords` → `grantedKeywords`. If `Policy == Stripped`: leave `modifiers` and `grantedKeywords` empty. Damage, aura bonuses, runtime flags, and aura-granted keywords are always discarded. Add the fabricated Card to owner's hand (respect hand cap → if full, the minion is destroyed instead, emit appropriate events); emit `CardReturnedToHandEvent`.

**Scenario tests:**
- *Stripped bounce resets:* a buffed (+2/+2) and damaged (took 2 dmg) minion bounced with `Stripped` → returns as a fresh base Card in hand (`modifiers = []`, `grantedKeywords = []`); board slot empty.
- *RetainEnchantments preserves buffs:* a 4/5 Yeti enchanted to 6/7 by a `BuffMinionAction` and granted Taunt by a non-aura buff → bounce with `RetainEnchantments` → returns as a Card with `modifiers = [{+2/+2}]` and `grantedKeywords = ["Taunt"]`. Replay it → summons a 6/7 with Taunt at full health, fresh `minionId`.
- *Aura grants do not survive RetainEnchantments:* a minion benefiting from a friendly aura's +1/+1 and granted DivineShield by the same aura → bounce with `RetainEnchantments` → returns as a Card with `modifiers = []` and `grantedKeywords = []` (only the aura was the source; nothing in `enchantments` or `grantedKeywords` to copy).
- *isInverted preserved under both policies:* an inverted minion bounced with `Stripped` → Card.isInverted = true. Same minion bounced with `RetainEnchantments` → Card.isInverted = true.
- *Damage always discarded:* a buffed (+2/+2) minion at 1/7 (took 6 dmg, max 7) bounced with `RetainEnchantments` → returns as a Card whose replay summons a 6/7 at full health.
- *Full hand → destroyed:* hand full → bounce destroys the minion instead (define & test); applies under both policies.
- *No deathrattle on bounce:* deathrattle does not fire under either policy (not a death).

---

## T16.4 — Enchantment buffs

**Goal:** Permanent stat buffs via `StatModifier` enchantments (distinct from auras).

**Files (create):** `Actions/BuffMinionAction.cs` (`{ required string MinionId; int AttackDelta; int HealthDelta; required string SourceId }`), `Handlers/BuffMinionHandler.cs`, `Effects/BuffMinionEffect.cs`. **Modify:** `MinionOnBoard` computed stats already sum enchantments. **Test:** `Scenarios/BuffScenarios.cs`.

**Handler:** add a `StatModifier(SourceId, 0, AttackDelta, HealthDelta)` to `Enchantments`; health buff raises `MaxHealth` *and* `CurrentHealth` (buffs heal-as-they-raise? — Hearthstone: +health raises both current and max; document); attack buff raises attack. Emit `MinionStatsChangedEvent` (add to `StatusEvents`). Cleared by Silence (T16.1).

**Scenario tests:**
- *+2/+2:* 2/3 → buff +2/+2 → 4/5, current health 5.
- *Stacks:* two buffs accumulate.
- *Silence clears:* covered in T16.1 but re-verify here.

---

## T16.5 — Deck manipulation: shuffle / discard / give

**Goal:** The remaining card-movement actions.

**Files (create):** `Actions/ShuffleDeckAction.cs`, `Actions/DiscardCardAction.cs` (`cardId?` null = random), `Actions/GiveCardAction.cs` (`{ string PlayerId; string CardId; string? SourceId }`) + handlers, `Events/CardEvents.cs` add `CardDiscardedEvent`, `CardShuffledIntoDeckEvent`, `CardAddedToHandEvent`. **Modify:** uses `IRandom` (from Epic 15) for random discard/shuffle. **Test:** `Scenarios/DeckManipulationScenarios.cs`.

**Scenario tests:**
- *Give card:* `GiveCardAction` → `CardAddedToHandEvent`, hand grows (respect cap → overflow burn).
- *Discard specific & random:* discard a named card → `CardDiscarded`; random discard with seeded RNG → deterministic pick.
- *Shuffle into deck:* a card shuffled in → `CardShuffledIntoDeck`, deck size grows; with seeded RNG position is deterministic for tests.
- *ModifyMana:* add `Actions/ModifyManaAction.cs` (`{ string PlayerId; int Delta }`) + handler → adjusts mana/crystals, emits `ManaChanged`. Test gain/loss of a mana crystal.

**Notes:** fold `ModifyManaAction` in here (deferred from Epic 02). Cover hand-cap overflow on give.

---

## T16.6 — Card crafting

**Goal:** The multi-step interactive sequence that produces a custom card.

**Depends on:** Epic 12 (PendingChoice continuation), T16.5 (give card).

**Files (create):** `Effects/CraftCardEffect.cs` + a crafting flow built from chained `StartChoiceAction`s (`ChoiceType.Craft`). **Test:** `Scenarios/CraftingScenarios.cs`.

**Mechanism:** crafting = a sequence of `PendingChoice` steps (e.g. pick a body → pick a keyword → pick an effect), each resuming via `SubmitChoiceAction`, accumulating selections in `PendingChoice.Context`, finally constructing a `Card` and enqueuing `GiveCardAction`. Reuses the Epic 12 pause/resume machinery end-to-end across multiple steps.

**Scenario tests:**
- *Three-step craft:* start craft → choose step 1 → choose step 2 → choose step 3 → resulting custom card added to hand reflecting all selections; events show three `ChoiceStarted`/`ChoiceResolved` pairs then `CardAddedToHand`.
- *Abandon/invalid:* invalid selection at a step → rejected, phase stays `PendingChoice` at that step.

**Notes:** crafting is the most elaborate use of the choice system — it validates that multi-step continuations compose. Keep the produced-card schema simple (body + one keyword + one effect) for the first implementation; richer crafting is future work.

---

## Epic 16 done-when

- Silence, Transform, Bounce, enchantment buffs, deck manipulation (shuffle/discard/give), and mana modification all implemented with regression tests.
- Card crafting works as a multi-step `PendingChoice` chain producing a custom card.
- The spec §2 action/event catalogs are fully realised; the engine is feature-complete against the game-mechanics spec.

---

## Post-epic: integration & polish (optional backlog)

Not full epics, but worth tracking once the above lands:
- **Full-game scenario tests** — scripted multi-turn games asserting end-to-end event logs.
- **Determinism/RNG audit** — every random effect uses seedable `IRandom`; add seed to `GameState` for replay.
- **Event-log replay** — reconstruct `GameState` from the event stream (validates the "client renders events" design and supports spectators/late-join).
- **Performance pass** — aura recalculation and bus dispatch under large boards.
- **Fatigue** (if deferred in Epic 02 T2.6).
