# Epic 15 — Neutral Zone

**Goal:** The third board zone owned by neither player. Neutral minions have no turn, can be attacked/commanded/mind-controlled, ignore Taunt, and are unaffected by player auras. Supports a configured capacity and optional repopulation on turn start.

**Depends on:** Epic 04 (board/minions/death), Epic 05 (combat), Epic 06 (Taunt), Epic 09 (auras).

---

## T15.1 — Neutral zone + spawn

**Goal:** The zone, its config, and spawning into it.

**Files (create):** `State/NeutralZoneConfig.cs` (`int MaxCapacity`, `List<Card> SpawnPool`, `bool RepopulateOnTurnStart`), `Actions/SpawnNeutralMinionAction.cs` (`{ required string CardId; int? Position; string? SourceId }`), `Handlers/SpawnNeutralMinionHandler.cs`, `Events/BoardEvents.cs` add `NeutralMinionSpawnedEvent { MinionOnBoard Minion }`. **Modify:** `GameState` (`List<MinionOnBoard> NeutralZone`, `NeutralZoneConfig? NeutralZoneConfig`), `ScenarioBuilder` (`WithNeutralZone(config)`, `WithNeutralMinion(...)`). **Test:** `Scenarios/NeutralZoneScenarios.cs`.

**Handler:** create a `MinionOnBoard` with `OwnerId = null`, `CanAttack = false` (neutral minions never take their own turn), place in `NeutralZone` respecting `MaxCapacity` (fizzle if full); emit `NeutralMinionSpawnedEvent`.

**Scenario tests:**
- *Spawn into zone:* config cap 3, empty → spawn → 1 neutral minion, `NeutralMinionSpawned`.
- *Capacity:* zone full → spawn fizzles (no event / a no-op event — decide; recommend silent fizzle).
- *No config = no zone:* `NeutralZoneConfig == null` → spawn rejected/ignored.

---

## T15.2 — Attacking & commanding neutral minions

**Goal:** Players can attack neutral minions and command them to attack.

**Depends on:** T15.1, Epic 05 combat.

**Files (modify):** `AttackHandler`/`ActionValidator` (a player's minion may target a neutral minion; a player may command a neutral minion to attack — the `AttackAction.AttackerId` is a neutral minion, `SourcePlayerId` is the commanding active player; mutual damage as normal). **Test:** `NeutralZoneScenarios.cs`.

**Scenario tests:**
- *Attack a neutral:* p1 minion attacks a neutral 2/3 → mutual damage; neutral can die → `MinionDied` (snapshot `OwnerId == null`).
- *Command a neutral:* active player commands a neutral minion to attack an enemy minion → resolves as a normal attack; neutral's `AttacksUsedThisTurn` increments (reset rule: define — recommend command-once-per-turn by the active player; lock it).
- *Death order with neutral:* neutral deaths sort after both players' boards (Epic 04 T4.6).

**Notes:** decide neutral-attack cadence carefully (who can command, how often). Keep a simple deterministic rule and document; this is novel (no Hearthstone analogue).

---

## T15.3 — Taunt ignored + auras don't apply

**Goal:** The two special exemptions for neutral minions.

**Depends on:** T15.1, Epic 06 Taunt, Epic 09 auras.

**Files (modify):** `ActionValidator` Taunt check (skip neutral minions entirely — a neutral Taunt does not constrain targeting, and Taunt does not protect/force-target within the neutral zone), `AuraService` (player `IAura`s exclude neutral-zone minions from their targets). **Test:** `NeutralZoneScenarios.cs`.

**Scenario tests:**
- *Neutral Taunt ignored:* a neutral minion with Taunt does not force enemies to attack it; players can still freely target heroes/other minions.
- *Player aura skips neutral:* a "+1/+1 friendly minions" aura does not buff neutral minions (they stay base).
- *(Optional) neutral-sourced aura:* if a neutral minion has an aura, define its scope — recommend neutral auras affect only the neutral zone, or nothing; document the chosen rule.

---

## T15.4 — Repopulation on turn start

**Goal:** Optionally refill the neutral zone each turn.

**Depends on:** T15.1, Epic 01 turn transition.

**Files (modify):** turn-start init (if `RepopulateOnTurnStart`, spawn from `SpawnPool` up to `MaxCapacity`), `Events/BoardEvents.cs` add `NeutralZoneRepopulatedEvent { List<MinionOnBoard> SpawnedMinions }`. **Test:** `NeutralZoneScenarios.cs`.

**Scenario tests:**
- *Refill to cap:* cap 3, zone has 1, repopulate on → turn start → 2 spawned from pool → zone has 3, `NeutralZoneRepopulated` with the 2 new.
- *Disabled:* `RepopulateOnTurnStart == false` → no spawns at turn start.
- *Selection from pool:* define pool draw (random vs in-order); for deterministic tests, in-order or seedable RNG — document and make seedable.

**Notes:** introduce a seedable RNG abstraction (`IRandom`) here if not already, so repopulation and other random effects are testable. Many earlier tickets (Discover, random discard) benefit — consider backfilling `IRandom` injection.

---

## Epic 15 done-when

- Neutral zone spawns (capacity-bounded), can be attacked and commanded, and its deaths sort last.
- Neutral minions ignore Taunt and are excluded from player auras (with neutral-aura scope documented).
- Optional turn-start repopulation works via a seedable RNG.
