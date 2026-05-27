# Epic 07 — Active Keywords

**Goal:** Keywords that *react* by registering event-bus listeners and enqueuing actions: Lifesteal, Poisonous, Windfury, Freeze. This is the first real exercise of the keyword→bus→queue loop and of subscriber lifecycle (register on summon, unregister on death/silence).

**Depends on:** Epic 06 (`IKeyword` infra, `OnApplied`/`OnRemoved`), Epic 05 (combat producing `DamageTakenEvent`), Epic 04 (death wave / `DestroyMinionAction`).

---

## T7.1 — Lifesteal

**Goal:** Damage dealt by this minion heals its owner's hero.

**Files (create):** `Keywords/LifestealKeyword.cs`, `Actions/HealAction.cs` (`{ required string TargetId; required int Amount; string? SourceId }`), `Handlers/HealHandler.cs`, `Events/CombatEvents.cs` add `HealedEvent { string TargetId; int Amount; string? SourceId }`. **Modify:** `GameEngineFactory` (register handler + keyword). **Test:** `Scenarios/LifestealScenarios.cs`.

**Mechanism:** `LifestealKeyword.OnApplied(minionId, bus, queue)` subscribes to `DamageTakenEvent` where `SourceId == minionId`; on fire, enqueue `HealAction(owner.HeroId, evt.Amount, minionId)`. `OnRemoved` unsubscribes (`bus.Unsubscribe(listenerId)` — listenerId scoped per keyword+minion, e.g. `$"lifesteal:{minionId}"`).

**`HealHandler`:** heal target hero/minion up to `MaxHealth` (heroes cap at starting max, or uncapped? decide — recommend heroes cap at 30 unless armor exists; minions cap at `MaxHealth`); set `IsDamaged` accordingly; emit `HealedEvent` with the *actual* amount healed (not requested).

**Scenario tests:**
- *Combat lifesteal:* Lifesteal 3/3 attacks a 2/4, owner hero at 20 → owner heals 3 → 23 (cap aware), `Healed(hero,3)`.
- *Overheal capped:* owner at 29 (max 30), lifesteal deals 5 → heals 1, `Healed(hero,1)`.
- *Removed on death:* lifesteal minion dies → its listener is gone (a later same-source damage event, if any, does not heal).

**Notes:** listener id convention `"<keyword>:<minionId>"` so `OnRemoved` can target precisely. Confirm `HealedEvent.Amount` = actual healed.

---

## T7.2 — Poisonous

**Goal:** Any damage this minion deals to another minion destroys it.

**Depends on:** T7.1 pattern, Epic 04 `DestroyMinionAction`.

**Files (create):** `Keywords/PoisonousKeyword.cs`. **Test:** `Scenarios/PoisonousScenarios.cs`.

**Mechanism:** subscribe to `DamageTakenEvent` where `SourceId == minionId` **and** target is a minion **and** `Amount > 0`; enqueue `DestroyMinionAction(targetMinionId, minionId)`. `OnRemoved` unsubscribes.

**Scenario tests:**
- *Poison kills big minion:* Poisonous 1/1 attacks 8/8 → 8/8 takes 1 then is destroyed via resolution → `MinionDied`. (Poison 1/1 also takes 8 and dies — both die in order.)
- *No damage, no kill:* poisonous minion deals 0 (0-attack) → no destroy.
- *Hero immune:* poisonous damage to a hero does not "destroy" (only minions).

**Notes:** destroy is enqueued, so it dies in the *next* death-wave pass — verify ordering relative to the poisonous minion's own combat death.

---

## T7.3 — Windfury

**Goal:** Two attacks per turn.

**Depends on:** Epic 05 attack counters.

**Files (create):** `Keywords/WindfuryKeyword.cs`. **Test:** `Scenarios/WindfuryScenarios.cs`.

**Mechanism:** simplest correct approach — `OnApplied` sets `minion.AttacksAllowedThisTurn = 2`; the turn-start reset (Epic 05 T5.6) must set `AttacksAllowedThisTurn = HasWindfury ? 2 : 1`. Update T5.6's reset to consult Windfury. (Listener-on-summon also works but turn-start needs to know regardless, so centralise in the reset.)

**Scenario tests:**
- *Two attacks:* Windfury 3/3 → attack twice in one turn (both resolve); third attack throws.
- *Resets next turn:* after two attacks, next turn it can attack twice again.
- *Gained mid-turn:* a minion granted Windfury after already attacking once this turn → can attack once more (total 2).

**Notes:** centralising `AttacksAllowedThisTurn` computation in the turn-start reset + on-apply keeps it consistent with future "grant Windfury" effects.

---

## T7.4 — Freeze

**Goal:** Freezing prevents attacking; frozen minions thaw at the owner's next turn end/start per the spec rule.

**Depends on:** Epic 05 (frozen blocks attack already validated), Epic 01 turn transition.

**Files (create):** `Actions/FreezeTargetAction.cs` (`{ required string TargetId; string? SourceId }`), `Handlers/FreezeTargetHandler.cs`, `Keywords/FreezeKeyword.cs` (the keyword that *applies* freeze when a minion with it deals damage — e.g. Frost attackers), `Events/StatusEvents.cs` add `MinionFrozenEvent { string MinionId }`. **Modify:** turn-start init (unfreeze logic). **Test:** `Scenarios/FreezeScenarios.cs`.

**Freeze semantics (Hearthstone-style):** a minion frozen during a turn unfreezes at the start of its owner's *next* turn **if it did not attack** the turn it was frozen / generally: track `FrozenSince` (turn number) and at the owner's turn start, if `IsFrozen` and it has been frozen through one of their turns, clear it. Simplest deterministic rule for us: **set `IsFrozen=true` + `FrozenOnTurn=Turn.Number`; at the owner's turn-start, if `IsFrozen` clear it** (one full opponent turn passes). Lock the exact rule at pickup and document; provide tests that pin it.

**`FreezeTargetHandler`:** set `IsFrozen = true`, `FrozenOnTurn = Turn.Number`, emit `MinionFrozenEvent`. **`FreezeKeyword`:** subscribe to `DamageTakenEvent` where `SourceId == minionId` & target is a minion → enqueue `FreezeTargetAction(targetMinionId)`.

**Scenario tests:**
- *Frozen can't attack:* freeze a minion → attack throws (validation from Epic 05).
- *Thaws next owner turn:* p1's minion frozen on p1's turn → end turn (p2) → end turn (p1 start) → minion unfrozen, can attack.
- *Freeze-on-hit:* Freeze-keyword minion attacks an enemy minion → enemy becomes frozen, `MinionFrozen`.
- *Freeze a hero:* `FreezeTargetAction` on a hero sets a frozen flag on the hero (heroes can be frozen → can't attack with weapon) — add `IsFrozen` to `PlayerState`; weapon attack check (Epic 10) consults it.

**Notes:** add `bool IsFrozen` + `int FrozenOnTurn` to `MinionOnBoard`; add `bool IsFrozen` to `PlayerState`. The single turn-start init method (Epic 05 T5.6) is where mana/draw/attack-reset/**unfreeze** all live, in documented order.

---

## Epic 07 done-when

- Lifesteal heals (cap-aware, actual amount), Poisonous destroys via the death wave, Windfury grants a second attack that resets each turn, Freeze blocks attacking and thaws on schedule.
- Keyword listeners register on summon and unregister on death (verified), proving the subscriber lifecycle.
- Turn-start init now also handles unfreeze and Windfury-aware attack-allowance.
