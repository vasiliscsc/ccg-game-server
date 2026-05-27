# Epic 10 — Hero Power & Weapons

**Goal:** Hero-level mechanics: a once-per-turn Hero Power, and Weapons that give the hero an attack value with durability that decays on use. Adds the Inspire trigger (fires on hero-power use).

**Depends on:** Epic 08 (triggers/handlers/effects), Epic 05 (attack flow — heroes now attack via weapons), Epic 03 (mana spend).

---

## T10.1 — Hero Power + cooldown

**Goal:** Use a hero power once per turn for its mana cost.

**Files (create):** `State/HeroPower.cs` (`CardId`, `ManaCost`, `JsonElement Definition`, `string? HandlerKey`), `Actions/UseHeroPowerAction.cs` (`{ required string PlayerId; string? TargetId }`), `Handlers/UseHeroPowerHandler.cs`, `Events/ManaEvents.cs`/new add `HeroPowerUsedEvent { string PlayerId; string? TargetId }`. **Modify:** `PlayerState` (add `HeroPower HeroPower`, `bool HeroPowerUsedThisTurn`), `ActionValidator` (active player, not used this turn, mana affordable), `GameEngineFactory`, `ScenarioBuilder` (`WithHeroPower`). **Test:** `Scenarios/HeroPowerScenarios.cs`.

**Handler:** deduct mana → emit `ManaChanged` → set `HeroPowerUsedThisTurn = true` → emit `HeroPowerUsedEvent` → enqueue the hero power's effect actions (via `DefaultCardHandler`-style interpretation of `HeroPower.Definition`, honouring `TargetId`).

**Scenario tests:**
- *Mage ping:* "Hero Power: deal 1 to a target" cost 2, mana 2 → use targeting enemy minion → 1 damage, mana 0, `HeroPowerUsed`.
- *Once per turn:* second use same turn → throws.
- *Can't afford:* mana < cost → throws.

---

## T10.2 — Hero-power reset on turn start

**Goal:** Reset the cooldown each turn.

**Depends on:** T10.1, Epic 05 turn-start init.

**Files (modify):** turn-start init (set new active player's `HeroPowerUsedThisTurn = false`). **Test:** extend `HeroPowerScenarios.cs`.

**Scenario test:** use hero power → end turn → end turn → can use again.

---

## T10.3 — Weapon equip + hero attack

**Goal:** Equipping a weapon gives the hero an attack value.

**Files (create):** `State/WeaponOnHero.cs` (`CardId`, `Attack`, `Durability`), `Actions/EquipWeaponAction.cs` (`{ required string PlayerId; required string CardId; string? SourceId }`), `Handlers/EquipWeaponHandler.cs`, `Events` add `WeaponEquippedEvent { string PlayerId; WeaponOnHero Weapon }`. **Modify:** `PlayerState` (add `WeaponOnHero? Weapon`, `int HeroAttack`), `PlayCardHandler` (Weapon-type card → enqueue `EquipWeaponAction`), `ScenarioBuilder` (`WithWeapon`). **Test:** `Scenarios/WeaponScenarios.cs`.

**Handler:** replace any existing weapon (old one destroyed → `WeaponDestroyedEvent`), set `Weapon`, set `HeroAttack = weapon.Attack`, emit `WeaponEquippedEvent`.

**Scenario tests:**
- *Equip sets attack:* play a 3/2 weapon → `HeroAttack == 3`, durability 2, `WeaponEquipped`.
- *Replace weapon:* equip over an existing → old destroyed (`WeaponDestroyed`) then new equipped.

---

## T10.4 — Hero attacks with weapon + durability loss

**Goal:** A hero with a weapon can attack; each attack costs 1 durability and takes return damage.

**Depends on:** T10.3, Epic 05 attack flow & validation.

**Files (modify):** `AttackHandler` (attacker can be a hero with `HeroAttack > 0`: deal `HeroAttack` to target; hero takes return damage from a minion target; lose 1 durability → `WeaponDurabilityLostEvent`), `ActionValidator` (hero attack eligibility: has weapon/attack > 0, not frozen, hasn't exceeded hero attacks this turn — add `HeroAttacksUsedThisTurn` to `PlayerState`, reset at turn start). `Events` add `WeaponDurabilityLostEvent { string PlayerId; int RemainingDurability }`. **Test:** `WeaponScenarios.cs`.

**Scenario tests:**
- *Hero kills minion, takes return:* hero 3-attack weapon attacks 2/2 → minion dies, hero takes 2, durability 2→1, events incl. `WeaponDurabilityLost`.
- *Hero face attack:* weapon hero attacks enemy hero → face damage, durability loss, no return.
- *One hero attack per turn:* second hero attack same turn → throws; resets next turn.
- *Frozen hero can't attack:* `PlayerState.IsFrozen` (Epic 07) → throws.

---

## T10.5 — Weapon destruction

**Goal:** Durability 0 destroys the weapon; explicit destroy too.

**Depends on:** T10.4.

**Files (create):** `Actions/DestroyWeaponAction.cs` (`{ required string PlayerId; string? SourceId }`), `Handlers/DestroyWeaponHandler.cs`, `Events` add `WeaponDestroyedEvent { string PlayerId; WeaponOnHero Weapon }`. **Modify:** `AttackHandler` (after durability loss, if `Durability <= 0` enqueue/perform destroy), `GameEngineFactory`. **Test:** `WeaponScenarios.cs`.

**Scenario tests:**
- *Worn out:* 3/1 weapon attacks once → durability 0 → `WeaponDestroyed`, `HeroAttack` back to 0.
- *Explicit destroy:* `DestroyWeaponAction` → weapon gone, `WeaponDestroyed`.

---

## T10.6 — Inspire trigger

**Goal:** "After you use your Hero Power" triggers.

**Depends on:** T10.1, Epic 08 trigger infra.

**Files (create):** sample Inspire `ITrigger` on `HeroPowerUsedEvent` filtered to owner; `Events/EffectEvents.cs` add `InspireTriggeredEvent { string PlayerId }` (or reuse). **Test:** extend `HeroPowerScenarios.cs`.

**Scenario tests:**
- *Inspire fires:* "Inspire: summon a 1/1" → use hero power → 1/1 appears; events `[HeroPowerUsed, (effect), InspireTriggered/MinionSummoned]`.
- *Only on own hero power:* opponent's hero power doesn't trigger it.

---

## Epic 10 done-when

- Hero Power usable once/turn for cost, resets each turn, supports targeting.
- Weapons equip (replacing old), grant hero attack, lose durability on attack, take return damage, and are destroyed at 0 durability or on demand.
- Hero attack eligibility (once/turn, frozen, attack>0) enforced; Inspire fires on hero-power use.
