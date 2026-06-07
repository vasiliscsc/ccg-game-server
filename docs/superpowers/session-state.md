# CCG Session State

> Full context for resuming work. Kept in the project so it travels with the repo.

---

## What we're building

An online 1v1 collectible card game (Hearthstone-clone for initial scope). Features: ranked PvP matchmaking (MMR-based), spectatable game sessions, full accounts + card collection system, authoritative server (client is a pure event renderer). Target scale: 1k–10k concurrent players.

---

## Architecture (locked)

**Core Split** — two services + one client:

| Component | Stack | Role |
|---|---|---|
| Game Server | C# / .NET (ASP.NET Core) | Authoritative game state, WebSocket connections, action processing, event broadcasting |
| Platform API | TypeScript / Node.js | Auth, card collection, deck management, matchmaking, MMR |
| Client | Unity or Godot (TBD) | Renders events, sends player actions |

**Shared logic:** `CCG.GameLogic` — standalone C# class library (no engine/framework deps). Referenced by Game Server; optionally by Unity client.

**Storage:** PostgreSQL (persistent) + Redis (ephemeral queue/session state).

---

## Subsystem build order

| # | Subsystem | Service |
|---|---|---|
| 1 | Game Server | C# / .NET |
| 2 | Auth & Accounts | Platform API |
| 3 | Collection & Decks | Platform API |
| 4 | Matchmaking | Platform API |
| 5 | Client | Unity / Godot |

---

## Where we stopped

**Game Mechanics spec is COMPLETE and approved** — all 4 sections written to `docs/superpowers/specs/2026-05-26-game-mechanics.md` (data model, actions/events, engine architecture, action pipeline).

**Implementation plan is COMPLETE** — broken into 16 epics with ~80 tickets under `docs/superpowers/plans/2026-05-27-ccg-game-logic/`. Start at the README there; it has the full epic index, ticket outline, and a writing-progress tracker. Each ticket is a small, testable increment using the scenario-test methodology (build state → submit action(s) → assert ordered events + state).

**Implementation is NOT yet started** — we deferred kickoff in favour of a spec-refinement pass driven by the older Unity proof-of-concept at `C:\Users\vasil\UnityProjects\ccg`.

### Spec refinements applied in session 2026-05-28

1. **Card and Minion Identity subsection** added to spec §1 — establishes `definitionKey` as the only cross-entity link; `Card.id` and `MinionOnBoard.minionId` are independent never-reused allocators; tokens, graveyard lineage, and replay semantics specified.
2. **Minion → Card Transitions subsection** added to spec §1 — defines `MinionToCardPolicy { Stripped, RetainEnchantments }`, retain/strip table, and the scope note that v1 doesn't track per-source keyword grants (future extension if needed).
3. **`grantedKeywords: string[]`** field added to both `MinionOnBoard` and `Card` — non-aura keyword grants survive `RetainEnchantments` bounces; aura grants do not. `isInverted` preserved across all minion↔card transitions.
4. **`ReturnToHandAction`** signature updated to include `policy: MinionToCardPolicy`. Epic 16 T16.3 rewritten with six scenario tests (Stripped, RetainEnchantments preserves buffs, aura grants don't survive, isInverted preservation, damage always discarded, full-hand destroy).

### Next session resume point

**▶ BORROW-LIST PASS COMPLETE — all 13 items resolved (✅ below). RESUME AT THE END-OF-PASS PLAN RECONCILIATION.** Session of 2026-06-02 walked Items 12 and 13 and closed here. The spec is now fully refined (plus post-pass Fireplace-comparison amendments — see below); the next task is purely mechanical plan-side reconciliation, then implementation.

**▶▶ Post-pass Fireplace amendments — current state (2026-06-03).** The spec is no longer frozen at the 13-item state; a comparison against jleclanche/fireplace (open-source HS clone) is driving refinements on shared subsystems, in the borrow-list note's **"Fireplace-comparison amendments (post-pass)"** section. Done so far:
- **Keyword-model collapse (2026-06-02, APPLIED)** — eliminated the active/declarative keyword split; all keywords declarative, pulled at own-action moments via role-interface hooks (`IOnDealtDamage`); `IKeyword` loses `OnApplied`/`OnRemoved` (bus = `ITrigger` only). Recorded under Item 8's "second follow-on amendment."
- **Death-resolution cadence → Option 2 — Comparison point B (DECIDED + APPLIED 2026-06-03).** Cascade-settle (Hearthstone-faithful) death timing replaces per-action ⑦: **§4 rewritten as a loop** — ①–⑥ per action draining the queue; ⑦ death + ⑧ win are **settle stages** firing only when the queue empties; ⑨ drives the loop. Introduces the **mortally-wounded / pending-death** minion state (≤0 HP, stays on board until the settle). Kept the FIFO trigger-queue (not Fireplace's literal stack). Game-design rationale: unlocks the "dying-matters" family (overkill, retaliation, simultaneous-death, count-the-doomed); clinched by **R1 Intervention × lethal** + **R2 Inversion × lethal**.
- **Pending-death targetability → `All…/Alive…/Dead…` selector trichotomy (APPLIED, §3).** `All… = Alive… ∪ Dead…`: `All…` include mortally-wounded (HS AoE/count), `Alive…` = `hp>0` (opt-out), `Dead…` = `hp≤0` on board (home for R1). +`AllNeutralMinions`; graveyard/resurrection = a separate primitive, not an `ITargetSelector`.
- **Item 12 Fork-A (APPLIED, §3).** Random-K consumption row now carries the *selector reference*, evaluated against the current board at ④ (not a frozen pool).
- **R1 dying-window intervention — pipeline-realization (recorded; mechanism APPLIED, card-layer DEFERRED).** Fits **by construction**: the death wave runs only at queue-drain, so the mortally-wounded minion lingers while the queue is non-empty; an enqueued intervention resolves before the death wave. Applied: **`MinionMortallyWoundedEvent`** (§2B), the **pre-halt death rule + R1 exception** (§4 choice/intervention blocks). Deferred to **menu point D**: what opens the window (general rule vs armed/secret card), batching, marked-for-destruction scope, and the R2 inversion stat-math.

**Spec is internally consistent as of 2026-06-03** — the §4 cadence rewrite + Fork-A are landed; the earlier §3→§4 forward reference is satisfied; no stale per-action-death language remains (grep-verified).

- **Point D — Secrets / armed reactive triggers → ONE unified reactive-trigger + interception model (DECIDED + APPLIED 2026-06-04).** Secrets, the locked Intervention system, and the R1 dying-window collapse into **one** mechanism over `ITrigger`: (1) **zone-scoped reactive hosting** (a trigger is live by its host's zone — board / **hand** = player-choice intervention / **secret zone** = auto, deferred — reusing the existing `ICardHandler` register-on-entry lifecycle); this *answers "what opens a window"* (a hand card whose live trigger matched, never a general rule) and harmonizes the locked "play one card / skip" literally. (2) **Two hook phases = the whole taxonomy**: **pre** (`ActionDeclaredEvent`, new stage **③′**) → interception; **post** events → plain reaction (the mortally-wounded events incl. new **`HeroMortallyWoundedEvent`** are post-reactions exploiting the ⑦/⑧ settle deferral — "consequence-deferral" is *not* a third kind). (3) **Interception = ordinary effects + re-resolution, NO disposition vocabulary** — the held action re-validates and most alterations emerge (shield absorbs, poison retaliates, killed/returned attacker fizzles, Taunt forces redirect); the entire irreducible surface is **two ops: `cancel` + `retarget`-in-place**. (4) **Uniform declaration** (one generic `ActionDeclaredEvent`, not a `*Declared` taxonomy; fires once; depth-1 cap). Completeness invariant: *anything interceptable is a discrete action*. **Subsumes menu point A (Predamage)**: Ice-Block/armor/Divine-Shield/double-damage/caps/prevent are interceptions on `DealDamageAction` — only residue of A is damage-modifier *precedence*. Spec **APPLIED**: §3 new "Reactive Triggers, Interventions & Interception" subsection; §4 new ③′ stage + PendingIntervention rewrite (two kinds, `heldAction` nullable) + ⑧ hero-save note; §2 `ActionDeclaredEvent` + `HeroMortallyWoundedEvent` + nullable `StartInterventionAction.heldAction`; §1 `PendingIntervention` nullable. **Batching RESOLVED (2026-06-04, APPLIED):** post-reaction player windows don't halt per sub-event — matches accumulate and **one** window opens per (card, settle) before ⑦ with the **matched set as the selector** (AoE on 7 minions → 1 window, pick 1); `SubmitInterventionAction` gained `targetIds[]?`; settle order = batched windows → ⑦ → ⑧; declaration-hold needs no batching; deterministic no-choice reactions → auto/secret flavor (no window). **Point A (Predamage) CLOSED 2026-06-04** — subsumed: damage→`DealDamageAction` (combat included), AoE = one selector-carrying damage action (one declaration → batched pre-window), ④ modifier precedence pinned. **Still deferred (game-feel):** Secret auto flavor + zone/constraints, visibility (hand-live = hidden), cost timing, marked-for-destruction save scope, R2 stat-math. Full record + Plan impact in the borrow-list note's "Comparison point D".

- **Point D follow-up — intervention-window timing made bulletproof (DECIDED + APPLIED 2026-06-06, session 4).** Resolved *when* the two hook kinds fire by tracing a worked example. **Pre-hooks (③′)** fire immediately, per-action, mid-cascade (forced — interception must precede ④). **Post-hooks** move to a **new stage ⑥′** (per-action, after each action's ⑤/⑥), batching *one action's* events → AoE = one window, "deal 1 twice" = two windows; **never per-event, never per-cascade** — superseding the 06-04 cross-cascade-accumulate-at-settle model (outcomes unchanged, only the *when*). **A window's response resolves immediately** (both kinds) before the queue resumes — load-bearing: FIFO-appending lets an unrelated `PendingChoice` pre-settle deaths and kill the minion a save targeted (a real bug the old model had). The **pre-halt death rule shrinks to type-A** (`PendingChoice` + declaration-holds only); **post-reaction windows never pre-settle** — which dissolves the confusing "R1 exception" framing into "post-reactions are simply a different category." Depth-1 cap suppresses only *player* windows; board/auto triggers fire inline. Applied to §3 (pillar 2 + Ordering table), §4 (⑥′, ⑨ rewrite, new "Response resolution", PendingChoice/PendingIntervention/Batching rewrites), §2 (`SubmitInterventionAction` row). Full record in the borrow-list note's "Comparison point D follow-up (2026-06-06)".

- **Combat atomicity → Option A (two independent damage actions) (DECIDED + APPLIED 2026-06-06, session 4).** Combat = **two independent `DealDamageAction`s** (attacker strike + defender retaliation), each its own ③′/⑥′, processed sequentially — *not* one atomic unit. Chosen because it's the **only option consistent with the just-locked ⑥′ rule** (two instances → two windows); B/C would carve a combat exception. Death simultaneity preserved regardless (one shared wave at ⑦). Accepted cost: combat not atomic w.r.t. windows (cards interleave between hits; per-hit board state). **Payoff: enables dying-swing retaliation by construction** (mortally-wounded defender retaliates with live state → Poisonous/Lifesteal/"retaliation doubled while mortally wounded" all work; base attack snapshot at `AttackAction` ④, ×modifiers pulled per damage ④). Applied to §3 (point-A close-out combat bullet) + §2A (`DealDamageAction` row). Full record in the borrow-list note's "Comparison point D follow-up (cont.) — combat atomicity".

- **Section 1 data-model hygiene pass (2026-06-06, session 4) — no design change, consistency only.** Added trigger mentions to `MinionOnBoard` (board-zone bus registration) + `Card` (hand/secret reactive triggers); fixed stale "declarative keywords" comments in §1 + §3 `IAura` (now agree with the collapsed `IKeyword`); defined `EffectContext` as the **base** with `CardPlayContext : EffectContext` **superset** (resolves an undefined-type gap); renamed `MinionOnBoard.cardId`→`definitionKey`, added `Card.definitionKey`, and aligned all by-definition action/supporting-type params (`SummonMinionAction`/`SpawnNeutralMinionAction`/`GiveCardAction`/`EquipWeaponAction`/`TransformMinionAction.newDefinitionKey`/`WeaponOnHero`/`HeroPower`) to `definitionKey`, with a new §2A ID-convention note (instance id vs. library key).

---

### Session 5 (2026-06-07): hole-hunting pass — neutral zone control/command/graveyard (DECIDED + APPLIED)

Open Q/A sweep for spec holes (distinct from the 13-item borrow list and the Fireplace points). First hole closed: **#3 — neutral zone**. Full record + Plan impact in the borrow-list note's new **"Hole-hunting pass"** section. Summary of what landed in the spec (§1/§2/§3/§4):

- **Vocabulary:** **control** = permanent move to controller's board (asleep first turn unless rush/charge), no longer neutral, dies to controller's graveyard; **command** = card-granted one-shot single attack this turn, no zone change, stays neutral.
- **Default attack legality fixed:** `attacker.ownerId == submitter` (own minions only) + new `AttackerNotControlled` — closes a latent bug (old validator let you swing with an opponent-owned minion) and keeps neutrals non-commandable by default (Req 2).
- **Per-lane Taunt** (REVERSES the old "Taunt ignored for neutrals"): constraint scoped to the target's lane; cross-lane never applies. Shared by `AttackAction`/`CommandAttackAction`.
- **Two new actions:** `TakeControlAction` (re-home, asleep-unless-rush/charge, trigger re-registration to new owner's bus list with `birthEpoch` unchanged, aura recalc, `MinionControlChangedEvent`, BoardFull-reject); `CommandAttackAction` (relaxed attacker rule, Windfury=2 strikes each re-validated so first-retaliation-kill fizzles the second, ignores `attacksUsedThisTurn`, desugars to combat `DealDamageAction` pair). Neutral/enemy/either target distinction = **selectors on cards**, no new actions/selectors.
- **Fizzle is the attacker's concern only** — a mortally-wounded *defender* still retaliates (separate `DealDamageAction`, defender as source, not gated by ③). Made explicit in §4 ③.
- **Neutral graveyard:** shared `GameState.neutralGraveyard`; immutable origin flag `MinionOnBoard.bornNeutral` (set only by `SpawnNeutralMinionAction`). §4 ⑦ Phase-1 routing: `ownerId!=null`→player graveyard; `bornNeutral && ownerId==null`→neutral graveyard; `!bornNeutral && ownerId==null`→**undefined/asserts** (deferred → Unaddressed Features; reachable only by a future release-to-neutral / summon-into-neutral path needing an owner-of-record).
- **`GraveyardEntry` refactor** (user's challenge): removed base `originalCard` (derived data that can only drift; `MinionOnBoard` never retains its source Card anyway) → each subtype keeps its snapshot (`GraveyardSpell` gains `definitionKey`+`isInverted`); **card form fabricated lazily at point of use**. Net less spec.
- **Deferred:** temporary/Shadow-Madness control (permanent only); the `!bornNeutral`-dies-in-neutral branch.

**Hole #3 follow-on (2026-06-08): summoning-sickness & Charge/Rush eligibility — DECIDED + APPLIED** (spec §1/§3/§4; full record in borrow-list note "Hole #3 follow-on"). Came out of the user's "does control reuse the summon pipeline?" question. Answer: control ≠ summon (distinct handlers; control preserves state, fires no Battlecry, keeps trigger subs/`birthEpoch`; they share only a thin board-entry routine). The fix: `MinionOnBoard.canAttack: bool` → **`summoningSick: bool`** (raw fact), and §4 ③ eligibility now **pulls** Charge/Rush target-aware (Charge→any; Rush→minion-only, hero rejected `RushCannotTargetHero` [new code]; neither→`AttackerCannotAttack`). Pulling (not a stored verdict) = keyword-collapse philosophy; aura/silence haste just works; shared board-entry routine has zero per-handler charge/rush logic. What would've been "hole #5" (Rush vs `canAttack` bool) is therefore **resolved, not deferred**. **Refinement 2.a — APPLIED (2026-06-08):** `TakeControlAction` trigger wording softened to "re-bucket by owner (bus orders by host owner vs. active player); no `OnSummon` re-run — triggers neither re-created nor re-fired; `birthEpoch` preserved."

**Attack-budget fix — APPLIED (2026-06-08):** the user's "could a minion have `attacksAllowedThisTurn = 2` without Windfury?" exposed a §1↔§3 contradiction (stored vs. derived) where the stored reading had a silence-desync bug. Resolved like `summoningSick`: **`attacksAllowedThisTurn` field DROPPED**; budget pulled at §4 ③ (`attacksUsedThisTurn < (windfury ? 2 : 1)`). Can't have budget 2 without an effective Windfury-family keyword; silence drops it instantly. `attacksUsedThisTurn` is now the only stored attack-state. Deliberate non-Windfury multi-attack = a budget-granting keyword; temporary "+1 attack this turn" = future Modifier System.

**Still-open holes from the session-5 menu** (#3 + its follow-on done): **#1** `AttackResolvedEvent` vs. the two-action combat model (looks vestigial/contradictory); **#2** fatigue not in the data model (no counter on `PlayerState`, no action/event, yet `HeroMortallyWoundedEvent` lists `fatigue` as a cause); **#4** Stealth-drops-on-attack + Freeze unfreeze-tracking (own-moment keyword behaviours with no home — no `frozenOnTurn`). Resume by picking one, applying 2.a, or continuing the open Q/A.

**State:** all session-5 work (Hole #3, summoning-sickness model, 2.a, attack-budget fix) committed + pushed to `origin/main` (commits `3adc01f`, `fefedb3`, `efd3ce0`, + the attack-budget commit). Spec re-grepped clean: no stale `canAttack` / `attacksAllowedThisTurn` / `originalCard` / "Taunt ignored" language (remaining mentions are deliberate supersede/rationale notes).

---

### ⏹ SESSION STOP (2026-06-06, end of session 4)

**State:** Session 4's spec work is **committed + pushed** — commit `566e17f` on `origin/main` (`git@github.com:vasiliscsc/ccg-game-server.git`): per-action intervention windows (new stage ⑥′), combat-as-two-actions (Option A), and the §1 data-model hygiene pass — all recorded in the three bullets directly above + the borrow-list note's "Comparison point D follow-up" entries. Spec re-grepped twice, internally consistent (no stale window/keyword/combat/`*Declared` language; the four `R1` spec refs are now self-contained). Borrow-list note also swept — clean as a historical log (every superseded claim has an adjacent supersede marker; we deliberately left the one dated `IKeyword.OnApplied` example in Item 3 as history). Implementation still NOT started.

**⚠ Open thread flagged but NOT resolved this session — pick up here if continuing combat:** the **source-displaced re-validation rule**. We established that a card played at the defender's ⑥′ that *bounces/transforms the defender* should make the queued retaliation `DealDamageAction (attacker ← defender)` fizzle — but §4 ③ currently spells out fizzle-on-re-validation only for a dead/displaced **attacker** of an `AttackAction`, not for a `DealDamageAction` whose **source** has left the board. Need to pin: does a `DealDamageAction` re-validate its **source** at ③ on resume and fizzle if the source is gone/displaced? (Target-gone is already covered.) Natural next combat thread; tightens §4 ③ + the §3 interception close-out.

**Resume options (user drives one at a time):**
- **(a)** the source-displaced re-validation thread above;
- **(b)** remaining Fireplace menu — **C Play-requirements** (fully open) or **D game-feel follow-ups** (Secret auto-flavor + secret-zone data model + one-per-name/max/own-turn; window **visibility**; **cost timing**; **marked-for-destruction** save scope; **R2** inversion stat-math). *Window timing is CLOSED.*
- **(c)** the official pre-implementation task — **end-of-pass plan reconciliation** (canonical new-work list in the session-3 stop block below + the borrow-list note), then implementation at **Epic 01 / T1.1**.

---

### ⏹ SESSION STOP (2026-06-05 ~01:50, end of session 3 — ran past midnight from 06-04)

**State:** All 13 borrow-list items resolved; Fireplace points keyword-collapse + B (death cadence) + selector trichotomy + Fork-A + **D (unified reactive/interception, which subsumed point A)** + **D follow-up (intervention-window timing — per-action ⑥′, session 4, 2026-06-06)** all **applied to the spec**. Spec internally consistent (grep-verified — no stale window-opening/`*Declared`/cross-cascade-batching language). Implementation still NOT started.

**Queued for next session — remaining Fireplace comparison points (pick one to walk, user drives one at a time):**
- ~~**A — Predamage**~~ — **CLOSED 2026-06-04** (fully subsumed): all damage routes through `DealDamageAction` (combat enqueues it), auto-hit AoE = one selector-carrying damage action (one declaration → one batched pre-damage window, targets=selector), ④ modifier precedence pinned (base→×mult→−reduce→cap→immune→Divine Shield→apply). Applied to §2A/§3/§4.
- **C — Play-requirements** — non-target board preconditions ("requires ≥2 minions", "requires a damaged friendly target") beyond the target selector. Still fully open.
- **D follow-ups (game-feel, if desired before implementation):** Secret auto-resolve flavor + secret-zone data model + one-per-name/max/own-turn; window **visibility** (hand-live hidden vs telegraphed); **cost timing**. *(Window **timing** now fully resolved 2026-06-06 — per-action ⑥′ + immediate response resolution; batching is per-action, not at-settle. Remaining D follow-ups are flavor/visibility/cost only.)*

**Then (the official pre-implementation task):** the **end-of-pass plan reconciliation** — apply every `Plan impact:` from the borrow-list note to the epic/ticket files under `plans/2026-05-27-ccg-game-logic/`, create the flagged new tickets/epics, update the README index/outline/progress. Then implementation begins at **Epic 01 / T1.1**.

**▶ Next task — Plan reconciliation** (full instructions under "Plan reconciliation — end-of-pass task" in the borrow-list note). Walk every `Plan impact:` list in the borrow-list note and apply the edits to the epic/ticket files under `docs/superpowers/plans/2026-05-27-ccg-game-logic/`, **creating new tickets/epics where flagged**. Known new work (consolidated in the note's reconciliation list):
- `StabilizationAbortReport` telemetry → new Epic 04 **T4.8** (Item 1)
- `ITriggerCondition` library → new Epic 08 **T8.10** (Item 2)
- **NEW EPIC: Bot Support** → `GetLegalActions` + bot logic (Item 5)
- `IRandom` foundation → Epic 01 (Item 6)
- Structured error codes / `SubmitResult` → Epic 01 validator (Item 7)
- Tribe/keyword 4-field model → data-model + auras tickets (Item 8 + follow-on)
- **`ITargetSelector` library → new Epic 08 ticket** beside T8.10 (Item 12); validator target-check becomes selector-membership; cards carry `(selector, cardinality)`
- Replay artifacts → Epic 16: keep event-log replay (client path) + add command/input-log canonical replay (Item 13); ruleset-version stamp flagged for Game Server spec
- **Keyword-model collapse → Epic 01 `IKeyword` + Epic 07 re-scope** (post-pass amendment 2026-06-02, from the Fireplace comparison): `IKeyword { KeywordId }` + role hooks (`IOnDealtDamage`), NO `OnApplied`/`OnRemoved`/subscriptions; Lifesteal/Poisonous = hooks, Windfury = declarative read, Enrage = stage-⑥ recompute, Freeze = turn sweep; aura grants accept any keyword; bus carries only `ITrigger`. Dissolved both Unaddressed-Features walls (aura-granted active keywords gone; dead-source reframed). Spec §3 `IKeyword`/`IEventBus`/Source-Attribution + Unaddressed Features amended.

Then re-verify the README index/outline/progress tracker and resume line, and implementation can begin at **Epic 01 / T1.1**. (Workflow note: reconciliation edits the *plan*, not the spec — the spec is done.)

**Discuss the borrow list from the old Unity project** — `docs/superpowers/notes/2026-05-28-old-project-borrow-list.md` contains 13 discrete spec-refinement candidates harvested from the old proof-of-concept. Each has: what's there, why it matters for our spec, recommendation, and open questions. The user picks one item at a time; each decision either amends the spec/plan or gets rejected, then the notes file records the outcome.

Items, ordered by impact (✅ = resolved this pass):
1. ✅ Stabilization loop iteration cap & wave events — ADOPTED (cap=16, abort-as-NoContest, wave markers, scenario-repro telemetry). Also produced a related amendment: Reborn keyword/charge split (`MinionOnBoard.rebornAvailable`). See borrow-list note for full decision.
2. ✅ Pre-built `ITriggerCondition` singletons — ADOPTED in spec §3 (2026-05-30): condition interface + single-condition library (parameterless singletons + parameterized factories) + `All`/`Any` combinators (`Not` deferred). Made multiplicity explicit (card = handler = N triggers via `definition.triggers`); clarified conditions *gate* while trigger `type` *routes* resolution (Battlecry synchronous in play pipeline, Deathrattle in death-wave Phase 2). JSON encoding left as `DefaultCardHandler` detail. See borrow-list note for full decision.
3. ✅ Snapshot triggers before processing (registration safety) — ADOPTED in spec (2026-05-31) as **per-event snapshot at publish + creation-epoch filter**: `GameEngine.currentActionEpoch` (incremented per action ④); subscribers stamped `birthEpoch`, events stamped `originEpoch`; `Publish` dispatches iff `birthEpoch < originEpoch` so a listener never reacts to the event from the action that created it (incl. its own `MinionSummonedEvent`). Chosen over per-batch threading (heavy: registry copy, contract leak, temporal coupling, boundary definition) — epoch is lighter + more robust + uniform across all entity-introducing events. Invariant added: all subscriptions happen inside action handlers. See borrow-list note for full decision.
4. ✅ Effect-op result chaining + Spell Damage +X — ADAPTED (2026-05-31). Split in two: **(A)** Spell Damage +X adopted as a read-only *pulled* `EffectContext.SpellDamageBonus` snapshotted once at cast; Spell Damage **demoted from active keyword to declarative** (no listener), magnitude in the keyword string (`spell_damage:X`), aura-granted via `grantedKeywords`; v1 sources = intrinsic + aura keyword. **(B)** `Results` ledger / op-chaining **rejected** — the epoch-filtered event bus already does outcome-chaining (consistent with deferred Item 10). Player-scoped modifiers ("+1 all spells this turn", "+3 next spell") **deferred to a new future Modifier System** (`docs/superpowers/notes/2026-05-31-modifier-system.md`) — same shape as **mana-cost reduction**, design once for both; no reserved field now. See borrow-list note for full decision + Plan impact.
5. ✅ `GetLegalActions(playerId)` API — ADOPTED (2026-05-31), seam now / enumerator deferred. **Locked in spec (§4 ③):** validation (②③) is a pure, side-effect-free, standalone-invokable predicate `(action, state) → Ok | Rejection` = single source of truth for legality; `Submit` and any "what's legal" query call the same predicate. **`GetLegalActions` itself deferred to a NEW future "Bot Support" epic** (lives in `CCG.GameLogic`, brute-force-then-filter, phase-aware, decoupled from Item 12) — bots are confirmed-coming, so deferred not rejected. See borrow-list note for full decision + Plan impact.
6. ✅ Deterministic `IRandom` interface — ADOPTED (2026-06-01) as **counter-based per-action reseed**: one match seed `GameState.rngSeed`, no long-lived PRNG; each action derives a fresh `IRandom` from `mix(rngSeed, currentActionEpoch)` (reuses Item 3's epoch, now moved into `GameState`). Makes any single action reproducible from `(seed, epoch, preActionState)` with no PRNG-state serialization → `StabilizationAbortReport` self-contained. Seed hidden by architecture (client never sees `GameState`); one-per-game (per-player moot). **Deck shuffle at init via Fisher-Yates over `IRandom`** — decklists stored, shuffled order regenerates. Replay = seed + decklists + player-action log (system actions/rolls/epoch regenerate). See borrow-list note for full decision + Plan impact (new Epic 01 `IRandom` ticket).
7. ✅ Structured error codes — ADOPTED (2026-06-01). `ActionRejectionCode` (closed enum) + `record ActionRejection(Code, string? Detail = null)`; `Detail` is free-form, **logs-only, non-contractual** (tests assert on `Code`, clients localize from `Code`). **Typed per-code payloads rejected as premature** (reconstructable from `(action, state)`; no consumer blocked; keeps upgrade path). **Delivery = return value only** — never a `GameEvent`/on the bus/in the log (rejected actions re-reject on replay). `Submit` now returns `SubmitResult = Accepted(events) | Rejected(rejection)`. Completes the Item 5 predicate's `Ok | Rejection`. See borrow-list note for full decision + Plan impact.
8. ✅ Card definition hooks — ADOPTED SELECTIVELY (2026-06-01) on the line "does logic read it, or only the client?" **Tribes → `[Flags] enum Tribe`** (not string[]: keywords are strings because they resolve to `IKeyword` behavior; a tribe is an inert label → closed designer taxonomy → enum; `[Flags]` gives cheap multi-tribe + granting via `|`/`&`). **Tribeless allowed** (`Tribe.None = 0`, no special-casing). **Granting** = three minion fields `intrinsicTribes`(survives Silence) / `grantedTribes`(Silence→None) / `auraTribes`(recomputed each pass) → `effectiveTribes` (OR). `AuraEffect` gains `GrantedTribes`. **Presentation (art/description/sfx) OUT of the library** — client resolves from Platform-API catalog by `definitionKey`; `rarity` stays (gameplay-relevant). See borrow-list note for full decision + Plan impact.
   - **Follow-on (2026-06-01): keywords expanded to the same 4-field model.** Triggered by a consistency question; exposed an Item 4 bug (aura `spell_damage` was routed into bounce-retained `grantedKeywords`). Now `MinionOnBoard`: `intrinsicKeywords`/`grantedKeywords`(permanent, bounce-retained)/`auraKeywords`(recomputed, declarative-only) → effective `keywords`. **Silence asymmetry vs tribes:** keywords Silence clears intrinsic+granted; tribes clears granted only. `AuraEffect` gained `GrantedKeywords`. Item 4 corrected (aura spell_damage → `auraKeywords`). **Active keywords can't be aura-granted** (would subscribe during ⑥, violating Item 3) → recorded in new **Unaddressed Features** spec section (registry for *indefinitely*-deferred features, distinct from future-epic deferrals).
9. ✅ Nested `EffectContext` source-attribution — ADOPTED as clarification (2026-06-01). Plumbing already existed (actions + key events carry `sourceId`; Item 2 conditions key off it). Four locks: (1) **`sourceId` = the entity whose effect this is, set by the enqueuer, never inherited from the upstream cause** — Yeti's deathrattle damage is sourced to Yeti, not the spell that killed it; trigger `OnFire` stamps host as source; `sourcePlayerId` = source's controller (death-snapshot if dead). (2) **`EffectContext.sourceCardId` → `sourceId`** (unified ref; old field couldn't express a minion source). (3) **Lifesteal listener corrected** — fires on damage *caused by* self (`evt.sourceId == me`), not "on self" (was backwards). (4) **Dead-source active keywords don't fire** — attribution (event data) ≠ keyword application (live listeners, gone at Phase 1); → **Unaddressed Features**. See borrow-list note for full decision + Plan impact.
10. ✅ Op `Results` ledger — **REJECTED for v1** (2026-06-01), both halves. Gameplay op-chaining → event bus (Item 4). Observability/tooling ledger → need served by deterministic replay-with-tracer (Item 6) + event log; not withheld, not blocked, trivially additive later → **not** an Unaddressed Feature. `IEffect.Execute` stays `void`. No spec change. (Established the disposition test: reject vs Unaddressed-Features vs future-epic-deferral.)
11. ✅ Game-engine builder pattern — **NOT A SPEC CONCERN** (2026-06-01); implementation detail, home = plan scenario builder T1.7 (production init = seed + setup actions per Item 6; tests = direct `GameState` construction). No spec change. Plan impact: T1.7 must now cover `rngSeed`/`currentActionEpoch`, seed injection, and the new tribe/keyword fields from Items 6–9.
12. ✅ Targeting strategy registry — ADOPTED (2026-06-02) as **one pure `ITargetSelector` primitive**, dual of `ITriggerCondition` (singletons + factories + `Filter` reusing conditions). Pure function of `GameState`, ordered by §4 ⑦ board order. **Three consumption modes** from one candidate set: auto-hit (AoE), random-K (pool-carrying action drawn at stage ④), player-choice (feeds existing `StartChoiceAction`/`PendingChoice`). **Player-target legality unified** into the same primitive — §4 ③ validity = `chosen ∈ selector.Select(ctx)` (+ cardinality); no separate `TargetRequirement` taxonomy. **RNG (Item-6 parked Q) → Fork A:** selectors are pure, do NOT get `IRandom`; random draw stays a stage-④ concern (Item-6 invariant verbatim). Note's `RequiresChoice` selector-return variant rejected (keeps compute-targets vs interrupt-for-input separate). Spec §3 amended. See borrow-list note for full decision + Plan impact (new Epic 08 `ITargetSelector` ticket).
13. ✅ Command-log vs event-log replay — ADOPTED (2026-06-02): confirms Item 6's model and **tightens the spec §3 Replay paragraph** with three gaps — (1) an *input* = any `Submit`-ed action **incl. timeout/forfeit/disconnect-injected** ones (external nondeterminism, must be logged; timer is a Game Server concern); (2) **ruleset-version pinning** (command-log replay valid only vs same engine + card-def version); (3) **command log = canonical / event log = client wire format + archive**, event-log derivable from command-log+seed but not vice versa; single total-order input stream (not per-player). Disposition: SPEC (tighten) + Plan-impact (Epic 16). See borrow-list note for full decision.

**All 13 borrow-list items are resolved.** The spec is fully refined. Remaining before implementation: the **end-of-pass plan reconciliation** (see resume point above + the borrow-list note). Then implementation begins at **Epic 01 / T1.1**.

**Spec-first workflow (2026-05-31):** the borrow-list pass amends the **spec only**; epic/ticket files are reconciled in one pass at the end. Each `✅ DECISION` in the borrow-list note carries a **Plan impact:** list (affected epics/tickets + flagged new tickets/epics) so the end reconciliation is mechanical. The end-of-pass reconciliation task (apply Plan-impact edits; **create new tickets/epics where flagged** — e.g. `StabilizationAbortReport` telemetry, `ITriggerCondition` library; update README index/outline/progress) is written up under "Plan reconciliation" in the borrow-list note. New epics/tickets may be created freely so each finalized amendment has a home.

**Do NOT start implementation** while the borrow-list pass is in progress. The user is driving the refinement.

**Key decisions locked during planning:**
- .NET 10 (current LTS), xUnit + FluentAssertions; records for actions/events, classes for state.
- `GameEngine.Submit(action)` returns the full event list; `EventBus` is independently inspectable.
- Trigger fire order = current player first, then by **board index at publish time** (not summonOrder, though summonOrder is kept for disambiguation).
- Death wave: Phase 1 remove → Phase 2 deathrattles → Phase 3 reborn; new deaths deferred to next wave.
- `Card.id` and `MinionOnBoard.minionId` are independent allocators; `definitionKey` is the only cross-entity link.
- `MinionToCardPolicy` (Stripped / RetainEnchantments) parameterises minion→card transitions; `isInverted` always survives both.

---

## Decided feature scope (locked in brainstorming)

### Card types
All four: **Minion, Spell, Weapon, Hero Power**. Hero/class is tied to the deck (like Hearthstone).

### Keywords (all selected, system must be extensible)
Taunt, Divine Shield, Charge, Rush, Lifesteal, Windfury, Poisonous, Stealth, Spell Damage +X, Reborn, Enrage, Freeze — plus any future keywords added via new `IKeyword` implementations.

### Trigger types (all selected, system must be extensible)
Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous), On Damage Taken, On Friendly Minion Death, On Spell Cast, Inspire, Combo, On Invert — plus any future triggers added via new `ITrigger` implementations.

### Inversion mechanic
- A card or minion can be in **Normal** or **Inverted** state
- Stats flip: Attack ↔ Health when inverted
- Trigger type can change: e.g. Battlecry → Deathrattle when inverted
- Effects are individually defined per card (normal and inverted sections in definition JSON)
- Reversible (can be inverted and un-inverted multiple times)
- Applies to cards in hand AND minions on board
- Triggered by: card effect/spell, player-initiated action, on draw/zone entry, opponent action

### Choice / interactive mechanics
- **Mulligan** — pre-game hand selection
- **Discover** — present 3 options, player picks 1
- **Mid-effect targeting** — pause mid-resolution for player to select a target
- **Card crafting** — multi-step interactive sequence producing a custom card
All require a `PendingChoice` game phase.

### Intervention system
When an opponent declares an action, the other player gets a **single-response window** (play one card / skip) before the action resolves. Exact trigger conditions TBD in Game Server spec. Requires `PendingIntervention` game phase.

### Neutral zone
- Third board zone between the two player boards — neither player owns it
- Neutral minions have no turn of their own
- Players can: attack neutral minions, **command** them to attack (card-granted one-shot, no zone change), **control** them (card-granted permanent move to own board) — see session-5 (2026-06-07) for the full control/command model
- Taunt keyword is **per-lane** (amended 2026-06-07; was "ignored for neutral minions"): a neutral Taunt forces attacks aimed into the neutral lane, but not into the opponent lane, and vice versa
- Player auras do **not** affect neutral minions (while they are neutral; a controlled minion is owned and gets its controller's auras)
- Neutral minions enter via: card effect, pre-populated at game start, event trigger
- Zone has a **max capacity** and may **repopulate on turn start** (configured per game mode)

### Architecture pattern
**Event Bus + Component System** — Actions (Commands) → Events → State updates.
- New keyword = new `IKeyword` class
- New trigger = new `ITrigger` class
- New effect = new `IEffect` class
- Custom card logic = new `ICardHandler` that registers triggers into the bus
- Auras = `IAura` recalculated after every board change

---

## Approved data model (Section 1 — approved)

### GameState
```
sessionId: string
player1: PlayerState
player2: PlayerState
neutralZone: MinionOnBoard[]          // ownerId = null; Taunt ignored; auras don't apply
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone this game mode
turn: TurnState
timer: TimerState
phase: GamePhase                      // Mulligan | WaitingForPlayers | InProgress | PendingChoice | PendingIntervention | Ended
winnerId?: string
pendingChoice?: PendingChoice
pendingIntervention?: PendingIntervention
mulliganState?: MulliganState
```

### NeutralZoneConfig
```
maxCapacity: int
spawnPool: Card[]
repopulateOnTurnStart: bool
```

### PlayerState
```
playerId, heroClass: string
health, mana, maxMana, heroAttack: int
hand: Card[]
deck: Card[]
board: MinionOnBoard[]        // player's own side only
graveyard: GraveyardEntry[]   // unified — minions + spells + weapons
weapon?: WeaponOnHero
heroPower: HeroPower
heroPowerUsedThisTurn: bool
cardsPlayedThisTurn: int      // Combo tracking
```

### MinionOnBoard
```
minionId, cardId: string
ownerId: string?              // null = neutral zone
baseAttack, baseHealth: int   // immutable, from card definition
enchantments: StatModifier[]  // permanent buffs — Silence clears
auraAttackBonus, auraHealthBonus: int  // recalculated, never in enchantments
attack: int                   // = baseAttack + Σenchantments.attackDelta + auraAttackBonus
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus
currentHealth: int            // takes damage; healed up to maxHealth
keywords: string[]            // resolved to IKeyword at runtime; Silence clears
isInverted: bool
canAttack: bool
attacksUsedThisTurn: int      // Windfury allows 2
isFrozen, isDamaged: bool
```

### Card
```
id, name: string
type: CardType                // Minion | Spell | Weapon | HeroPower
rarity: CardRarity
baseManaCost: int
baseAttack?, baseHealth?: int // Minion / Weapon only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // has "normal" and "inverted" sections
handlerKey?: string
isInverted: bool
```

### StatModifier (unified — used on both Card and MinionOnBoard)
```
sourceId: string
costDelta: int      // meaningful on Card only; ignored on board
attackDelta: int
healthDelta: int
```

### GraveyardEntry hierarchy
```
GraveyardEntry (abstract)
  originalCard: Card
  turnPlayed: int

GraveyardMinion : GraveyardEntry
  snapshot: MinionOnBoard   // full state at death
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  // base fields sufficient

GraveyardWeapon : GraveyardEntry
  weaponState: WeaponOnHero
  destroyedOnTurn: int
```

### Supporting types
```
WeaponOnHero    — cardId: string, attack: int, durability: int
HeroPower       — cardId: string, manaCost: int, definition: JsonElement, handlerKey?: string
TurnState       — activePlayerId: string, number: int
TimerState      — secondsRemaining: int
PendingChoice   — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[], context: JsonElement
PendingIntervention — respondingPlayerId: string, heldAction: GameAction, timeoutSeconds: int
```

---

## Key files

| File | Purpose |
|---|---|
| `docs/superpowers/specs/2026-05-25-ccg-system-design.md` | System design doc (approved) |
| `docs/superpowers/plans/2026-05-26-game-server.md` | Game Server plan — **SUPERSEDED, do not execute** until new mechanics spec is written and a new plan is created |
| `docs/brainstorm/` | Brainstorm HTML artifacts |
| `.superpowers/brainstorm/` | Visual companion session files |
