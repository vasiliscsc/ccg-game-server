# Epic 04 — Minions & Death Resolution

**Goal:** Put bodies on the board. Summon vanilla minions (no text), target them with spells, and implement the **death resolution wave** (Phase 1 only — no deathrattles/reborn yet) with the spec's deterministic sort order. This wires pipeline stage ⑦.

**Outcome demo:** Summon two 2/3 minions; a spell deals 3 to one; it dies, leaves a graveyard entry, and `MinionDiedEvent` fires. Multiple simultaneous deaths resolve in current-L→R then opponent-L→R order.

**Depends on:** Epic 03 (spells, damage, pipeline win-check).

---

## T4.1 — MinionOnBoard + board zone

**Goal:** The on-board minion type and the board list.

**Files (create):** `State/MinionOnBoard.cs`. **Modify:** `State/PlayerState.cs` (add `List<MinionOnBoard> Board`), `State/Card.cs` (add `int? BaseAttack`, `int? BaseHealth` for minion/weapon cards), `ScenarioBuilder` (`WithMinion(playerId, MinionOnBoard, position?)`), `Tests/Helpers/TestMinions.cs` (factory, e.g. `TestMinions.Vanilla(attack:2, health:3)`).

**`MinionOnBoard` shape (per spec §1; include all fields now to avoid churn, even if unused until later epics):** `MinionId`, `CardId`, `string? OwnerId`, `BaseAttack`, `BaseHealth`, `List<StatModifier> Enchantments`, `AuraAttackBonus`, `AuraHealthBonus`, computed `Attack`, computed `MaxHealth`, `CurrentHealth`, `List<string> Keywords`, `IsInverted`, `CanAttack`, `AttacksUsedThisTurn`, `AttacksAllowedThisTurn` (default 1), `SummonOrder`, `IsFrozen`, `IsDamaged`. **Create:** `State/StatModifier.cs` now (used by computed stats even if buffs come in Epic 16).

**Computed props:** `Attack => BaseAttack + Σ Enchantments.AttackDelta + AuraAttackBonus`; `MaxHealth => BaseHealth + Σ Enchantments.HealthDelta + AuraHealthBonus`.

**Verify:** builds; computed stat unit test (base + enchant + aura). Commit.

**Notes:** `SummonOrder` is assigned by the summon handler from a per-session counter on `GameState` (add `int NextSummonOrder` to `GameState`). Even though trigger ordering uses board index (spec decision), `SummonOrder` is kept for disambiguation.

---

## T4.2 — SummonMinionAction + board capacity

**Goal:** Summon a minion to a player's board.

**Depends on:** T4.1.

**Files (create):** `Actions/SummonMinionAction.cs` (`{ required string CardId; string? OwnerId; int? BoardPosition; string? SourceId }`), `Handlers/SummonMinionHandler.cs`, `Events/BoardEvents.cs` (`MinionSummonedEvent { MinionOnBoard Minion; string? OwnerId }`). **Modify:** `GameEngineFactory`, `ActionValidator` (board cap when summoned via card play), `State/GameConstants.cs` (`MaxBoardSize = 7`). **Test:** `Scenarios/MinionSummonScenarios.cs`.

**Handler:** build `MinionOnBoard` from the card (base stats, keywords from definition), assign `MinionId` (new guid) + `SummonOrder = state.NextSummonOrder++`; insert at `BoardPosition` (or append); set `CanAttack=false` (summoning sickness) unless Charge/Rush (Epic 06); emit `MinionSummonedEvent`.

**Scenario tests:**
- *Summon appears:* empty board → summon 2/3 → board has 1, event `MinionSummoned`.
- *Position respected:* board `[A,B]`, summon C at pos 1 → `[A,C,B]`.
- *Cap enforced:* board has 7, play a minion → `InvalidActionException` (full board). (Effect-summons over cap: just fizzle silently — note for Epic 08.)

---

## T4.3 — Summon-minion spell

**Goal:** A card whose effect summons a minion (connects play → summon).

**Depends on:** T4.2, Epic 03 play path.

**Files (modify):** `TestCards.cs` (`TestCards.SummonVanilla(...)` with a `summon` effect in `Definition`), `PlayCardHandler` inline interpreter (handle `effect.type == "summon"` → enqueue `SummonMinionAction` for the casting player). For a **Minion-type card**, playing it summons itself (the common case) — handle `Type == Minion` by enqueuing a self-summon directly.

**Scenario tests:**
- *Play minion card → on board:* hand `[Vanilla2/3]`, mana ok → play → board has it, hand empty, events `[ManaChanged, CardPlayed, MinionSummoned]`.
- *Spell that summons:* a Spell card with summon effect → minion appears, `SpellCast` + `MinionSummoned` present.

**Notes:** confirm event order for playing a minion: `ManaChanged → CardPlayed → MinionSummoned` (no `SpellCast` for minions). Document.

---

## T4.4 — Spell targeting a minion

**Goal:** Damage actions can hit minions, with target validation.

**Depends on:** T4.2, Epic 03 `DealDamageHandler`.

**Files (modify):** `Handlers/DealDamageHandler.cs` (resolve `TargetId` to a minion if not a hero: search both boards + neutral zone; reduce `CurrentHealth`; set `IsDamaged = CurrentHealth < MaxHealth`; emit `DamageTakenEvent`), `ActionValidator` (target exists & is alive; stealth/targeting rules deferred to Epic 06). **Test:** `Scenarios/SpellScenarios.cs` extend.

**Scenario tests:**
- *Spell hits minion:* 2/3 minion, Strike(3 dmg) → `CurrentHealth` 0, `IsDamaged` true, `DamageTaken(minionId,3,0,_)`. (Death itself in T4.5.)
- *Invalid target:* target a non-existent id → throws.

**Notes:** the engine infers hero vs minion from id (spec §2A) — implement a `state.ResolveTarget(id)` returning a discriminated result (`Hero p` / `Minion m` / `null`). Put this on `GameState` for reuse by combat (Epic 05).

---

## T4.5 — Death resolution Phase 1 + graveyard

**Goal:** The core death sweep — remove dead minions, grave them, announce deaths. Wire pipeline stage ⑦.

**Depends on:** T4.4.

**Files (create):** `Engine/DeathResolutionService.cs`, `State/GraveyardEntry.cs` (abstract + `GraveyardMinion`), `Events/BoardEvents.cs` add `MinionDiedEvent { string MinionId; MinionOnBoard Snapshot; string? SourceId; int DiedOnTurn }`. **Modify:** `PlayerState` (add `List<GraveyardEntry> Graveyard`), `GameEngine.Submit` (after queue drains: run `DeathResolutionService.Resolve` *before* win-check — implement stage ⑦ extension point). **Test:** `Scenarios/DeathResolutionScenarios.cs`.

**Phase 1 only this ticket:** collect minions with `CurrentHealth <= 0` (sorted — see T4.6); for each: remove from board, add `GraveyardMinion` snapshot (`DiedOnTurn = Turn.Number`), emit `MinionDiedEvent`. **No** deathrattle/reborn yet (Epic 08). After removals, loop again (a future deathrattle could kill more — structure the wave loop now even though Phase 2/3 are empty).

**Scenario tests:**
- *Lethal spell kills minion:* 2/3, Strike(3) → minion gone from board, graveyard has snapshot, `DamageTaken` then `MinionDied`.
- *Non-lethal leaves it:* 2/3, Strike(2) → still on board at 1 health, no `MinionDied`.

**Notes:** structure as `while (CollectDeaths() is { Count: >0 } dead) { Phase1(dead); /* Phase2/3 hooks no-op */ }`. Mark the Phase 2/3 call sites with `// Epic 08`. Win-check (stage ⑧) runs after this returns.

---

## T4.6 — Death sort order across boards

**Goal:** Deterministic multi-death ordering per spec.

**Depends on:** T4.5.

**Files (modify):** `DeathResolutionService` (sort collected deaths: current player `Board` by index, then opponent `Board` by index, then `NeutralZone` by index — neutral zone field added in Epic 15; guard for now). **Test:** extend `DeathResolutionScenarios.cs`.

**Scenario tests:**
- *Four simultaneous deaths order:* current player board `[A,B]`, opponent `[C,D]`, all at lethal via an AoE-ish sequence (enqueue 4 `DealDamageAction`s) → `MinionDied` events in order `A,B,C,D`.
- *Order independent of damage order:* damage them in order `D,C,B,A` → deaths still `A,B,C,D`.

**Notes:** "current player" = `Turn.ActivePlayerId` at resolution time. Add a helper `state.PlayersInDeathOrder()` returning `[active, opponent]`.

---

## T4.7 — DestroyMinionAction (instant kill)

**Goal:** Effects that destroy regardless of health.

**Depends on:** T4.5.

**Files (create):** `Actions/DestroyMinionAction.cs` (`{ required string MinionId; string? SourceId }`), `Handlers/DestroyMinionHandler.cs`. **Modify:** `GameEngineFactory`. **Test:** extend `DeathResolutionScenarios.cs`.

**Handler:** mark the minion for death (set `CurrentHealth = 0` or a `PendingDestroy` flag). Recommend a `MarkedForDeath` bool on `MinionOnBoard` so "destroy" is distinct from "0 health" and survives aura recompute. Death resolution collects `CurrentHealth <= 0 || MarkedForDeath`.

**Scenario tests:**
- *Destroy full-health minion:* 5/5 at full → `DestroyMinion` → dies via resolution, `MinionDied` (no `DamageTaken`).
- *Destroy respects death order:* destroy two minions across boards → ordered deaths.

---

## Epic 04 done-when

- Minions summon (with cap), take spell damage, and die through a proper death-resolution wave with correct multi-death ordering.
- Graveyard captures snapshots; `MarkedForDeath` supports instant-destroy.
- Pipeline stages ⑦ (death) + ⑧ (win) both run after queue drain; the wave loop has Phase 2/3 hooks ready for Epic 08.
- `state.ResolveTarget` / `PlayersInDeathOrder` helpers exist for combat to reuse.
