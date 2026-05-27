# Epic 05 — Combat

**Goal:** Minions and heroes attack. Implement `AttackAction` with mutual damage, attack-eligibility rules (summoning sickness, one attack per turn, frozen), and per-turn attack reset. Death from combat flows through the Epic 04 resolution wave.

**Outcome demo:** A 3/2 attacks a 2/3 — attacker deals 3 (defender dies), defender deals 2 back (attacker survives at... wait, 2 health - 2 = 0, also dies). Both die simultaneously and resolve in order. Attacker can't attack twice the same turn.

**Depends on:** Epic 04 (minions, damage, death wave, `ResolveTarget`).

---

## T5.1 — AttackAction + AttackDeclaredEvent

**Goal:** The attack command and the pre-resolution announcement (the hook future interventions/triggers attach to).

**Files (create):** `Actions/AttackAction.cs` (`{ required string AttackerId; required string TargetId }` — no type discriminators per spec; engine infers via `ResolveTarget`), `Events/CombatEvents.cs` add `AttackDeclaredEvent { string AttackerId; string TargetId }`. 

**Notes:** `SourcePlayerId` on the action identifies who declared it. No handler yet — T5.2 adds it. Emitting `AttackDeclared` *before* damage is what lets Epic 13 interventions and Epic 08 triggers react. Commit with the type only, or fold into T5.2.

---

## T5.2 — AttackHandler (minion vs minion)

**Goal:** Resolve mutual combat damage and mark attacker as having attacked.

**Depends on:** T5.1, Epic 04 death wave.

**Files (create):** `Handlers/AttackHandler.cs`, `Events/CombatEvents.cs` add `AttackResolvedEvent { string AttackerId; string TargetId; int DamageToTarget; int DamageToAttacker }`. **Modify:** `GameEngineFactory`. **Test:** `Scenarios/CombatScenarios.cs`.

**Handler flow:** emit `AttackDeclaredEvent`; resolve attacker & target via `ResolveTarget`; enqueue `DealDamageAction(target, attacker.Attack, attacker.MinionId)` and `DealDamageAction(attacker, target.Attack, target.MinionId)` (both via the queue so keywords like Lifesteal/Poisonous in Epic 07 hook the resulting `DamageTakenEvent`s); increment `attacker.AttacksUsedThisTurn`; emit `AttackResolvedEvent`. Death resolution (post-drain) handles any deaths.

**Design note — damage simultaneity & event order:** both damage actions are enqueued before either resolves; target damage first, then attacker damage. Confirm event sequence: `AttackDeclared → DamageTaken(target) → DamageTaken(attacker) → AttackResolved`, then `MinionDied`s in death-order. Document; this order matters for regression tests.

**Scenario tests:**
- *Both survive:* 3/4 attacks 2/5 → target 5→3, attacker 4→2, both alive; events as above with no deaths.
- *Defender dies:* 3/2 attacks 2/1 → target dies, attacker takes 2 (→0) also dies → two `MinionDied` in board order.
- *Attacker marked:* after one attack `AttacksUsedThisTurn == 1`.

---

## T5.3 — Minion attacks hero

**Goal:** Attacks can target a hero (no counter-damage from a bare hero).

**Depends on:** T5.2.

**Files (modify):** `AttackHandler` (if target is a hero: deal `attacker.Attack` to hero, no return damage unless hero has a weapon — weapons in Epic 10; for now heroes don't retaliate). **Test:** extend `CombatScenarios.cs`.

**Scenario tests:**
- *Face damage:* 3/2 attacks p2 hero → p2 health 30→27, `DamageTaken(p2,3,..)`, no return damage; lethal path triggers win-check (Epic 03).

**Notes:** hero-attacks-minion (a hero with weapon attacking) deferred to Epic 10. This ticket is minion→hero only.

---

## T5.4 — Attack validation

**Goal:** Enforce who/what may attack.

**Depends on:** T5.2.

**Files (modify):** `ActionValidator` (attack rules). **Test:** extend `CombatScenarios.cs`.

**Rules (this epic; Taunt added Epic 06):** attacker is owned by active player; `attacker.CanAttack` true; `!attacker.IsFrozen`; `attacker.AttacksUsedThisTurn < attacker.AttacksAllowedThisTurn`; `attacker.Attack > 0`; target is a valid enemy (enemy minion or enemy hero). Failures throw with specific messages.

**Scenario tests:**
- *Summoning sickness:* freshly summoned minion (CanAttack=false) → attack throws.
- *Already attacked:* `AttacksUsedThisTurn == AttacksAllowedThisTurn` → throws.
- *Frozen:* `IsFrozen` → throws.
- *0-attack:* 0/3 → throws.
- *Can't attack own:* target a friendly minion → throws.

**Notes:** `CanAttack` becomes true at the owner's next turn start (T5.6). Charge/Rush (Epic 06) set it true on summon.

---

## T5.5 — Simultaneous death from combat

**Goal:** Verify the death wave handles both combatants dying.

**Depends on:** T5.2, Epic 04 T4.6 ordering.

**Files:** test-only (`CombatScenarios.cs`) — behaviour already implemented; this ticket locks it with regression scenarios.

**Scenario tests:**
- *Trade:* 2/2 (active, board idx 0) attacks 2/2 (opponent, idx 0) → both die → `MinionDied` order: active's first, then opponent's.
- *Attacker survives, defender dies + defender's deathrattle later:* placeholder note — full deathrattle interplay tested in Epic 08.

---

## T5.6 — attacksAllowed + per-turn reset

**Goal:** Reset attack counters and clear summoning sickness at turn start.

**Depends on:** Epic 01 turn transition, T5.4.

**Files (modify):** `EndTurnHandler` / turn-start init (for the new active player's minions: `AttacksUsedThisTurn = 0`; `CanAttack = true` for minions with `Attack > 0` and not frozen; reset others appropriately). **Test:** extend `CombatScenarios.cs`.

**Scenario tests:**
- *Sick minion can attack next turn:* summon on p1 turn (CanAttack=false) → end turn → end turn (back to p1) → minion now `CanAttack`, can attack.
- *Counter resets:* minion attacked last turn → new turn → `AttacksUsedThisTurn == 0`, can attack again.
- *Frozen stays out:* frozen minion at turn start → unfreeze handled in Epic 07; here just confirm frozen ⇒ not given CanAttack.

**Notes:** the precise unfreeze timing (frozen *last* turn unfreezes now) is owned by Epic 07; coordinate the turn-start init so both concerns compose. Document the turn-start init as the single place attack/freeze/mana/draw all happen, in the spec's documented order.

---

## Epic 05 done-when

- Minions attack minions (mutual damage) and heroes (one-way), with deaths resolving in order.
- All attack-eligibility rules enforced; counters and summoning sickness reset at turn start.
- Event order `AttackDeclared → DamageTaken(target) → DamageTaken(attacker) → AttackResolved` is locked by tests.
