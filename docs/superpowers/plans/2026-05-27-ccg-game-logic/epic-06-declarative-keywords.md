# Epic 06 — Declarative Keywords

**Goal:** Introduce the `IKeyword` system and the keywords that are *queried* by the engine rather than reacting to events: Taunt, Divine Shield, Charge, Rush, Stealth. These need no event listeners — the pipeline checks `Keywords.Contains(...)` at the relevant decision point.

**Depends on:** Epic 05 (combat, attack validation), Epic 04 (summon, damage).

---

## T6.1 — IKeyword infra + registration hook

**Goal:** The keyword contract, a registry, and the summon-time hook that calls `OnApplied`.

**Files (create):** `Interfaces/IKeyword.cs` (`string KeywordId`, `OnApplied(string minionId, IEventBus bus, IActionQueue queue)`, `OnRemoved(string minionId, IEventBus bus)`), `Keywords/KeywordRegistry.cs` (`IReadOnlyDictionary<string, IKeyword>`; resolves keyword strings → implementations). **Modify:** `SummonMinionHandler` (after placing a minion, for each keyword call `OnApplied`), the engine composition (`GameEngineFactory` builds the registry and passes the bus/queue). **Test:** `Scenarios/KeywordInfraScenarios.cs`.

**Design:** declarative keywords implement `IKeyword` with **no-op** `OnApplied`/`OnRemoved`; they exist mainly so the registry is uniform and so silence (Epic 16) can call `OnRemoved`. Keyword string constants live in `Keywords/Keywords.cs` (`const string Taunt = "taunt"`, etc.) to avoid stringly-typed bugs.

**Scenario test:** summon a minion with `["taunt"]` → no error, keyword present on board minion. (Behavioural effects in T6.2+.)

**Notes:** the bus/queue passed to `OnApplied` are the engine's live instances; active keywords (Epic 07) capture them in closures to register listeners. Confirm `KeywordRegistry` is injected, not static, so tests can register custom keywords.

---

## T6.2 — Taunt

**Goal:** Enemy attacks must target a Taunt minion if any exist.

**Depends on:** T6.1, Epic 05 attack validation.

**Files (create):** `Keywords/TauntKeyword.cs` (no-op listeners). **Modify:** `ActionValidator` attack rules (if the defending player has any minion with Taunt on its board, the attack target must be one of them; attacking the hero or a non-Taunt minion throws). **Test:** `Scenarios/TauntScenarios.cs`.

**Scenario tests:**
- *Must hit taunt:* opponent has Taunt 2/3 + vanilla 2/2; attack the vanilla or hero → throws; attack the Taunt → ok.
- *No taunt, free targeting:* opponent has no taunt → can hit hero.
- *Dead taunt frees targeting:* taunt at 0 health after a prior action but already removed by death wave → no longer constrains (it's off-board).

**Notes:** neutral-zone Taunt is **ignored** (spec) — that's Epic 15; here Taunt only matters for owned boards.

---

## T6.3 — Divine Shield

**Goal:** First instance of damage absorbs the hit and pops.

**Depends on:** T6.1, Epic 03/04 damage handler.

**Files (create):** `Keywords/DivineShieldKeyword.cs` (no-op listeners). **Modify:** `DealDamageHandler` (if target minion has `divine_shield` and `Amount > 0`: remove the keyword, emit `DivineShieldBrokenEvent`, apply **no** damage), `Events/StatusEvents.cs` (`DivineShieldBrokenEvent { string MinionId }`). **Test:** `Scenarios/DivineShieldScenarios.cs`.

**Scenario tests:**
- *Absorbs spell:* DS 2/3, Strike(5) → no `DamageTaken`, `DivineShieldBroken`, minion full health, keyword gone.
- *Second hit lands:* after pop, another Strike(2) → `DamageTaken(2)`, health 1.
- *Absorbs combat damage:* DS minion attacked → shield pops instead of taking combat damage; attacker still takes return damage.

**Notes:** DS is checked inside the damage handler (spec §3) — *not* a listener. 0-damage hits do **not** pop the shield.

---

## T6.4 — Charge & Rush

**Goal:** Bypass summoning sickness, with Rush's minion-only restriction.

**Depends on:** T6.1, Epic 05 attack rules.

**Files (create):** `Keywords/ChargeKeyword.cs`, `Keywords/RushKeyword.cs` (both set state on summon — implement via `OnApplied` setting `CanAttack = true`, OR have `SummonMinionHandler` check these keywords; recommend handler-side check to keep `OnApplied` no-op style consistent — decide & document). **Modify:** `SummonMinionHandler` (Charge ⇒ `CanAttack=true`; Rush ⇒ `CanAttack=true` + flag minion-only this turn), `ActionValidator` (Rush minion cannot target enemy hero on its summon turn). **Test:** `Scenarios/ChargeRushScenarios.cs`.

**Tracking Rush:** add `SummonedOnTurn` (int) to `MinionOnBoard`; Rush restriction applies while `SummonedOnTurn == Turn.Number`.

**Scenario tests:**
- *Charge hits face turn 1:* summon Charge minion → immediately attack hero → ok.
- *Rush hits minion not face:* summon Rush minion → attack enemy minion ok; attack enemy hero → throws.
- *Rush hits face next turn:* after a turn cycle, Rush minion can attack hero.

---

## T6.5 — Stealth

**Goal:** Stealthed minions can't be targeted by enemies; stealth breaks on attacking.

**Depends on:** T6.1.

**Files (create):** `Keywords/StealthKeyword.cs`. **Modify:** `ActionValidator` (enemy spells/attacks cannot target a minion with `stealth`), `AttackHandler` (when a stealthed minion attacks, remove `stealth` before/after declaring — decide timing; recommend remove on `AttackDeclared`). **Test:** `Scenarios/StealthScenarios.cs`.

**Scenario tests:**
- *Can't be targeted:* enemy Stealth minion → Strike at it throws; friendly spell *can* target own stealthed minion (stealth only blocks enemies).
- *Breaks on attack:* stealthed minion attacks → `stealth` gone afterward, now targetable.
- *AoE vs stealth:* note — untargeted/AoE effects (Epic 08) still hit stealthed minions; single-target enemy effects don't. Out of scope to test here beyond the targeting rule.

---

## Epic 06 done-when

- `IKeyword` registry + summon-time `OnApplied` hook exist and are injectable.
- Taunt constrains targeting; Divine Shield absorbs in the damage handler; Charge/Rush manage summoning sickness with Rush's face restriction; Stealth blocks enemy targeting and breaks on attack.
- All keyword scenario suites green.
