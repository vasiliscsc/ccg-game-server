# Epic 11 — Spell Damage, Combo & Synergies

**Goal:** Cross-card synergy mechanics: Spell Damage +X (boosts spell damage), Combo (bonus if not the first card this turn), and the On Spell Cast trigger. Small epic, but it exercises mutating an effect's parameters from a third party and per-turn play counting.

**Depends on:** Epic 08 (effects/triggers), Epic 03 (spell play, `SpellCastEvent`), Epic 02 (`cardsPlayedThisTurn` — add if not present).

---

## T11.1 — Spell Damage +X

**Goal:** A minion that increases the owner's spell damage.

**Files (create):** `Keywords/SpellDamageKeyword.cs` (carries an amount — keyword string like `spelldamage:1`, or a separate `SpellDamageBonus` int on `MinionOnBoard`; recommend a dedicated `int SpellDamage` summed from minions rather than a string param). **Modify:** `DealDamageEffect`/spell resolution (when the damage source is a spell cast by a player, add that player's total board spell damage to the amount), `MinionOnBoard` (add `int SpellDamage` set from definition). **Test:** `Scenarios/SpellDamageScenarios.cs`.

**Mechanism:** compute `ownerSpellDamage = Σ owner.Board.Where(alive).SpellDamage`; spell-origin `DealDamage` effects add it. Distinguish spell-origin damage from combat/effect damage — pass an origin/source-type through `EffectContext` so only spell damage is boosted.

**Scenario tests:**
- *+1 boosts spell:* board has Spell Damage +1; cast Strike(2) at a 3/3 → 3 damage dealt; minion → 0, dies.
- *Stacks:* two +1 minions → +2.
- *Doesn't boost combat/hero-power:* a minion attack or hero-power ping is unaffected (unless spec says hero power counts — default: no).

**Notes:** decide whether hero powers benefit (Hearthstone: no by default). Document the chosen rule.

---

## T11.2 — Combo

**Goal:** Effects that are stronger if you've already played a card this turn.

**Depends on:** `cardsPlayedThisTurn` tracking.

**Files (modify):** `PlayerState` (ensure `int CardsPlayedThisTurn`, reset at turn start), `PlayCardHandler` (increment on each play; expose pre-increment count to the card so "combo active = count > 0"), `Events/EffectEvents.cs` add `ComboTriggeredEvent { string CardId; string PlayerId }`. Combo cards' `ICardHandler.OnPlay` chooses the combo branch when active. **Test:** `Scenarios/ComboScenarios.cs`.

**Scenario tests:**
- *Combo inactive (first card):* play combo card first → base effect; no `ComboTriggered`.
- *Combo active:* play any card, then the combo card → combo branch, `ComboTriggered`.
- *Resets next turn:* combo active state resets after turn passes.

**Notes:** define "combo active" as `CardsPlayedThisTurn > 0` at the moment the combo card begins resolving (so it doesn't count itself). Increment after the play resolves, or read count before incrementing — lock the order.

---

## T11.3 — On Spell Cast trigger

**Goal:** Minions/effects that react to a spell being cast.

**Files (create):** sample `ITrigger` on `SpellCastEvent`. **Test:** extend `SpellDamageScenarios.cs` or new `OnSpellCastScenarios.cs`.

**Scenario tests:**
- *Fires on any spell:* "Whenever you cast a spell, draw a card" → cast a spell → draw occurs; fires for owner's spells.
- *Whose spell:* decide if it reacts to enemy spells too (define per card; sample reacts to owner's only).
- *Ordering with spell damage:* a single spell that both triggers this and is boosted — confirm event ordering (`SpellCast` published before damage effect resolves).

---

## Epic 11 done-when

- Spell Damage +X boosts only spell-origin damage and stacks; combat/hero-power unaffected (documented).
- Combo branches correctly based on prior plays this turn and resets each turn.
- On Spell Cast triggers fire with verified ordering relative to spell resolution.
