# Epic 14 — Inversion Mechanic

**Goal:** The signature custom mechanic. A card or minion can be **Inverted**: stats flip (Attack ↔ Health) and its trigger type can change (e.g. Battlecry → Deathrattle) using the `normal`/`inverted` sections of the card definition. Reversible, applies in hand and on board, and fires an On-Invert trigger.

**Depends on:** Epic 08 (triggers/handlers/definitions), Epic 04 (minions, computed stats), Epic 02 (cards in hand).

---

## T14.1 — Invert/UnInvert minions + stat flip

**Goal:** Toggle a board minion's inverted state, flipping its base stats.

**Files (create):** `Actions/InvertTargetAction.cs` (`{ required string TargetId; string? SourceId }`), `Actions/UnInvertTargetAction.cs`, `Handlers/InvertTargetHandler.cs`, `Handlers/UnInvertTargetHandler.cs`, `Events/StatusEvents.cs` add `MinionInvertedEvent { string MinionId; bool IsInverted }`. **Modify:** `MinionOnBoard` computed stats to account for inversion, `GameEngineFactory`. **Test:** `Scenarios/InversionScenarios.cs`.

**Stat flip model:** when `IsInverted`, the *base* stats swap: effective `BaseAttack`↔`BaseHealth` before enchantments/auras apply. Decide how `CurrentHealth` behaves on flip — recommend: recompute `MaxHealth` from swapped base; clamp `CurrentHealth` to new `MaxHealth` (so a damaged minion stays damaged proportionally? or reset? **pick the simplest defensible rule: clamp to new max, keep current if lower**) and document precisely. The engine infers minion-vs-card from `TargetId` (spec) — `InvertTargetAction` carries no type.

**Scenario tests:**
- *Flip stats:* 2/5 minion → invert → 5/2 (`Attack==5`, `MaxHealth==2`); `MinionInverted(true)`.
- *Reversible:* invert then un-invert → back to 2/5; `MinionInverted(false)`.
- *Multiple toggles:* invert ×3 → inverted; un-invert → normal (idempotent per direction).
- *With enchantments:* a +1/+1 enchanted 2/5 → invert → base swaps to 5/2 then +1/+1 → 6/3. Lock the order (swap base, then apply enchant/aura).

---

## T14.2 — Invert cards in hand

**Goal:** Inversion also applies to cards not yet played.

**Depends on:** T14.1.

**Files (modify):** `Card` (already has `IsInverted`), `InvertTargetHandler`/`UnInvertTargetHandler` (resolve `TargetId` to a card in hand → flip `IsInverted`; flip displayed stats for minion/weapon cards), `Events/StatusEvents.cs` add `CardInvertedEvent { string CardId; string PlayerId; bool IsInverted }`. **Test:** `InversionScenarios.cs`.

**Scenario tests:**
- *Invert in hand:* a 3/2 minion card in hand → invert → shows 2/3; `CardInverted(true)`.
- *Play inverted card:* play it → summoned minion enters already inverted (stats + trigger per inverted section, see T14.3).

---

## T14.3 — Trigger-type change on inversion

**Goal:** The inverted definition section can specify different behaviour (e.g. Battlecry becomes Deathrattle).

**Depends on:** T14.1/T14.2, Epic 08 handlers.

**Files (modify):** `DefaultCardHandler` (when the card/minion `IsInverted`, read `Definition.inverted` instead of `Definition.normal` for battlecry/deathrattle/effects). **Test:** `InversionScenarios.cs`.

**Scenario tests:**
- *Battlecry → Deathrattle:* a minion whose `normal` has a battlecry and `inverted` has a deathrattle → play inverted → no battlecry on play; on death the deathrattle fires.
- *Effect differs:* normal "deal 2", inverted "heal 2" → behaviour matches current inversion state at resolution time.
- *Invert after summon changes future triggers:* summon normal (battlecry fired), then invert on board → its death now uses inverted deathrattle (if defined). Decide whether already-fired battlecry is unaffected (yes) and document.

---

## T14.4 — On Invert trigger

**Goal:** React to a minion/card being inverted.

**Depends on:** T14.1, Epic 08 trigger infra.

**Files (create):** sample `ITrigger` on `MinionInvertedEvent` (and/or `CardInvertedEvent`). **Test:** `InversionScenarios.cs`.

**Scenario tests:**
- *On-invert fires:* "When this is inverted, draw a card" → invert it → draw; fires on each invert (define whether un-invert also fires — recommend only on becoming inverted, i.e. `IsInverted==true`).
- *Doesn't fire on un-invert:* un-invert → no draw (per chosen rule).

---

## Epic 14 done-when

- Inversion flips base stats (with locked enchant/aura ordering and current-health rule), is reversible, and works in hand and on board.
- Inverted cards/minions use the `inverted` definition section for triggers/effects, including trigger-type changes.
- On-Invert triggers fire per the documented becoming-inverted rule.
