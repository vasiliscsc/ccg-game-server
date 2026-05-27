# Epic 09 — Auras

**Goal:** Continuous stat effects recalculated after every board-affecting action. Implement `IAura` + `AuraService`, wire the recalculation into pipeline stage ⑥, and handle aura-driven death (a minion that drops to ≤0 max-health when an aura is lost dies like any other).

**Depends on:** Epic 08 (triggers/handlers, `OnSummon`/`OnDeath` lifecycle), Epic 04 (death wave, computed stats with `AuraAttackBonus`/`AuraHealthBonus`).

---

## T9.1 — IAura infra + AuraService

**Goal:** The aura contract and the recalculation sweep.

**Files (create):** `Interfaces/IAura.cs` (`string AuraId`, `string SourceMinionId`, `IEnumerable<AuraEffect> Calculate(GameState)`), `State/AuraEffect.cs` (`record AuraEffect(string TargetMinionId, int AttackBonus, int HealthBonus)`), `Engine/AuraService.cs`, `Engine/AuraRegistry.cs` (tracks active `IAura` instances keyed by source minion). **Modify:** `GameEngineFactory`. **Test:** `Scenarios/AuraScenarios.cs`.

**`AuraService.Recalculate(state)`:** zero every minion's `AuraAttackBonus`/`AuraHealthBonus`; for each registered aura whose source is still on board, sum its `Calculate` results onto the targeted minions; clamp `CurrentHealth` to new `MaxHealth` if it now exceeds (aura health loss reduces current health if over cap). Auras are registered/unregistered by the source minion's `OnSummon`/`OnDeath` (Epic 08 lifecycle) — an aura is just a registered `IAura`.

**Scenario tests (unit-ish):** register an aura "+1/+1 to other friendly minions"; recalc → siblings buffed, source unaffected; remove source → recalc → bonuses gone.

**Notes:** aura bonuses live **only** in `AuraAttackBonus`/`AuraHealthBonus`, never enchantments (spec). Recalc fully rewrites them each pass — idempotent.

---

## T9.2 — Basic stat aura

**Goal:** A concrete aura minion.

**Depends on:** T9.1.

**Files:** `TestMinions.cs` add an aura minion (e.g. "Other friendly minions have +1/+1"); its `ICardHandler.OnSummon` registers the `IAura`, `OnDeath` unregisters. **Test:** `AuraScenarios.cs`.

**Scenario tests:**
- *Buffs allies:* summon aura minion next to a 2/3 → 2/3 shows 3/4 (via `Attack`/`MaxHealth`); summon another ally → also buffed after recalc.
- *Excludes self / excludes enemies:* source unbuffed; enemy minions unaffected.

---

## T9.3 — Recalc wired into pipeline (stage ⑥)

**Goal:** Recalculation happens automatically at the right moments.

**Depends on:** T9.1, Epic 04 pipeline.

**Files (modify):** `GameEngine.Submit` (implement the stage ⑥ extension point: recalc after any board-affecting action — pragmatically, recalc after every handled action that could change board/stats/keywords; simplest correct: recalc once per action before death resolution, and again after Phase 2 actions and Phase 3 in the wave per spec). **Test:** extend `AuraScenarios.cs`.

**Scenario tests:**
- *Buff appears immediately after summon:* play aura minion → ally stats updated in the same `Submit` (no extra action needed).
- *Buff updates on new ally:* summon ally after aura already present → ally buffed within that `Submit`.
- *Recalc during death wave:* aura source dies mid-wave → dependent stats update before win-check.

**Notes:** don't over-optimise — a full recalc pass each action is fine at this scale. Mark where in the wave recalc runs (after each Phase 2 action; after Phase 3) per spec §4 ⑥/⑦.

---

## T9.4 — Aura death

**Goal:** Losing an aura that propped up a minion's health kills it.

**Depends on:** T9.3, Epic 04 death collection.

**Files (modify):** ensure death collection treats `MaxHealth <= 0` (or `CurrentHealth <= 0` after clamp) as death; `AuraService` clamps `CurrentHealth` down when max drops. **Test:** `AuraScenarios.cs`.

**Scenario tests:**
- *Health-aura source dies → dependents die:* aura "+0/+2"; a 2/1 minion is alive at 2/3 (current 3) thanks to it... refine: a 1/1 that took 2 damage but survives only because aura gives +2 health → aura source destroyed → recalc drops its max below current damage → it dies in the same wave (`MinionDied`).
- *Ordering:* aura source and dependent die in correct board order within the wave.

**Notes:** craft the scenario carefully so the dependent is only alive *because* of the aura (took damage equal to base health). This is the canonical "aura death" regression.

---

## Epic 09 done-when

- `IAura` recalculation runs automatically (stage ⑥) after board-affecting actions and at the documented points in the death wave.
- Stat auras buff the right minions, exclude self/enemies, and update live.
- Aura-loss death works and orders correctly within the wave.
