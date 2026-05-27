# Epic 03 — Spells & Win Condition

**Goal:** First card that *does* something. Implement playing a card from hand (mana cost paid, card leaves hand), a damage spell that hits a hero, and the win-condition check that ends the game when a hero reaches 0. This introduces the `④ handler → ⑤ publish → ⑧ win-check` portion of the pipeline.

**Outcome demo:** Player has a 1-mana "Strike: deal 1 damage to a hero" spell. They play it targeting the opponent hero; opponent health drops by 1, mana is spent, the spell leaves hand. Enough strikes end the game.

**Depends on:** Epic 02 (mana, hand). Epic 01 (pipeline).

---

## T3.1 — Card play fields + PlayCardAction

**Goal:** Extend `Card` with cost/targeting basics and add the play command.

**Files (modify):** `State/Card.cs` (add `string? HandlerKey`, `JsonElement Definition` — `System.Text.Json` — for effect data; add `EffectiveCost` computed from `BaseManaCost` for now). **Create:** `Actions/PlayCardAction.cs` (`{ required string PlayerId; required string CardId; string? TargetId }`).

**Scope out:** `modifiers`/`StatModifier` (Epic 16), inversion fields (Epic 14).

**Notes:** `Definition` is a `JsonElement` holding effect descriptors (e.g. `{ "effects": [ { "type": "damage", "amount": 1, "target": "chosen" } ] }`). The interpreter is `DefaultCardHandler` (Epic 08) — but for this epic, wire a **minimal inline effect resolution** in `PlayCardHandler` good enough for a damage spell, and refactor to `DefaultCardHandler` in Epic 08. Note the seam explicitly.

**Verify:** builds. Commit.

---

## T3.2 — PlayCardHandler (mana + zone move + events)

**Goal:** Validate affordability, spend mana, remove from hand, announce the play.

**Depends on:** T3.1.

**Files (create):** `Handlers/PlayCardHandler.cs`, `Events/CardEvents.cs` add `CardPlayedEvent { string PlayerId; Card Card; string? TargetId }`, `Events/EffectEvents.cs` add `SpellCastEvent { string PlayerId; string CardId; string? TargetId }`. **Modify:** `ActionValidator` (mana affordability + card-in-hand + active player), `GameEngineFactory`. **Test:** `Scenarios/SpellScenarios.cs`.

**Handler flow (spell):** deduct `EffectiveCost` from mana → emit `ManaChangedEvent` → remove card from hand → emit `CardPlayedEvent` → if `Type == Spell` emit `SpellCastEvent` → enqueue the spell's effect actions (T3.3 supplies `DealDamageAction`).

**Validation (in `ActionValidator`):** player is active; card is in player's hand; `EffectiveCost <= Mana`. Failures throw `InvalidActionException` with specific message; no mutation.

**Scenario tests:**
- *Mana spent & card leaves hand:* p1 mana 1, hand `[Strike(cost1)]` → `PlayCard(Strike, target=p2hero)` → mana 0, hand empty, events begin `[ManaChanged(p1,0,_), CardPlayed, SpellCast, ...]`.
- *Can't afford:* mana 0, cost-1 card → throws; state unchanged.
- *Not in hand:* play unknown cardId → throws.

**Notes:** event order decision — `ManaChanged` before `CardPlayed`? Or after? Recommend **CardPlayed → SpellCast → effects → ManaChanged**? No: mana is paid *to* play, so emit `ManaChanged` first (cost paid), then `CardPlayed`. Lock this and document.

---

## T3.3 — DealDamageAction + handler (hero target)

**Goal:** A reusable damage action that lowers hero health.

**Depends on:** T3.2.

**Files (create):** `Actions/DealDamageAction.cs` (`{ required string TargetId; required int Amount; string? SourceId }`), `Handlers/DealDamageHandler.cs`, `Events/CombatEvents.cs` add `DamageTakenEvent { string TargetId; int Amount; int Overkill; string? SourceId }`. **Modify:** `GameEngineFactory`. **Test:** extend `SpellScenarios.cs`.

**Handler (hero only this epic):** resolve `TargetId` to a hero (a player whose `PlayerId == TargetId`); reduce `Health` by `Amount`; compute `Overkill = max(0, Amount - healthBefore)`; emit `DamageTakenEvent`. (Minion targets added in Epic 04; targeting type inferred from id per spec.)

**Scenario tests:**
- *Spell damages hero:* full pipeline — p1 plays Strike at p2 → p2 health 30→29; events end with `DamageTaken(p2,1,0,Strike)`.
- *Overkill recorded:* hero at 1 health, 5 damage → health -4 (or clamp to 0? clamp display health to ≥ -∞ but win-check uses ≤0); `Overkill = 4`.

**Notes:** decide whether `Health` clamps at 0 or goes negative. Recommend **let it go negative**; win-check uses `≤ 0`. `Overkill` still useful for future overkill triggers.

---

## T3.4 — First concrete damage spell

**Goal:** Prove the definition→effect path with one real card.

**Depends on:** T3.3.

**Files (create):** `Tests/Helpers/TestCards.cs` (factory for test cards, e.g. `TestCards.Strike(cost:1, amount:1)` producing a `Card` with `Definition` describing a damage effect). **Modify:** `PlayCardHandler` minimal effect interpretation (read `Definition.effects[].type == "damage"` → enqueue `DealDamageAction` with the action's `TargetId` from `PlayCardAction.TargetId`).

**Scenario tests:**
- *End-to-end via definition:* build Strike from `TestCards`, play at opponent → opponent takes the defined amount. Asserts the definition is actually read (not hardcoded).
- *Targetless/AoE deferred:* note that "all enemies" targeting is out of scope here (single chosen target only).

**Notes:** keep the inline interpreter dead simple — one effect type (`damage`), one target mode (`chosen`). Epic 08 generalises into `IEffect`/`DefaultCardHandler`. Leave a `// TODO Epic 08: replace with DefaultCardHandler` marker.

---

## T3.5 — Win-condition check

**Goal:** End the game when a hero hits ≤ 0; wire pipeline stage ⑧.

**Depends on:** T3.3, Epic 01 pipeline extension points.

**Files (create):** `Engine/WinConditionChecker.cs`, `Events/LifecycleEvents.cs` add `GameEndedEvent { string? WinnerId; required EndReason Reason }`; `Enums.cs` add `EndReason { Surrender, Defeat, Draw, Timeout }`. **Modify:** `GameEngine.Submit` (after each handled action / once queue drains, run win-check — implement the stage ⑧ extension point from Epic 01). **Test:** `Scenarios/WinConditionScenarios.cs`.

**Checker:** if either hero `Health <= 0`: both ≤ 0 → `GameEnded(null, Draw)`; one ≤ 0 → `GameEnded(otherPlayerId, Defeat)`. Set `Phase = Ended`. Emit once; subsequent actions rejected by `PhaseGuard`.

**Design note — when to check:** run win-check after the action queue drains (end of `Submit`), not after every micro-action, so a multi-hit sequence resolves fully first. Confirm this matches spec §4 ⑧ (after death resolution). Death resolution arrives in Epic 04; until then win-check runs after publish.

**Scenario tests:**
- *Lethal ends game:* p2 at 1 health, p1 plays Strike(1) → `GameEnded(p1, Defeat)`; phase `Ended`.
- *Simultaneous → draw:* both at ≤0 after a sequence → `GameEnded(null, Draw)`.
- *Post-end rejection:* after end, any action throws via `PhaseGuard`.

---

## T3.6 — Surrender

**Goal:** A player can concede.

**Depends on:** T3.5.

**Files (create):** `Actions/SurrenderAction.cs` (`{ required string PlayerId }`), `Handlers/SurrenderHandler.cs`. **Modify:** `GameEngineFactory`, `PhaseGuard` (allow surrender in any non-`Ended` phase). **Test:** extend `WinConditionScenarios.cs`.

**Handler:** set surrendering player health to 0 (or directly emit `GameEnded(opponent, Surrender)` + set phase). Recommend emitting `GameEnded(opponentId, Surrender)` directly and setting `Phase = Ended`.

**Scenario tests:** p1 surrenders on their turn → `GameEnded(p2, Surrender)`; surrender on opponent's turn also allowed → same result.

---

## Epic 03 done-when

- A definition-driven damage spell can be played end-to-end, paying mana and leaving hand.
- Hero damage works with overkill tracked; win-check ends the game (defeat/draw) and locks the phase.
- Surrender works from either side.
- Pipeline stage ⑧ (win-check) is implemented; ⑥/⑦ still pending (Epic 04/09).
