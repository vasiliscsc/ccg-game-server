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

**Hole #1 — DECIDED + APPLIED (2026-06-08):** `AttackResolvedEvent` was vestigial (atomic-combat shape, referenced nowhere). Removed it; added **`AttackPerformedEvent`** (`AttackAction` ④, committed/post-interception swing, no damage numbers — the two `DamageTakenEvent`s carry those) and **wrote the `AttackAction` ④ handler** (snapshot base attack → `attacksUsedThisTurn++` → break Stealth → emit `AttackPerformedEvent` → enqueue strike + conditional retaliation `DealDamageAction`s). `AttackDeclaredEvent` stays as the ③′ intended/pre-interception declaration. **Per user request, folded in Stealth + unfreeze:** Stealth now **breaks on attack** (`StealthBrokenEvent`, new — **closes the Stealth half of hole #4**); Freeze noted (frozen attacker rejected at ③; `attacksUsedThisTurn` feeds the end-of-turn unfreeze sweep). Full record in borrow-list note "Hole #1".

**Hole #4 (freeze half) — DECIDED + APPLIED (2026-06-08):** found TWO bugs — the lifecycle unfroze *after* the swap (start of controller's turn → negated Freeze), and my hole-#1 note had the thaw condition inverted. Fixed with the HS "Freeze costs exactly one attack" rule: new `MinionOnBoard.frozenOnTurn` (stamped by `FreezeTargetAction`); unfreeze sweep **moved to end of the ending player's turn, pre-swap** (Turn Lifecycle step 3); thaw unless `frozenOnTurn == turn.number && exhausted`; new `MinionThawedEvent`. Water-Elemental case verified. **Hero freeze deferred** (recorded in Unaddressed Features — needs `PlayerState` fields + hero events + ③ wiring, belongs with hero-combat pass). **Hole #4 now fully closed** (Stealth half via #1, Freeze half here). Full record in borrow-list note "Hole #4 (freeze half)".

**Still-open holes** (#1, #3+follow-on, #4 done): **#2** fatigue not in the data model (no counter on `PlayerState`, no action/event, yet `HeroMortallyWoundedEvent` lists `fatigue` as a cause). *(Closed in session 7, 2026-06-09 — see below.)* **Recorded deferrals:** temporary control; `!bornNeutral`-dies-in-neutral routing; hero freeze. Resume by picking #2, a deferral, or continuing the open Q/A.

### Session 6 (2026-06-09): intervention multi-card resolution — MTG-style re-declare/re-offer loop (DECIDED + APPLIED)

Continued the open Q/A hole-hunting, **revisiting point D (interventions)**. The user probed: what if a player holds **multiple** intervention cards matching the **same** event? The ③′ procedure was written in the singular ("the single-response window") with no multi-match rule — an underspecified hole (⑥′ had "one window per matched card, hand order"; ③′ had no parallel). Full record + Plan impact in the borrow-list note's **"Hole #6"**. Summary of what landed in the spec (§1/§2/§3/§4):

- **MTG-style loop, both phases.** Window is **set-valued** (`PendingIntervention.candidateCardIds` = matching + **affordable** reactive hand cards); player plays one (`SubmitInterventionAction { chosenCardId?, targetIds[]? }`) or skips. **PRE = re-declare loop** (held action re-published to ③′ after each play — **reverses the old "declared once / not re-published" rule**; fizzle on re-validation ends the loop and gives sibling "withdrawal" for free). **POST = re-offer loop** (no held action; re-offer remaining matched set against frozen events, lapsed-condition cards drop out).
- **Skip terminates; only a play re-opens** (a skip changes nothing → re-presenting the same set is pointless). Skip does **not** consume the card (stays armed for future events). Replaces the old ⑥′ per-card-in-hand-order rule; replay determinism = logged choices.
- **Iteration ≠ nesting.** Depth-1 **kept**, but caps **nesting only** (a response can't be intervened). **Iteration** (one responder chaining their own cards on one action) is uncapped by count — different axis, self-bounding (played cards leave hand).
- **Termination = mana, no hard cap** (user's call). Pay-on-response from `mana`; only residual is 0-cost-begets-0-cost (Item-1 stabilization cap is the backstop).
- **Mana model — user's simplification (replaced my `reservedMana` proposal mid-edit).** **Moved the mana ramp+restore from turn-START to turn-END.** The refreshed pool sits through the opponent's turn; interventions spend straight from `mana` (ordinary `effectiveCost ≤ mana`); your next turn begins with the remainder. **No `reservedMana` field, no special ceiling**; ceiling = full ramped next-turn pool. Outcome-identical to HS on your own turns — only bookkeeping moved. Caveats in-spec: game-init seeds turn-1 mana; future overload applies at this turn-end refresh.
- **Worked-example answer** (user's "already-queued actions" question): a strike's ⑥′ responses resolve **immediately, ahead of** the already-queued retaliation + the attacker's draw — the ⑥′ window is a checkpoint inside the cascade at the triggering action's position, not behind the queue (`spec` §4 "Response resolution"; combat-as-two-actions per-hit reaction, `spec:698`).

**Spec edits APPLIED** across §1 (`PendingIntervention.candidateCardIds`; no `reservedMana`), §2 (`SubmitInterventionAction`/`StartInterventionAction`), §3 (pillar-2 pre/post, ordering-table Interception + Post-reaction rows, hand-zone row), §4 (③′ re-declare loop + depth-1-nesting reword, ⑥′ re-offer loop, Response-resolution loop continuation, PendingIntervention Interruption full rewrite incl. Batching's two granularities, **Turn Lifecycle mana refresh moved to step 4 / turn-end**). Re-grepped clean (no stale `single-response` / `declared once` / `not re-published` / `one window per matched card` / `reservedMana`; the two remaining matches are the intentional supersede note and the combat-atomicity rationale).

**Session-6 follow-up (same day): secret-vs-intervention worked example → two refinements.** User probed: A holds an active **secret** "before the hero takes damage, add 8 armor"; B holds an intervention "when a friendly minion takes damage, deal that much to the enemy hero"; A attacks B's minion. Resolution: B's ⑥′ reflect resolves immediately (ahead of the queued retaliation); the reflect's ③′ triggers **A's secret inline** (autos are **not** capped by depth-1 — only *player windows* are), so armor absorbs the hit. **(a) Depth-1 wording tightened** in 4 spots — "a response cannot itself be intervened" over-claimed (an auto/secret *can* intercept a response's sub-action); now scoped to "no further *player window*; auto/secret hosts still fire — and may intercept — inline." **(b) Hero armor logged as a gap:** `armor` is **not** in `PlayerState` (only an interception *example* in §3); added an Unaddressed-Features "Hero armor" entry (needs `PlayerState.armor` + `GainArmorAction` + ④ precedence slot) deferred to the **hero-combat pass** — the consume-armor mechanics already resolve once the field exists.

### Session 7 (2026-06-09): hole #2 — fatigue (DECIDED + APPLIED)

Closed the **last open hole from the menu**. The empty-deck draw was unspecified and `HeroMortallyWoundedEvent` carried a dangling `fatigue` cause. Full record + Plan impact in the borrow-list note's **"Hole #2"**.

- **Model (HS-exact):** `PlayerState.fatigueCounter: int` (init 0, never resets); empty-deck draw does `++` then deals the new value (1,2,3,…). **Routed through `DealDamageAction`** (forced by the locked all-damage-through-one-channel rule) → inherits ④ precedence (future hero-armor slot), interception/⑥′ windows, and the hero→`HeroMortallyWoundedEvent`→⑧ settle path (deck-out loss + mutual-deckout draw free). `sourceId = null` (sourceless).
- **`DrawCardAction` handler** spelled out, three branches (draw / overflow-burn / fatigue). New **`FatigueDamageEvent { playerId, amount }`** (fires before the damage lands).
- **The real fork — `cause` discriminator: DROPPED (user's call, YAGNI).** It sat on both mortally-wounded events but nothing populated it and no card branches on it. Removed from `HeroMortallyWoundedEvent` → `{playerId, sourceId}` and `MinionMortallyWoundedEvent` → `{minionId, sourceId}`; fatigue is identified by its preceding `FatigueDamageEvent`. Also reworded §6 ⑥ ("cause = aura loss" → "an aura-loss death"), the one place a cause value was written. Rejected the full `DamageCause` taxonomy (real hot-path surface for an informational field).
- **Same-pass cleanup:** fixed stale §2B `ActionDeclaredEvent` text ("fires once / not re-declared") that contradicted the session-6 re-declare loop — a session-6 grep miss. Doc consistency only.

**Spec edits APPLIED** across §1 (`fatigueCounter`), §2A (`DrawCardAction` handler), §2B (`FatigueDamageEvent`; `cause` removed ×2; `ActionDeclaredEvent` reworded), §6 ⑥ (aura-loss reword). Re-grepped: no `cause =` field-writes remain; no "fires once"/"not re-declared".

**Open holes: NONE remain from the hole-hunting menu** (#1, #2, #3+follow-on, #4 all done). **Recorded deferrals unchanged:** temporary control; `!bornNeutral`-dies-in-neutral routing; hero freeze; hero armor; hero-combat pass; the 0-cost-intervention-loop residual.

**Then started the QUEUED TOPICS. Queued topic #1 — trigger-order = deathrattle-order — RESOLVED + APPLIED (2026-06-09, session 7).** The hole was real: `IEventBus` kept only two subscriber lists (current player, opponent) and never mentioned neutral, while death sort ended in neutral — and the selector row already assumed a unified order the trigger side didn't state. Unified into **one canonical board order** (active `board[0..n]` → opponent `board[0..n]` → neutral by index), used identically by trigger dispatch (snapshot at publish), death/deathrattle/reborn sort (snapshot at collect), and selectors; `summonOrder` the fallback; heroes only via selectors. Pinned **neutral's slot last** on the trigger side (`IEventBus` → three dispatch groups); **de-duplicated** the two Ordering-table rows into one. **Design call:** neutral fires **board-wide event** triggers (last) but **not turn-scoped** Start/End-of-Turn (no turn of its own — falls out of controller-scoped condition semantics; Turn Lifecycle steps 2/10 now state the exclusion). Spec edits: §3 `IEventBus` (three groups + the event-vs-turn-scoped distinction) + `ITargetSelector` cross-ref; §4 ⑤ Publish + Turn Lifecycle 2/10; Deterministic-Ordering table merged row. Full record + Plan impact in the borrow-list note's "Queued topics" #1.

**Queued topic #2 — neutral-lane auras → generalized to a LANE-BASED friendly/enemy model — RESOLVED + APPLIED (2026-06-09, session 7).** Began as "do neutral minions emit auras"; the user's probe ("a neutral minion's friendlies are its lane-mates") surfaced that the real question was the **central** `friendly`/`enemy` definition shared by auras, triggers, AND selectors — which was controller-based and left neutral empty. **North-star (user): the neutral lane should behave as close to a regular lane as possible so mechanics work out of the box.** Landed: (a) **lane-based friendly/enemy, formalized by the user** — `friendly(h,e) ⟺ h.ownerId==e.ownerId` (null==null → neutral host's friendlies = lane-mates); `enemy(h,e) ⟺ h.ownerId!=e.ownerId && e.ownerId!=null` (the `!=null` clause makes a neutral minion never anyone's *enemy*; player host's enemy excludes neutral = the 3rd category, neutral host's enemy = both boards; player-host case is HS-identical). (b) **Auras fall out** — player friendly aura excludes neutral, neutral's friendly aura buffs lane-mates, reaching the lane from a player aura needs an explicit `AllNeutralMinions`/`AllMinions` selector, `AdjacentTo` per-lane, no hardcoded wall. (c) **Triggers inherit it** — neutral `FriendlyOnly` fires off lane-mates; **corrected my topic-#1 overstatement** (only turn/hero-context conditions — Start/End-of-Turn, Inspire, Combo — stay empty for neutral; the friendly *relation* resolves). (d) **Summon-event consolidation** — neutral spawn now emits the regular **`MinionSummonedEvent` with `ownerId==null`** via the same board-entry routine; **removed `NeutralMinionSpawnedEvent`** (it was the bug); `NeutralZoneRepopulatedEvent` demoted to a renderer-only batch marker. Spec edits: §1 (neutralZone comment), §2A (`SpawnNeutralMinionAction`), §2B (`MinionSummonedEvent` strengthened, `NeutralMinionSpawnedEvent` removed, `NeutralZoneRepopulatedEvent` reframed), §3 (`ITriggerCondition` formal predicates + `FriendlyOnly`/`EnemyOnly` rows, `ITargetSelector`, `IAura` scope note, `IEventBus` topic-#1 note corrected), §4 ⑥ (every minion incl. neutral). Re-grepped: no stale controller-based phrasing, no `player auras do not apply`, no live `NeutralMinionSpawnedEvent`. Full record + Plan impact in the borrow-list note's "Queued topics" #2.

**Remaining queued topics: #3 debug text format, #4 intervention visibility.**

**▶ RESUME OPTIONS (user drives one):** (a) continue the **queued topics** (#2 neutral-lane auras — natural next, now that ordering is pinned; #3 debug text format; #4 intervention visibility) or call hole-hunting **done**; (b) a **recorded deferral** / the **hero-combat pass** (bundles hero freeze + hero armor + heroAttack/weapon/durability); (c) the **end-of-pass plan reconciliation** → implementation at **Epic 01 / T1.1**.

**Uncommitted:** session-7 spec edits + this state + the borrow-list note are in the working tree (not yet committed — the earlier session-state mirror sync is also uncommitted per the user's "leave it for now").

### Session 8 (2026-06-10): queued topic #3 — debug text format → structured per-action debug trace (DECIDED + APPLIED)

**Re-scoped by the user at the first question:** not a scenario-authoring/parse format — a **read-only diagnostic the engine emits per processed action** (state + queued actions + events) for stepping through a failed test manually. Full record + Plan impact in the borrow-list note's **"Queued topics" #3**.

- **Q&A pinned:** diff per action + full snapshots at cascade boundaries; stages traced **only when they act** (quiet stages invisible).
- **The format fork (sigil ledger / narrative / YAML) was dissolved by the user's reframe:** emit *parsable structured records*; every human-friendly rendering — including an eventual **visualizer** — is a derived view. One schema, not three formats. With raw readability demoted (visualizer coming), encoding landed on **JSON Lines via System.Text.Json** (zero dependency, the stack `JsonElement` already commits to, append-as-it-happens so a mid-cascade crash leaves a valid trace, greppable/diffable) over YAML (YamlDotNet dependency + implicit-typing footguns; its readability edge no longer decisive), XML, and binary.
- **Model:** `ITraceSink { Record(TraceRecord) }`, engine ctor takes `ITraceSink?` (null = zero overhead); emission from *inside* stages (rejections produce zero events — invisible to any event post-processor); **diffs by structural before/after compare** (clone cost only when attached; no mutation escapes unseen); `JsonLinesTraceSink(Stream)` default impl. **Record kinds:** `state` (match start + settle-to-idle; `definition` bodies elided, `definitionKey` only), `action` (per ④), `reject`, `window`/`choice` (opening only; the `Submit*` resolves as its own `action` record), `settle` (per ⑦ wave; ⑧ only when it acts). Conventions: bare ids, dotted diff paths `[from,to]`, zone moves, no `queueAfter` (derivable).
- **Non-goals:** third purely-diagnostic stream (vs command log = canonical, event log = wire); visualizer + pretty-printers **out of library scope** (animation/bots ruling); no authoring DSL.
- **Prior art:** HS `Power.log` (same shape at scale), old-project Spine/Exact traces (structured records enable assertion helpers later), `StabilizationAbortReport.cascadeTrace` = event-only cousin.

**Spec edits APPLIED:** §3 — new **"Debug Trace (`ITraceSink`)"** subsection (after `IRandom`, before Deterministic Ordering): interface, emission model, record-vocabulary table, JSONL worked example, conventions, non-goals. §4 — intro cross-ref (emission points); `StabilizationAbortReport` note (attached sink = strictly richer repro view). No data-model change; no new events.

**Queued topic #4 — intervention visibility (responder's-eye view) — RESOLVED + APPLIED (2026-06-10, session 8).** Opened with a **user-requested prior-art survey** (MTG/YGO/LoR/FaB with responder windows vs HS/Gwent/Snap without): (1) **cause visibility is unanimous** — every game with a responder window shows the pending object in full (hidden identity exists only on pre-armed face-down cards, protecting the *responder's* card, never the actor's pending action); (2) the real split is **structural** windows (MTG paper/LoR — leak-free, slow) vs **contingent** (Arena auto-pass/Master Duel — fast, pause = tell). User locked: **③′ cause = held action verbatim** (per the unanimity, "what the players expect") and **contingent + thin public event** (tell accepted; masking = server/UX option). Landed as **one information ceiling** — a prompt shows exactly the responder's own candidates + the cause, which is public-by-nature (③′ pre-resolution params) or already on their wire (⑥′ → `causeEventIds[]` refs, never re-embedded — an opponent-draw trigger can't expose the card). **Wire:** directed/public split formalized (§2B "Directed events", mulligan precedent): directed `InterventionPromptEvent` (cause + candidates + timeout) / public thin `InterventionWindowOpenedEvent`; **adjacent fix** — `ChoiceStartedEvent` was broadcasting `options[]` (Discover leak) → thinned to `optionCount` + new directed `ChoicePromptEvent`. **Data model:** `candidateCardIds` → **`candidates: InterventionCandidate[] {cardId, targetIds[]}`** (engine-computed targets; single source for ③-validation + the prompt; mirror block re-synced); new **`eventId: long`** on all events (global append-order ref). Armed-state telegraph + secret identity/cost/mark-scope stay deferred. Spec edits: §1, §2A (`StartInterventionAction` enriched, `SubmitInterventionAction` validation), §2B (preamble + directed paragraph + 5 event rows), §3 (visibility block + deferred-list reword), §4 (③′/⑥′/Interruption). Re-grepped: no stale `candidateCardIds`. Full record + Plan impact: borrow-list note "Queued topics" #4.

**Queued topics: ALL FOUR RESOLVED (#1–#4). The hole-hunting/probing pass is fully drained.**

**Session 8 part 2 (ran past midnight → 2026-06-11): HERO-COMBAT PASS started — and mid-pass the user pivoted the hero kit: ARTIFACTS replace the hero-power system (DECIDED + APPLIED).** First lock of the pass proper: **hero defenders never retaliate** (HS-faithful). Then, at the `heroAttack` question, the user introduced a stakeholder-driven spec change: no unique hero powers — a new **Artifact** system (per-player row, cap 3 incl. a shared **starter** "pay mana, draw" with escalating per-turn cost; passive = triggers/auras, active = cost+target+effects, or both; ONE durability counter with definition-declared consumers; generic `charges`; in-turn free discard; reject-at-③ on full row). Full hero-power demolition: `heroPower`/`heroPowerUsedThisTurn`/`HeroPower` type/`UseHeroPowerAction`/`HeroPowerUsedEvent`/`InspireTriggeredEvent` all removed; CardType gains `Artifact`; Inspire → **`OnArtifactActivated`**. Trigger ordering: artifacts append **inside their owner's dispatch group after that owner's minions** (slot order) — preserves active-first, no fourth global group. Discard vs destroy = **disjoint event types** (the session-7 no-`cause`-enum idiom). §1 mirror blocks re-synced (PlayerState, ArtifactOnBoard, GraveyardArtifact, CardType, locked-scope card-types + trigger list). Full record + Plan impact: borrow-list note **"Hero-combat pass" Part 1**. **Pass still open: heroAttack semantics, weapon wiring, hero armor, hero freeze.**

### ⏹ SESSION STOP (2026-06-11, end of session 8 — ran past midnight from 06-10)

**State:** ALL session-8 work committed + pushed to `origin/main` — working tree clean. Commit trail (chronological):
- `9bf7557` — queued topic #3: debug trace (`ITraceSink`, JSON Lines; structured records, visualizer = derived view)
- `de3590f` — queued topic #4: intervention visibility (held-action-verbatim ③′ cause per unanimous prior art; contingent windows + thin public event; directed/public wire split incl. the `ChoicePromptEvent` Discover-leak fix; `candidates: InterventionCandidate[]`; `eventId`)
- `7718215` — **ARTIFACTS replace the hero-power system** (user pivot mid hero-combat pass) + the no-hero-retaliation lock
- `<this commit>` — session-8 stop block.

**What session 8 was:** closed the last two queued topics (#3, #4 — probing pass fully drained), then **started the hero-combat pass**, which immediately produced the session's headline: the user replaced HS-style hero powers with the **Artifact system** (per-player row cap 3, shared escalating-cost starter draw-artifact, passive/active/both, one durability counter with declared consumers, charges, in-turn discard; full hero-power demolition incl. Inspire → `OnArtifactActivated`). Per-decision records: borrow-list note "Queued topics" #3/#4 + "Hero-combat pass" Part 1.

**▶ RESUME POINT — the hero-combat pass is MID-FLIGHT.** Already locked: **hero defenders never retaliate** (HS-faithful); artifacts DONE. **The very next step is the open `heroAttack` question** (asked twice, user deferred both times — re-pose on resume): temporary this-turn bonus (recommended; effective swing = `heroAttack + weapon.attack`, attack iff > 0, cleared at controller's turn end) vs persistent stat. Then the rest of the pass, in rough order:
1. **Hero-as-attacker wiring** — `AttackAction` with a hero `attackerId`: ③ checks get hero-appropriate reads (control = own hero; aliveness = health > 0; frozen → hero fields, below; budget → new `heroAttacksUsedThisTurn` on `PlayerState`, pulled budget = weapon Windfury → 2 else 1 — weapon keywords question rides here: lean = WeaponOnHero gains `keywords: string[]` seeded from definition, hero swing's keyword pulls read the weapon); heroes are never summoning-sick; equip-then-swing same turn is legal.
2. **Durability consumption** — hero swing ④: `durability--`, `WeaponDurabilityLostEvent`, at 0 → `DestroyWeaponAction` (→ `GraveyardWeapon`). Defending NEVER spends durability (locked). Equip-over-existing destroys the old weapon first.
3. **Hero armor** — `PlayerState.armor: int`, `GainArmorAction` + `ArmorGainedEvent`; pin the ④ precedence slot (the Unaddressed-Features entry sketches it: absorb after flat reductions/caps, before health application; immune still short-circuits; DamageTakenEvent likely gains an `armorAbsorbed` field — decide); fatigue then auto-absorbs (the §2A DrawCardAction row already anticipates this).
4. **Hero freeze** — `heroIsFrozen`/`heroFrozenOnTurn` on `PlayerState`, ③ reads the character-appropriate fields, lifecycle step-3 thaw sweep already says "each frozen character" (exhaustion test → `heroAttacksUsedThisTurn`), `HeroFrozenEvent`/`HeroThawedEvent`. The Unaddressed-Features entry says the rule applies unchanged — wiring only.
Then close the pass: delete the two Unaddressed-Features entries (freeze/armor), record in the note's "Hero-combat pass" section, sweep `heroAttack`-stale text.

**After the pass:** the only remaining pre-implementation item is the **end-of-pass plan reconciliation** (now also: re-scope any hero-power tickets to artifacts; sweep `candidateCardIds` → `candidates`, `eventId`, trace ticket, directed events) → then implementation at **Epic 01 / T1.1**.

### Session 9 (2026-06-11): hero-combat pass CLOSED — the hero-weapon concept is DROPPED, armor lands (DECIDED + APPLIED)

Resumed at the re-posed `heroAttack` question (temporary-vs-persistent). **The user pivoted past the fork: drop the whole concept** — heroes can't attack, can't be frozen, have no equippable weapons; *"the Artifacts we scoped are sufficient"* as the hero kit. That dissolved pass items 1/2/4 (hero-as-attacker wiring, durability, hero freeze). The one genuine residual fork — **hero armor**, which is defense and so survives the offense demolition — went to **keep, land now** (recommended; its "rides the hero-combat path" deferral reason evaporated with the path).

- **Demolition:** §1 `heroAttack` + `weapon?` + `WeaponOnHero` + `GraveyardWeapon` REMOVED; `CardType` → `Minion | Spell | Artifact` (no weapon cards); §2A `EquipWeaponAction`/`DestroyWeaponAction` + §2B the 3 weapon events REMOVED. Freeze ruling (flagged, no real fork): **heroes cannot be frozen** — freeze denies attacks, heroes have none; `FreezeTargetAction.targetId` is a minionId; "frozen character" → "frozen minion" swept.
- **Armor:** `PlayerState.armor` (init 0, no cap), `GainArmorAction`/`ArmorGainedEvent`; ④ precedence step (6) generalized to the **target-typed absorb slot** (minion → Divine Shield all-or-nothing; hero → armor partial absorb); **`armorAbsorbed` = a field on `DamageTakenEvent`** (the `overkill` one-occurrence-split-delta idiom, not the disjoint-event idiom). Fatigue auto-absorbs; the §3 "gain armor" interception pillar + session-6 armor-secret example now fully backed.
- **Part-1 lock now structural:** hero defenders never retaliate because retaliation is gated on defender attack > 0 and heroes have none. Heroes stay ordinary attack *targets*.
- Both Unaddressed-Features entries (hero freeze / hero armor) replaced by one RESOLVED record. Spec re-grepped clean (no stale `weapon`/`heroAttack`/`frozen character`/`hero-combat` outside intentional decision notes). §1 mirror block re-synced (provenance updated). Full record + Plan impact: borrow-list note **"Hero-combat pass" Part 2**.

**HERO-COMBAT PASS CLOSED.** (Commit `8b46ddf`.)

**Session 9 part 2 (same day): deferral scan → SIGILS — the unified reactive-card model (DECIDED + APPLIED).** User asked for a sweep of deferred-but-never-done items; the scan found: (1) the **secret bundle** (biggest), (2) **card crafting** (locked scope, zero spec presence, recorded nowhere), (3) R2 stat-math + marked-for-destruction scope, (4) tracked items (Modifier System, bots, temp control…), (5) a stale §3 tribe note masking a missing `HasTribe` condition. User picked (1) and **reframed past my secret-vs-intervention forks entirely**: secrets and interventions are ONE family — a reactive card (**sigil**) deployable two ways, **invoked** from hand (window, choice) or **inscribed** face-down (auto on first match). Zone = mode selector; the zone-scoped hosting model already supported it with zero new machinery. **Vocabulary (user's):** sigil / invoke / invocation / inscribe / inscription / `PlayerState.inscriptions`. **User-locked:** universal inscribability — choice-ful sigils degrade choices to RANDOM when inscribed (target requirements consume as the existing random-K-at-④ selector mode; replaced my choice-free-guard); one-per-definitionKey, cap 3; **actor-attribution firing rule** (replaced both own-turn options after the user's "opponent invokes on my turn" probe — an inscription never fires on its owner's own actions; attribution via submitter/source-controller; neutral springs both; windows never change `activePlayerId`). Flagged rulings: pay-on-inscribe, one-shot → graveyard, empty-pool guard (stays inscribed), inscribed+held independence, dispatch slot after owner's artifacts, identity hidden/existence public (directed `SigilInscribedEvent` + thin `InscriptionAddedEvent` replace `CardPlayedEvent`; `SigilRevealedEvent` at fire), redacted ③′ declaration for inscribe plays. **This closes the whole point-D deferred bundle** — §3 "Still deferred" is now just the marked-for-destruction scope. Full record + Plan impact: borrow-list note **"Sigils"**; spec §3 new "Sigils" subsection.

**Session 9 part 3 (same day): card crafting → re-scoped to V2 (RECORDED, nothing built).** User clarified: crafting is **not v1** — it's in scope so v1 machinery can support it later. Vision recorded: **aether** (third pool — from neutral kills / artifact conditions / battlecries), a crafting phase (trigger + placement undecided: per-X-turns/condition/spell; intermission-parallel vs in-turn) where the player purchases components (kind → triggers/stats/battlecry-deathrattle, aether-priced by quality) → crafted card to hand at 0 mana. All forks stay open until a v2 spec. **Audit verdict: v1 already supports the shape** (PendingChoice continuation chains, mulligan-style parallel phase, directed choice prompts, per-instance `Card.definition` JSON + `DefaultCardHandler`, additive aether field) — the ONE reserved mechanism is a **match-local definition-registry overlay** (crafting mints fresh `definitionKey`s into a per-match library layer; graveyard/resurrect/replay/trace work unchanged; replay derives the overlay from logged choices). v1's only obligation: **library lookup stays behind an overlay-capable seam** (fold into Epic 01/02 reconciliation as an implementation constraint). Full sketch + audit: `notes/2026-06-11-card-crafting-v2.md`.

**Session 9 part 4 (same day): `HasTribe(Tribe)` condition added (APPLIED).** The deferral scan's item 5: the §3 condition catalog's `MinionTypeIs` row still said tribal matching "depends on Item 8 tribes, not yet in the model" — stale since Item 8 landed; the selector side had `MinionsWithTribe(Tribe)` but the condition side had no tribal predicate, so tribal *triggers* ("whenever a Beast dies") were inexpressible. Added `HasTribe(Tribe)` (matches on `effectiveTribes` intersection — aura-granted tribes match while the aura holds; condition-side dual of `MinionsWithTribe`), fixed the stale note, and corrected the combinator example (it used `MinionTypeIs("beast")` as if tribes were definitionKeys → now `HasTribe(Tribe.Murloc | Tribe.Beast)`, with the `[Flags]` union replacing the `Any`). Plan impact: T8.10 condition-library ticket gains the row. **DEFERRAL SCAN FULLY DRAINED.**

**Session 9 part 5 (same day): R2 inversion stat-math → PINNED (option 3).** User asked what it was; the re-check found `InvertTargetAction` was a bare row — inversion's stat arithmetic was undefined even for healthy minions (a core-v1 handler gap, not an exotic-card detail). User locked: **`currentHealth` ↔ `attack` trade on aura-exclusive values, auras re-applied after** (free — the ⑥ recalc does it). Derived + accepted: enchantment deltas flip + base pair swaps systemically (inverted section never authors stats) ⇒ post-invert always **full flipped health, damage persists as reduced attack** via a system `StatModifier {attackDelta: −min(D, auraless maxHealth)}`; re-invert returns the wound; Silence forgives it; R2 save works iff auraless attack > 0; UnInvert = same symmetric op, mismatch no-ops. Full record: borrow-list note "R2 inversion stat-math — PINNED".

**Session 9 part 6 (2026-06-12, past midnight): marked-for-destruction scope → RESOLVED, option A (the last open design question).** Re-grounding exposed a latent trap (`MinionMortallyWoundedEvent` fired on destroy-marks → a mark would bait a futile save window). Prior-art survey (user-requested explanation first): post-resolution rescue from fiat destruction exists nowhere — MTG deprecated regeneration after decades of "can't be regenerated" text; LoR tiers damage<kill<obliterate. **Locked (user's formulation confirmed):** MortallyWounded = `currentHealth ≤ 0` only (damage or aura/enchant collapse — savable); a destroy-mark is FINAL — no event, no window, no save; ⑦ reaps regardless of health; sole counterplay = ③′ interception of the `DestroyMinionAction` before ④ applies the mark. Riders: event narrowed (retro-strengthens the session-7 cause-drop); "pending death" = shared lingering state, "mortally wounded" = the savable health-domain kind; invert trade executes on a marked minion but the mark survives; selectors stay health-only deliberately; mark+≤0 coexist, mark wins; "cleanse" = additive, rejected v1. `DestroyMinionAction` row was bare → full ④ handler specced. Full record: borrow-list note "Marked-for-destruction scope — RESOLVED".

**Session 9 part 7 (2026-06-12): read-only SPEC REVIEW → `notes/2026-06-12-spec-review-findings.md`** (user-requested, max-effort full read at `a4a1b10`; NO spec edits). **33 numbered findings** [CONTRADICTION/BUG/HOLE/DRIFT/LIMITATION × H/M/L] + a "Differences from Hearthstone" chapter. Headliners: #1 ③ "target is alive" contradicts the `Dead…` save-targeting family (H); #2 directed-events list omits `CardDrawnEvent`/`CardAddedToHandEvent` → broadcast hand reveal, contradicting ⑥′'s own no-expose argument (H); #11 match setup unspecified (starting health/hand sizes/mulligan procedure/2nd-player comp/turn-1 mana/`WaitingForPlayers`/opening-hand sigil registration) (H); #13 healthDelta↔currentHealth adjustment rules undefined (H); #16 "the responder" never defined — hand windows lack the actor-attribution rule inscriptions got; self-response + neutral-attributed ambiguity (H); #3 Poisonous "enemy" scope wrong under lane model; #17 sourceless system actions spring own inscriptions; #27 `Start*` actions are themselves ③′-declarable. Findings are an INPUT QUEUE for a fix pass (spec edits) before/during reconciliation — none applied yet.

**THE SPEC HAS ZERO OPEN DESIGN QUESTIONS** (the review found *execution* defects + holes, not unsettled design forks — except where flagged). Everything is decided, recorded, or explicitly v2 (crafting). **The only remaining pre-implementation work is the end-of-pass plan reconciliation** (all `Plan impact:` lists → epic/ticket files; weapon-scope deletion + artifact re-scope + sigil additions + the library-seam constraint + `HasTribe` + `InvertTargetAction` + `DestroyMinionAction` semantics; stale-name sweeps) → then **Epic 01 / T1.1**.

### ⏹ SESSION STOP (2026-06-17, end of session 15)

**State:** Session 15 = a **SECOND, fresh read-only SPEC REVIEW** (user-requested: "stop inconsistencies, logical bugs, wording, robustness before implementation") of the *post-fix-pass* spec, **followed by walking + applying the findings one-by-one** (distinct from the session-9 review, whose 33 findings are all already applied). New findings doc: **`notes/2026-06-17-spec-review-2.md`** — **17 findings (F1–F17) + 4 design notes (D1–D4)**. Read that doc's "▶ Walk progress" block FIRST on resume.

**Applied this session (spec edited + grep-verified; F-tags trace to the findings doc):**
- **F1 [H, BUG]** — going-second player started turn 1 with **0 mana** (turn-end refresh only refreshes the *ending* player; setup seeded only P1). Fix: **setup seeds BOTH players** `mana=maxMana=StartingMana(1)`. (§1 StartingMana + PlayerState, §4 Match Setup step 8.)
- **F2 [H, CONTRADICTION]** — stale single-stage `SubmitInterventionAction` row deleted; `RespondInterventionAction`+`SubmitInterventionAction` moved into a new **"Responder-initiated (off-turn)"** §2A sub-table; `StartInterventionAction` stays system-issued.
- **F3 [H, CONTRADICTION]** — "iteration bounded by mana, not a hard cap" reworded (3 sites) to **bounded by BOTH the per-turn press budget `MaxInterventionsPerTurn` AND mana** (each chained play = a fresh press ⇒ ≤3 intervention cards per opponent turn).
- **F4 [M, HOLE] → option B** — `Filter` predicate mismatch fixed by factoring a shared **`IEntityPredicate`** (new §3 subsection): entity-property checks (`Friendly`/`Enemy`/`IsSelf`/`IsDamaged`/`IsMortallyWounded`/`HasTribe`/`HasKeyword`/`MinionTypeIs`/`CardTypeIs`/`CostAtLeast/AtMost`) + `All`/`Any`/`Not`. `ITargetSelector.Filter(selector, IEntityPredicate)` (negation now works in a Filter); `ITriggerCondition` reframed as event layer = `SelfIs*` + `Subject(p)`. Friendly/Enemy formalism moved into `IEntityPredicate`; `MinionsWithTribe/Keyword` → sugar; `Not` promoted to first-class; HasTribe-cond/MinionsWithTribe-sel duplication dissolved. **Plan impact:** Epic-08 selector ticket + T8.10 condition ticket both now build on `IEntityPredicate`.
- **F5 [M, BUG] → option a** — Reborn granted *after* summon never fired. Fix: `rebornAvailable` = "charge unspent", **init `true` for every minion**, gated on `keywords has reborn && rebornAvailable`; reborn copy summoned false; Transform resets true; **Silence leaves it untouched** (keyword strip already gates it). (§1, §2A Transform+Silence, §4 ⑦ Phase 3.)
- **F6+F7 [M, HOLE]** — Match Setup reordered to **deal → mulligan → Coin → sigil registration**: Coin after mulligan (unmulliganable); sigil hand-triggers registered once on the *final* post-mulligan hand (no swap bookkeeping — nothing reactive fires during Mulligan phase).
- **F8 [M, DRIFT]** — added `NoInterventionBudget`, `NoMatchingCard` to the §2C `ActionRejectionCode` enum (server-authoritative Intervene guards).

**⏳ STOPPED AT F9 + F10 — PRESENTED, user deferred the decision to next session.** Both proposals are captured verbatim in the findings doc's "▶ Walk progress" block + their own headings (`⏳ AWAITING DECISION`):
- **F9** — define `ChoiceType {Discover,Target,Modal}` + `ChoiceOption(OptionId, EntityId?, DefinitionKey?, Label?)` and **drop `SubmitChoiceAction.choiceId`** (single pending choice ⇒ unambiguous). Open: lock-now vs defer the exact `ChoiceOption` fields to the Epic-01 data-model ticket.
- **F10** — fizzled (countered) minion card currently → `GraveyardMinion` (wrongly resurrectable). Fork: **(a)** remove-from-game (recommended), **(b)** graveyard + non-resurrectable flag, **(c)** accept recyclable.

**Still to walk after F9/F10:** **F11–F17** (low/wording — batch-apply: Battlecry-in-ResolveCardAction wording, directed-events sibling off-by-one, `turn.number` increment step, `EntityId` vs `string`, `autoSkipAll`/`SkipAll` naming, maxHealth≤0 "savable" caveat, `SpellDamageBonus` casing) and **D1–D4** (confirm-on-record design notes: own-turn self-save impossible by design, SpellCastEvent-on-countered-spell, Reborn-model note, press-cap-as-balance-lever).

**▶ RESUME PLAN:**
1. **Read `notes/2026-06-17-spec-review-2.md` "▶ Walk progress" block.** Decide **F9** (lock shape / defer fields) and **F10** (a/b/c), apply, then batch **F11–F17** and confirm **D1–D4**. Mark each `✅` in the findings doc.
2. **Then the (still-pending) end-of-pass plan reconciliation** — now it must ALSO fold in this session's spec-review-2 changes (esp. F4's new `IEntityPredicate`, F1 mana seeding, F2/F3 intervention model, F5 Reborn) on top of the prior backlog → then **Epic 01 / T1.1**.

**Git:** session-15 spec edits + findings doc + this stop block committed and pushed to `origin/main` at session end. Working from `main` (project convention — design-doc edits commit directly to main).

**Note on style (carried):** walked findings one-by-one with present-tradeoff-+-recommend; user picked B on F4 explicitly to **maximize design space** (asked for examples of both A and B first — see [[feedback-design-fork-style]], [[feedback-deliverable-visibility]]).

### ⏹ SESSION STOP (2026-06-17, end of session 14)

**State:** Session 14 **completed the SPEC-REVIEW FIX PASS** — walked the final block (#19–#21, #23–#29, #31–#33) and applied all of it to the spec. **The entire fix pass is now DRAINED** (#1–#10b/#18/#34 batch 1; #10c moot; #11–#15/#22/Items 1&2/Interturn session 12; #16/#17 session 13; #19–#33 this session; #30 moot; #35 reviewed→kept). **Spec grep-verified clean. Spec work committed (`f05a6e0`) + this stop-note; both PUSHED to `origin/main` at session end (`3dd95e5`/`70d9f05` were already on origin — the session-13 "verify on next fetch" note confirmed: only `f05a6e0` was ever local).**

**Post-commit Q&A (explanatory only, NO spec change):** walked how **Discover** works in our model — it is **not its own phase**; it is the shared `PendingChoice` machinery with `choiceType = Discover` (same path as mid-effect targeting + mulligan). Triggered by an **effect** issuing `StartChoiceAction` (options = an `ITargetSelector` candidate set, §3 "Player-choice" consumption mode); §4 "PendingChoice Interruption" halts the pipeline, serialises the continuation into `context`, emits directed `ChoicePromptEvent` + thin public `ChoiceStartedEvent`; `SubmitChoiceAction` resumes. Engine-internal (no `ActionDeclaredEvent` per #27 → can't be countered); inscribed sigils degrade it to random-K (no `PendingChoice` opens); timeout = server-random / mulligan keep-all (#24). **Flagged a small gap for reconciliation:** the spec references `choiceType: ChoiceType` + `options: ChoiceOption[]` but **never enumerates `ChoiceType`'s values nor defines `ChoiceOption`'s shape** → nail down in the **Epic 01 data-model ticket** alongside the other supporting types.

**What landed (spec §1–§4):** 4 user-decided game-feel forks + 2 shape forks + 7 mechanical fixes —
- **#19** neutral-lane full → **fizzle at ④** (effect/system spawn; refill self-limits). **#21** full-hand bounce → **destroy → Deathrattle fires** (HS-faithful; normal bounce is *not* a death → no deathrattle; one invariant, surfaced by the user's edge probe). **#23** Combo → **on-turn only, invocations don't count, per-player reset.** **#24** PendingChoice timeout → **server-rolled random** (logged input, replay-exact) + **mulligan keep-all.**
- **#20** Reborn → `SummonMinionAction.currentHealthOverride: int?` (clamp `[1,maxHealth]`; Reborn passes 1). **#27** `Start*`/`Respond*`/`Submit*` window-machinery actions **don't publish `ActionDeclaredEvent`** (engine-internal window/choice-opening; closes the "cancel a window's opening" recursion).
- **#25** drop-list "leaves the hand by any route." **#26** empty-pool guard fire-time honesty. **#28** `GameConstants.MaxBoardMinions = 7` + `BoardFull` ref. **#29** deleted vestigial `MinionStatsChangedEvent`. **#31(+#33)** new Unaddressed-Features "Inscription counterplay" entry (`InscriptionSelector` + `DestroyInscriptionAction`, additive; inscribe-locking note). **#32** Sigil/Stealth mode-asymmetry sentence.

**Records:** findings-doc #19–#33 each annotated `▶ RESOLVED (session 14)`; borrow-list "Spec-review fix pass" header → **COMPLETE**, new "### #19–#33 ✅ APPLIED (session 14)" subsection with the consolidated session-14 Plan-impact list; data-model mirror `cardsPlayedThisTurn` re-synced.

**▶ RESUME PLAN:**
1. **END-OF-PASS PLAN RECONCILIATION** — the last task before implementation. Walk every `Plan impact:` line accumulated across the borrow-list note (the 13-item pass, the Fireplace points, the hole-hunting pass, the spec-review fix pass incl. the session-14 additions) into the Epic/ticket files under `plans/2026-05-27-ccg-game-logic/`; create the flagged new tickets/epics (known: `StabilizationAbortReport` telemetry T4.8, `ITriggerCondition` lib T8.10, Bot Support epic, `IRandom`+error-codes Epic 01, tribe/keyword 4-field, `ITargetSelector` lib Epic 08, command-log replay Epic 16, `IKeyword` collapse + Epic 07 re-scope, hero-power→Artifact re-scope, weapon-concept deletion, sigils, library-overlay seam, `ResolveCardAction` split, the session-14 list above, **+ the `ChoiceType` enum values / `ChoiceOption` shape gap flagged in the Discover Q&A**). The **#17 actor-classification is DROPPED, not built.** Stale-name sweep of plan/README (e.g. `canAttack`, `attacksAllowedThisTurn`, `originalCard`, weapon refs, hero-power refs).
2. **Then implementation: Epic 01 / T1.1.**
3. **PUSH** — done at session-14 end (`f05a6e0` + this note on `origin/main`); nothing pending.

### ⏹ SESSION STOP (2026-06-16, end of session 13)

**State:** Session 13 RESOLVED + APPLIED + committed findings **#16 + #17** — the session-12 parked foundational question ("should the active player ever intervene on his own turn?") was decided **Model B′ (turn-based)**, and the whole intervention delivery model was reworked. **1 commit (`3dd95e5`) ahead of `origin/main`; the session-12 commits already appear on `origin/main` per local refs (the earlier "not pushed" note was stale — verify on next fetch). Tree clean.**

**The multi-turn design arc (how we got to B′):**
1. **A vs B foundational question → Model B′.** Prior-art grounding: MTG (APNAP), YGO (Quick Effects), LoR (reactive stack) all let the active player respond on his own turn; HS — our genre — does **not**, and our sigils were the only thing that had drifted from HS. Chose turn-based: responder = always the non-active player; inscriptions **and** invocations off-turn only; no intervention while `activePlayerId == null`. This **dissolves #17** (no source-attribution — own-turn cards are simply off) and makes depth-1 automatic.
2. **Probed dropping ⑥′** (keep only ③′, since ③′ holds an action with a clean actor). Surfaced the **reap-as-action** idea (make death-removal a declared action so R1 dying-window saves live at *its* ③′, restoring point-D's "everything interceptable is an action" invariant). Explored thoroughly, **not taken** — B′ + keeping ⑥′ was simpler; reap-as-action recorded as a viable future option.
3. **User authored a delivery spec** (turn-gated responder; unconditional 1s decision + 3s extension windows; skip/skip-all; 3-press cap). Reviewed → reworked into the final model. The Effigy-class "lost content" boundary I flagged was **wrong** (HS Secrets are off-turn anyway — Effigy included) and dropped.

**Applied to spec (`3dd95e5`, §1–§4, grep-verified clean):** GameConstants (`InterventionDecisionWindow` 1s / `InterventionExtensionWindow` 3s / `MaxInterventionsPerTurn` 3); `GamePhase` split `PendingIntervention`→`InterventionDecision|InterventionExtension`; `TurnState.interventionsUsed`; `PlayerState.autoSkipAll`; new `RespondInterventionAction` + re-scoped `Start`/`SubmitInterventionAction`; `InterventionWindowOpenedEvent { phase, stage }`; new §3 "The responder & window delivery" block + Sigils off-turn rule + contingent-window reversal; Phase Guard two rows; ③′/⑥′ unconditional+turn-gated+budget guards; Turn Lifecycle `interventionsUsed` reset; trace `window` record `stage` + worked examples (2)/(3); base-window-budget pin (Unaddressed Features). **Design log:** borrow-list "Spec-review fix pass" #16/#17 + findings-doc #16/#17 (both → RESOLVED).

**▶ RESUME PLAN:**
1. **Continue the findings walk: #19–#21, #23–#33** (skip #18 done, #30 moot, #35 kept). User-needed game-design forks: #19 neutral-zone capacity, #20 Reborn-HP summon param, #21 full-hand bounce/give, #23 Combo accounting, #24 choice-timeout, #27 `Start*` declarability. Mechanical: #25/#26/#28/#29/#31/#32/#33. (Per-finding split: session-9 stop block.)
2. **Then end-of-pass plan reconciliation** (all `Plan impact:` lists → epic/ticket files; **the #17 actor-classification is now DROPPED, not built**) → **Epic 01 / T1.1**.
3. **PUSH** `3dd95e5` (and re-confirm the session-12 commits are on `origin`) when ready.

### ⏹ SESSION STOP (2026-06-15, end of session 12)

**State:** Session 12 continued the SPEC-REVIEW FIX PASS past batch 1, walking findings #11–#17 (+ logging new #35). **Applied + committed: #11, #12, #13, #14, #15, #22 + two NEW user mechanics (Items 1 & 2) + the Interturn neutral-handover lifecycle step.** #35 logged + reviewed→kept. **#16 + #17 REVIEWED → PARKED** pending a foundational question (below). **4 commits are LOCAL on `main` — NOT pushed to `origin`; tree clean after a revert.**

**Commit trail (session 12, on `main`, NOT yet pushed):**
- `45d7141` — #11 match setup (GameConstants block; Match Setup subsection; `WaitingForPlayers` phase-guard; HS-faithful mulligan; **The Coin** = `ModifyManaAction(+1)`) + **Item 1** (deck-build guaranteed opening card — no-RNG post-shuffle swap) + **Item 2 / the Interturn step** (lifecycle step 5 = the neutral lane's "turn", `activePlayerId = none` ⇒ only the neutral trigger group fires; `NeutralZoneConfig.repopulateOnTurnStart`→`firstSpawnTurn`; `SpawnNeutralMinionAction` pool-draw mode; `TurnState.activePlayerId` nullable; steps 5–11 renumbered 6–12; Interturn = a reserved seam, A–E future inhabitants in the note) + #12 cap half + #22 (`ModifyManaAction` clamp) + the `AttackDeclaredEvent` "renderer convenience" fix + #35 log.
- `d01fa91` — #35 reviewed → current behavior KEPT (the combat pipeline already has **two** interception points — `AttackAction` ③′ = prevent, `AttackAction` ④ commits Stealth/budget, strike `DealDamageAction` ③′ = counter; a swing-counter already breaks Stealth; only a *declaration-cancel* preserves it — parked, revisit if a card needs it).
- `feb43aa` — #12 (entity-id half: full id scheme — type-prefixed **prefix-disjoint** allocators, `m*` minions **incl. neutral** on the same allocator, `c*`/`a*`, heroes = the seats `p1`/`p2` = the player itself, no separate hero id, `playerId` = match-local seat never an account id; trace `h1`/`p2.hero` unified to `p1`/`p2`) + #13 (currentHealth follows maxHealth — HS-exact clamp `newCurrent = min(currentHealth + max(0, Δmax), newMax)`).
- `208b573` — #14 (`SilenceMinionAction` handler) + #15 (`TransformMinionAction` handler), both HS-faithful full specs.
- `<this commit>` — session-12 stop block (notes/state only; #16/#17 parking records). **#16's Model-A spec edit was applied then `git checkout`-REVERTED — the spec is unchanged from `208b573`.**

**▶▶ THE PARKED FOUNDATIONAL QUESTION (decide FIRST next session, then apply #16 + #17):** *Should the active player EVER open a hand-intervention window on his own turn?*
- The session walked #16 (define "the responder" for hand windows) and provisionally landed on **Model A (attribution-based):** responder = any player the action is *not* attributed to; the active player never responds to his own actions but **does** respond to non-own (neutral/opponent-sourced) actions resolving mid-cascade on his turn; a **neutral (unattributed)** action opens windows for **both** players, **active player first** (the user's ordering, = the canonical active→opponent→neutral order). A worked scenario confirmed the both-respond case exists under current rules: a neutral **"Toll-Keeper"** (board-wide trigger "when a card is played, deal 1 to each hero") + both players holding a **"Ward"** sigil ("when your hero takes damage, gain 2 armor") → one neutral (unattributed) action, both players entitled to a window. The §3 responder paragraph (+ inscription-rule cross-ref) was **applied then reverted — not committed.**
- Then the user broadened: *everything cascading during the active player's turn is, by chain-of-causation, caused by his own actions — so perhaps he should simply deal with the consequences and never intervene on his own turn* → **Model B (turn-based):** the active player **never** intervenes on his turn; only the off-turn player ever gets windows; the neutral both-respond case collapses to a single (off-turn) responder; **#17 largely dissolves** (the active player can't react to his own draw/fatigue because he can't react at all on his turn).
- **Trade-off:** Model B is simpler/leaner and dissolves #17; Model A preserves active-player mid-cascade self-saves. Full record: borrow-list "#16 + #17 REVIEWED → PARKED" + findings-doc #16/#17.

**▶ RESUME PLAN:**
1. **Decide the parked foundational question (A vs B), apply #16 (responder definition — §3 Reactive Triggers pillar 1, mirroring the inscription attribution rule at the Sigils subsection) + #17 (sourceless-attribution fallback — only needed under A) per the choice, commit.**
2. **Then continue the walk: #19–#21, #23–#33** (skip moot/done/kept: #18 done, #30 moot, #35 kept). User-needed game-design forks: #19 neutral-zone capacity enforcement, #20 Reborn-HP summon param, #21 full-hand bounce/give policy, #23 Combo accounting, #24 choice-timeout policy, #27 `Start*` declarability. Mechanical: #25/#26/#28/#29/#31/#32/#33. (Per-finding mechanical-vs-fork split: session-9 stop block below.)
3. **Then end-of-pass plan reconciliation** (all `Plan impact:` lists → epic/ticket files) → **Epic 01 / T1.1**.
4. **PUSH** the 5 session-12 commits to `origin/main` when ready (currently local-only).

### ⏹ SESSION STOP (2026-06-14, end of session 11)

**State:** Session 11 did **two** things — (A) **parked the entire inversion mechanic as a V2 feature**, and (B) **applied fix-pass batch 1** to the spec. Both DECIDED + APPLIED + committed + pushed; **working tree clean.** It resumed at the open #10c micro-fork; exploring it (a stat-math walk: a `1/6` minion buffed +3 atk → `4/6`, damaged 4 → `4/2`, then inverted → **`2/4`**) surfaced the unresolved **enchantment-flip-on-invert** question, and the user pivoted to the broader call to park inversion (the deepest, most-ramifying item and the only one with no HS reference). Parking dissolved #10c, which unblocked batch 1, applied next.

**What was done (all applied):**
- **Spec** (`2026-05-26-game-mechanics.md`) — every inversion surface removed: §1 `MinionOnBoard.isInverted`, `Card.isInverted`, the `definition` **`normal`/`inverted` split (v1 definitions are now FLAT)**, `GraveyardMinion`/`GraveyardSpell` `isInverted`; §2A `InvertTargetAction` + `UnInvertTargetAction` + the `DestroyMinionAction` inversion clause; §2B `MinionInvertedEvent` + `CardInvertedEvent`; §3 `ITrigger` "On Invert" type; Identity retain-table row + replay line. Dying-window save examples reworded "invert"→"heal". **grep-verified clean** (zero `invert`/`isInverted` in the spec).
- **Plan** — Epic 14 marked ⛔ deferred-to-v2 (file retained as the v2 seed); refs scrubbed from Epic 03, Epic 16 (incl. the `isInverted`-preserved bounce test), README (row + ticket list strikethrough).
- **v2 seed created:** `notes/2026-06-14-inversion-v2.md` — captures option-3 stat-math (full), the **OPEN** enchantment-flip question (flip-deltas vs persist-label = first v2 question), and the keyword **offense↔defense** pairing work (4 native pairs: Divine Shield↔Poisonous, Enrage↔Lifesteal — both *systemically* consistent with the stat-swap — Reborn↔Windfury, Taunt↔Charge; Stealth/Spell-Damage need invented anti-keywords; involution constraint noted).
- **Records:** borrow-list note — R2 stat-math section + #10c marked SUPERSEDED/MOOT, new "Inversion parked to V2" entry; session-10 fix-pass status updated; findings doc — #30 (`CardInvertedEvent`) marked ⛔ MOOT, HS-differences inversion bullet annotated; session-state data-model mirror re-synced (isInverted removed).

**Key consequences:**
- **#10c is dissolved** (its `sourceId:"inversion"` memento no longer exists) → **fix-pass batch 1 is UNBLOCKED.**
- **Death-cadence UNAFFECTED** — R1 (heal a mortally-wounded minion above 0 in the dying window) still justifies the cascade-settle window without R2.

**Update (later in session 11, 2026-06-14): FIX-PASS BATCH 1 ✅ APPLIED + committed.** All of #1–#10b + #18 + #34 landed in the spec (grep-verified clean); full edit list in the borrow-list "Spec-review fix pass (session 10–11)" section. Headliner #34: `PlayCardAction` ④ rewritten to author the normal-play **commit** branch + new **`ResolveCardAction`** row (counterspell window at its ③′) + `CardPlayFizzledEvent` + window-cause on `InterventionWindowOpenedEvent` + 3-block trace rewrite with the new `fizzle` record kind.

**Commit trail (session 11, all pushed to `origin/main`, tree clean):**
- `57f2fa8` — park inversion entirely as a v2 mechanic (spec + plan scrub; `notes/2026-06-14-inversion-v2.md` seed)
- `375d40d` — spec-review fix-pass batch 1 (#1–#10b, #18, #34)
- `<this commit>` — session-11 stop block finalize

**▶ RESUME POINT (batch 1 done):**
1. **Continue the walk at #11** (match setup) → #33. **Now-moot/changed by the inversion parking:** **#30 fully moot** (skip); **#14** drops its inversion-memento sub-question (rest stands); **#15** drops its `isInverted`-across-transform sub-question (rest stands). The session-9 stop block below carries the per-finding mechanical-vs-fork split + recommendations for #11–#33. **Several #11–#33 items are pure game-design number/policy choices needing the user** (e.g. #11 starting health/hand sizes/mulligan, #20 Reborn HP, #21 full-hand bounce/give policy, #24 choice-timeout) — present trade-off + recommend per [[feedback-design-fork-style]], don't apply unilaterally. Apply mechanical fixes in batches + commit per rhythm (same as batch 1).
2. Then end-of-pass **plan reconciliation** (backlog + the inversion-parking + batch-1 edits already applied) → **Epic 01 / T1.1**.

### ⏹ SESSION STOP (2026-06-13, end of session 10 — spanned 06-12 → 06-13)

**State:** Session 10 = the **SPEC-REVIEW FIX PASS, part 1** — walking all 33 findings one by one (user's call: every item, fork or not, Q&A batches). **Decisions are LOCKED for #1–#10b, #18, and a NEW #34, but ⚠ NO SPEC EDITS HAVE BEEN APPLIED YET** — the full decision record (with reasoning + Plan-impact accumulator) is the borrow-list note's new section **"Spec-review fix pass (session 10)"**. Read that section before resuming; this block is the pointer + resume plan only.

**Decided this session (headlines):**
- Mechanical accepts: **#1** (③ aliveness → selector-carried), **#2** (draw-event directed/thin split), **#3** (Poisonous any-side), **#4** (Lifesteal neutral no-op), **#6** (`isDamaged` DELETED), **#7** (⑤ + inscriptions), **#9** (`SourceEntityId`), **#10a** (artifact-activation wording), **#10b** (host list incl. **inscriptions** — user correction).
- **#5**: `AttackDeclaredEvent` → **bus-only** (revised from carve-out after grounding; renderer covered by `AttackPerformedEvent` + window events).
- **#34 (NEW, user-found via the fireball/counterspell probe — the session's headliner):** cancel was a free no-op (no mana, card kept, zero deltas, Counterspell worthless, replay-invisible). **Resolution = two-action split:** `PlayCardAction` ④ commits (pay/leave-hand/count/`CardPlayedEvent`) + enqueues **`ResolveCardAction`**; interception targets the resolution; fizzle → `CardPlayFizzledEvent`, never refunds; mutations stay ④-only; dissolves the cost-gate-skip + commitment-trace patches; pre-cast vs post-cast cancel both expressible. Spell graveyard at play-④; minion-card entry on fizzle only.
- **Window-cause rider:** `InterventionWindowOpenedEvent` gains the cause (③′ held-action verbatim / ⑥′ `causeEventIds[]`; inscribe-redaction carve-out).
- **#18 (pulled forward):** enqueue-and-drain — wave-enqueued actions get the full pipeline incl. ③′/⑥′; fix ⑦ Phase-2 wording.
- **#8:** trace example replaced + extended with BOTH verified four-stream scenarios (fireball-countered, deathrattle-intercepted); **trace schema** gains the `fizzle` record kind.

**Methodology notes (carry forward):** (1) The user's "describe the fireball scenario across bus/log/wires and verify it adds up" probe found #34 — walking concrete renderability scenarios is a power tool; do it proactively for stream-touching decisions. (2) **Deliverable-visibility lesson saved to memory** (`feedback-deliverable-visibility`): twice I produced the requested walkthrough but buried it ahead of an AskUserQuestion dialog and the user never saw it — requested artifacts go in a plain reply as the centerpiece, question cadence resumes next turn. (3) User again reframed past my fork (pipeline-purity question → option B), and corrected my #10b host list.

**▶ RESUME POINT:**
1. **#10c first** — presented verbose-with-example, awaiting the user's pick: sentinel-documented (recommended) / `kind` field / typed union.
2. **Then APPLY batch 1 to the spec** (everything in the borrow-list section: #1–#10c, #18, #34+`ResolveCardAction`, window-cause, trace examples+`fizzle` kind) — re-grep for stale text (`isDamaged`, `SourceMinionId`, "damaged enemy minion", "hero-power damage", old trace vocabulary, ③ "is alive", ⑦ "④–⑥"), **commit per the established rhythm**.
3. **Then continue the walk at #11** (match setup — first of the H holes) through #33, skipping #18 (done); the session-9 stop block below still carries the per-finding mechanical-vs-fork split + recommendations for #11–#33.
4. Then: borrow-list Plan-impact lines are already accumulating in the new section → end-of-pass **plan reconciliation** → **Epic 01 / T1.1**.

### ⏹ SESSION STOP (2026-06-12, end of session 9 — spanned 06-11 → 06-12)

**State:** ALL session-9 work committed + pushed to `origin/main` — working tree clean. Commit trail (chronological):
- `8b46ddf` — hero-weapon concept DROPPED + hero armor landed (hero-combat pass **CLOSED**; heroes never attack/freeze, Weapon CardType gone, `armor` + absorb slot + `armorAbsorbed`)
- `3f5aade` — **SIGILS**: unified reactive-card model (invoke from hand / inscribe face-down; choice→random degradation; one-per-name cap 3; actor-attribution firing rule; pay-on-inscribe; one-shot; directed/public sigil events)
- `57920e8` — card crafting re-scoped to **v2** (aether + purchase-components sketch + v1 audit → `notes/2026-06-11-card-crafting-v2.md`; v1 obligation = overlay-capable library-lookup seam)
- `bdf29bd` — `HasTribe(Tribe)` condition (Item-8 condition-side gap; combinator example fixed)
- `9f64ee8` — inversion stat-math PINNED (option 3: `currentHealth`↔`attack` trade on aura-exclusive values; delta-flip; damage→attack-penalty memento)
- `a4a1b10` — destroy-marks FINAL + SILENT (option A; `MinionMortallyWoundedEvent` narrowed to health-domain; `DestroyMinionAction` ④ handler specced; "pending death" vs "mortally wounded" terminology)
- `443a407` — **read-only SPEC REVIEW**: `notes/2026-06-12-spec-review-findings.md` — 33 tagged findings + Hearthstone-differences chapter (NO spec edits)
- `<this commit>` — session-9 stop block.

**What session 9 was:** the most decision-dense session yet — closed the hero-combat pass via the weapon-drop pivot, invented the sigil system (the secrets bundle's resolution), drained EVERY deferral in the document (crafting→v2, `HasTribe`, R2 inversion math, destroy-mark scope), then produced the full spec audit. The spec is design-complete; the audit found execution defects, not open forks (with a handful of micro-forks flagged inside findings).

**▶ RESUME POINT — the SPEC-REVIEW FIX PASS.** The user took `notes/2026-06-12-spec-review-findings.md` away to review offline and will return with their own read. Next session:
1. **Ask for their annotations first** — which findings they accept/reject/re-scope; their review overrides the document (expect reframes per [[feedback-design-fork-style]]).
2. Walk the accepted findings **in document order** (contradictions → bugs → holes → drift → limitations). Two work kinds, interleaved:
   - **Mechanical fixes** (apply on go-ahead, batch-commit): #1 ③-aliveness wording, #2 directed draw-events split, #3 Poisonous scope, #4 Lifesteal null-owner no-op, #5 `AttackDeclaredEvent` log-status carve-out, #7 ⑤-ordering text, #8 trace-example vocabulary, #9 `IAura` naming note, #10 stale-text bundle, #25 hand-drop-list wording, #28 constants block, #29 `MinionStatsChangedEvent` deletion.
   - **Micro-forks needing the user** (present trade-off + recommend, per the established Q&A pattern): #6 `isDamaged` delete-vs-specify; #11 setup values (starting health/hand sizes/mulligan rules/2nd-player comp — pure game-design numbers); #12 hero max-health + hero entity-id convention; #13 healthDelta↔currentHealth rules (recommend HS-exact: gain raises current, loss clamps, silence clamps no-min-1); #16 responder definition + hand-side actor-attribution (recommend: mirror the inscription rule — hand sigils never window on their owner's own actions; neutral-attributed → both players, active-player-first order); #17 sourceless-action attribution fallback (recommend: `playerId` param); #18 windows inside ⑦ Phase 2/3 (recommend: active — next-wave deferral makes saves coherent); #20 Reborn 1-HP param; #21 full-hand bounce/give policies (recommend HS: destroy / burn); #23 invocation Combo accounting; #24 choice-timeout policy; #27 `Start*` declarability exemption; #30 `CardInvertedEvent` visibility; #31 inscription-counterplay Unaddressed entry; #32 stealth/random-K sentence.
3. **Record the pass** in the borrow-list note (new section "Spec-review fix pass", with `Plan impact:` lines where fixes touch epics) + re-grep after each batch + commit per batch (established rhythm).
4. **Then:** the end-of-pass **plan reconciliation** (existing backlog in the paragraph above + whatever the fix pass adds) → implementation at **Epic 01 / T1.1**.

### ⏹ SESSION STOP (2026-06-09, end of session 7)

**State:** ALL session-7 work committed + pushed to `origin/main` — working tree clean, `main` in sync with remote. Commit trail (chronological):
- `d3faf2e` — session-7 spec work: hole #2 fatigue + queued topics #1 (trigger ordering) + #2 (lane-based friendly/enemy + summon-event consolidation), plus the session-state data-model mirror re-sync. (One substance commit — the three decisions' edits interleave across the spec/note/state files, so they couldn't be cleanly split per-decision.)
- `<this commit>` — session-7 stop block.

**What session 7 was:** continued from the hole-hunting menu into the queued topics. Closed: **hole #2 fatigue**; **queued topic #1** (trigger order = deathrattle order → one canonical board order); **queued topic #2** (neutral-lane auras → a central **lane-based friendly/enemy** model, with the summon event consolidated so the neutral lane behaves like a regular lane). Also re-synced the session-state §1 data-model mirror (provenance note added). Per-decision records + Plan impact live in the borrow-list note ("Hole #2", "Queued topics" #1 and #2). Spec re-grepped clean each round (no stale `cause` field-writes, no "fires once"/"not re-declared", no controller-based friendly phrasing, no live `NeutralMinionSpawnedEvent`, no "player auras do not apply").

**Open holes: NONE from the hole-hunting menu** (#1, #2, #3+follow-on, #4, #6 all done). **Queued topics remaining: #3 debug text format, #4 intervention visibility** (both more standalone than #1/#2 were). **Recorded deferrals unchanged:** temporary control; `!bornNeutral`-dies-in-neutral routing; hero freeze; hero armor; hero-combat pass; the 0-cost-intervention-loop residual.

**▶ RESUME OPTIONS (user drives one):**
- **(a)** the remaining **queued topics** — **#3 debug text format** (human-readable `GameState`/queue/event-stream dump for debugging + scenario authoring; cf. old-project Spine/Exact traces + the `StabilizationAbortReport` repro format) or **#4 intervention visibility** (what the responder sees at ③′/⑥′; dual of the deferred secret-visibility question) — or **call hole-hunting/probing done**.
- **(b)** a **recorded deferral** / the **hero-combat pass** (bundles hero freeze + hero armor + heroAttack/weapon/durability — several deferrals converge here).
- **(c)** the **end-of-pass plan reconciliation** (apply every `Plan impact:` list to the epic/ticket files under `plans/2026-05-27-ccg-game-logic/`, create flagged new tickets/epics, update the README; sweep the plan/README for stale field names — `canAttack`/`attacksAllowedThisTurn`/`originalCard` — and the session-6/7 shape changes: `candidateCardIds`, turn-end mana step, `fatigueCounter`, `MinionSummonedEvent` as sole summon event, lane-based friendly/enemy) → then implementation at **Epic 01 / T1.1**.

**Methodology note (carry forward):** session 7 again had the user correct two of *my* over-claims — the controller-based friendly/enemy reading (they reframed it lane-based, which generalized cleanly and even matched a literal `null==null` reading of the existing condition), and a topic-#1 overstatement where I'd lumped `FriendlyOnly` in with genuinely-empty turn/hero-context conditions. The user also supplied the crisp `enemy` predicate (`!=` && `!=null`) and the "neutral lane behaves like a regular lane" north-star that justified removing `NeutralMinionSpawnedEvent`. Keep presenting trade-offs + recommending, but expect the user to reframe toward the leaner/more-coherent model — verify against their reframes rather than defending the first framing. See [[feedback-design-fork-style]].

### ⏹ SESSION STOP (2026-06-09, end of session 6)

**State:** ALL session-6 work committed + pushed to `origin/main` — working tree clean, `main` in sync with remote (verified). Commit trail (chronological):
- `a7be11d` — hole #6: MTG-style intervention loop (re-declare/re-offer, set-valued window, depth-1 caps nesting only) + turn-end mana refresh (dropped `reservedMana`)
- `b360eab` — depth-1 wording scoped to player windows + hero-armor gap logged (Unaddressed Features)
- `ed9fc46` — 4 queued topics logged for next session

Spec re-grepped clean (no stale `single-response` / `declared once` / `not re-published` / `one window per matched card` / `reservedMana` / `cannot itself be intervened`). Spec believed internally consistent.

**Open holes unchanged:** only **#2 fatigue** remains from the menu (not modelled — no `PlayerState` counter, no `FatigueDamage` action/event, yet `HeroMortallyWoundedEvent` lists `fatigue` as a cause). **Recorded deferrals:** temporary control; `!bornNeutral`-dies-in-neutral routing; hero freeze; **hero armor** (`PlayerState.armor` + `GainArmorAction` + ④ precedence slot); hero-combat pass; the 0-cost-intervention-loop residual (no cap, per user's resource-bound call).

**▶ RESUME OPTIONS (user drives one):** (a) **#2 fatigue**; (b) a **recorded deferral** / hero-combat pass; (c) keep **probing** (open Q/A) or call hole-hunting done; (d) the **end-of-pass plan reconciliation** (apply all `Plan impact:` lists; sweep old field names `canAttack`/`attacksAllowedThisTurn`/`originalCard` in plan/README; note session-6 also moves the mana-refresh step + adds `candidateCardIds`/`SubmitInterventionAction` shape) → implementation at **Epic 01 / T1.1**.

**▶ QUEUED TOPICS (logged 2026-06-09 for next session — full detail in the borrow-list note's "Queued topics" section):**
1. **Trigger ordering = deathrattle ordering** — confirm/unify that general trigger fire order and death-wave sort order are one rule (active player first, then L→R by board index); pin neutral's slot in trigger order; de-duplicate the two Ordering-table rows.
2. **Neutral-lane auras** — do neutral minions emit auras, to whom; can any aura affect the neutral lane? (Spec only says *player* auras don't reach neutral minions while neutral.)
3. **Debug text format** — readable rendering of `GameState` + action queue + event stream for debugging/scenario authoring (cf. old-project Spine/Exact traces + the `StabilizationAbortReport` repro format).
4. **Intervention visibility** — what the responding player sees about the action/event they're intervening on (held-action params at ③′, matched events at ⑥′); responder's-eye dual of the deferred secret-visibility question.

**Methodology note:** session 6 again corrected one of *my own* over-builds — I proposed a `reservedMana` field; the user replaced it mid-edit with the simpler "refresh mana at turn-end" model (one fewer field, resolves the ceiling question for free). Keep reaching for the leaner state-model when an equivalent reframe exists.

### ⏹ SESSION STOP (2026-06-08, end of session 5)

**State:** ALL session-5 work committed + pushed to `origin/main` — working tree clean, `main` in sync with remote (verified). Commit trail (chronological):
- `3adc01f` — Hole #3 neutral zone (control/command + neutral graveyard + `GraveyardEntry` refactor)
- `fefedb3` — summoning-sickness model (`canAttack`→`summoningSick`, pulled Charge/Rush eligibility)
- `efd3ce0` — refinement 2.a (`TakeControlAction` trigger re-bucket wording)
- `220b562` — attack-budget derived from keywords (dropped `attacksAllowedThisTurn`)
- `6832645` — Hole #1 (`AttackResolvedEvent`→`AttackPerformedEvent` + `AttackAction` ④ handler + Stealth-break)
- `f6e415b` — Hole #4 freeze half (`frozenOnTurn` + end-of-turn thaw + `MinionThawedEvent`)

Spec (`specs/2026-05-26-game-mechanics.md`) re-grepped clean each round: no stale `canAttack` / `attacksAllowedThisTurn` / `originalCard` / `AttackResolvedEvent` / "Taunt ignored" / post-swap-unfreeze language (the few remaining mentions are deliberate supersede/rationale notes). Spec believed internally consistent.

**What session 5 was:** an open Q/A "hole-hunting" pass (the user probes, I characterize + present trade-offs + recommend, then apply on their go-ahead). Full per-hole records + Plan-impact lists live in the borrow-list note's **"Hole-hunting pass"** section (`notes/2026-05-28-old-project-borrow-list.md`). Holes closed: **#3** (+ eligibility follow-on, 2.a, attack-budget), **#1**, **#4** (both halves).

**▶ RESUME OPTIONS (user drives one):**
- **(a)** Hole **#2 — fatigue**: not in the data model at all (no counter on `PlayerState`, no `FatigueDamage` action/event), yet `HeroMortallyWoundedEvent` already lists `fatigue` as a cause. The last open hole from the session-5 menu; clean standalone. *(Likely next.)*
- **(b)** A **recorded deferral** — temporary/Shadow-Madness control; the `!bornNeutral`-dies-in-neutral graveyard branch; or **hero freeze** (needs `PlayerState.heroIsFrozen/heroFrozenOnTurn` + `HeroFrozenEvent`/`HeroThawedEvent` + ③ wiring — naturally bundled with a **hero-combat pass**, which is itself only lightly specced: hero attacking via weapon, `heroAttack`, durability).
- **(c)** Keep **probing for new holes** (open Q/A) — or call the hole-hunting pass done.
- **(d)** The official pre-implementation task — **end-of-pass plan reconciliation** (apply all `Plan impact:` lists to the epic/ticket files under `plans/2026-05-27-ccg-game-logic/`, create flagged new tickets/epics, update README). NOTE: the plan/README files still reference the OLD field names (`canAttack`, `attacksAllowedThisTurn`, `originalCard`) — reconciliation must sweep those too. Then implementation at **Epic 01 / T1.1**.

**Methodology note for next session:** the user values coherence/correctness and HS-faithfulness; present trade-offs + recommend (don't just pick), give prior-art (Hearthstone/Fireplace) comparisons, reject fuzzy dispositions. Each applied decision → spec edit + borrow-list note record (with Plan impact) + session-state + memory, then commit+push (per-decision granularity, matches this repo's established workflow). Two of session 5's fixes corrected *my own* earlier hand-waves (the inverted freeze rule; the over-specified trigger list-move) — worth re-reading freshly rather than trusting prior prose.

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
**Minion, Spell, Artifact**. Hero/class is tied to the deck (like Hearthstone). *(Revised 2026-06-11 twice: the original fourth type, Hero Power, was replaced by the **Artifact** system — stakeholder/playtest feedback; then **Weapon was REMOVED** with the whole hero-weapon concept — heroes never attack, the hero kit is the artifact row. See sessions 8/9 + spec §1.)*

### Keywords (all selected, system must be extensible)
Taunt, Divine Shield, Charge, Rush, Lifesteal, Windfury, Poisonous, Stealth, Spell Damage +X, Reborn, Enrage, Freeze — plus any future keywords added via new `IKeyword` implementations.

### Trigger types (all selected, system must be extensible)
Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous), On Damage Taken, On Friendly Minion Death, On Spell Cast, On Artifact Activated (renamed from Inspire, 2026-06-11), Combo, On Invert — plus any future triggers added via new `ITrigger` implementations.

### Inversion mechanic ⛔ DEFERRED TO V2 (parked 2026-06-14 — see `notes/2026-06-14-inversion-v2.md`; builds nothing in v1)
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
- **Card crafting** — multi-step interactive sequence producing a custom card *(re-scoped 2026-06-11: **NOT v1** — a v2 feature; aether pool + purchase-components vision and the v1 compatibility audit live in `notes/2026-06-11-card-crafting-v2.md`; v1's only obligation = keep library lookup behind an overlay-capable seam)*
All require a `PendingChoice` game phase.

### Intervention system
When an opponent declares an action, the other player gets a **single-response window** (play one card / skip) before the action resolves. Exact trigger conditions TBD in Game Server spec. Requires `PendingIntervention` game phase. *(Long since superseded in detail: set-valued looping windows (session 6), visibility (session 8), and the **sigil** unification (session 9, 2026-06-11) — reactive cards are sigils, **invoked** from hand or **inscribed** face-down as auto-firing inscriptions; see spec §3 "Sigils".)*

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

## Approved data model (Section 1)

> **Provenance (synced 2026-06-09):** this block is a convenience mirror of spec §1 (`specs/2026-05-26-game-mechanics.md`). The **spec is authoritative** — on any conflict, the spec wins. Re-synced through end of session 6 (`a9cdb97`): field renames `cardId`→`definitionKey`, `canAttack`→`summoningSick`, dropped `attacksAllowedThisTurn`; added neutral-zone/origin/3-field tribe+keyword/freeze/reborn fields; `GraveyardEntry.originalCard` removed; `PendingIntervention` set-valued; turn-end mana refresh; **session 7** added `PlayerState.fatigueCounter` + dropped `cause` from the two mortally-wounded events; **session 9 (2026-06-11)** dropped the hero-weapon concept (`heroAttack`, `weapon`, `WeaponOnHero`, `GraveyardWeapon`, Weapon card type all REMOVED — heroes never attack), added `PlayerState.armor`, and added `PlayerState.inscriptions` (the sigil model). Comments trimmed vs. spec; read the spec for full rationale.

### GameState
```
sessionId: string
player1: PlayerState
player2: PlayerState
neutralZone: MinionOnBoard[]          // ownerId = null; Taunt is per-LANE (neutral Taunt forces attacks INTO the neutral lane only); player auras don't apply
neutralZoneConfig: NeutralZoneConfig? // null = no neutral zone this game mode
neutralGraveyard: GraveyardEntry[]    // shared, owned by GameState (not per-player); a dead minion lands here iff bornNeutral && ownerId==null at death (§4 ⑦ routing)
turn: TurnState
timer: TimerState
phase: GamePhase                      // Mulligan | WaitingForPlayers | InProgress | PendingChoice | PendingIntervention | Ended
winnerId?: string
pendingChoice?: PendingChoice
pendingIntervention?: PendingIntervention
mulliganState?: MulliganState
rngSeed: ulong                        // fixed at match creation; drives ALL randomness via IRandom; server-side, never sent to client
currentActionEpoch: int               // monotonic; ++ per action at stage ④; event-visibility filter + per-action RNG derivation
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
health, mana, maxMana: int
armor: int                    // hero damage ABSORBER (init 0, no cap); depleted before health at §4 ④ (hero absorb slot; minion analogue = Divine Shield); gained only via GainArmorAction. Heroes have NO attack/weapon/frozen state — heroes never attack (2026-06-11)
hand: Card[]
deck: Card[]
board: MinionOnBoard[]        // player's own side only
graveyard: GraveyardEntry[]   // unified — minions + spells + artifacts
artifacts: ArtifactOnBoard[]  // artifact row, cap 3 incl. the starter (hero-power REPLACEMENT, 2026-06-11); heroPower/heroPowerUsedThisTurn REMOVED
inscriptions: Card[]          // INSCRIBED sigils (§3 Sigils, 2026-06-11): reactive cards pre-armed face-down; cap 3, one per definitionKey; identity hidden (directed/public sigil events); hand → inscriptions → graveyard on reveal
cardsPlayedThisTurn: int      // Combo tracking — this player's ON-TURN plays (PlayCardAction incl. inscribe); off-turn invocations do NOT count; per-player reset at turn advance (#23, session 14)
fatigueCounter: int           // empty-deck-draw counter; init 0, never resets; ++ then deal the new value to this hero (1,2,3,…) — §2A DrawCardAction
```

### MinionOnBoard
```
minionId: string
definitionKey: string         // SOLE link into card-def library + ICardHandler; NOT a Card.id (playing a card never transfers its id; tokens have definitionKey but no Card.id)
ownerId: string?              // null = neutral zone. TakeControlAction sets a player (re-homes onto their board; then dies to their graveyard)
bornNeutral: bool             // immutable ORIGIN flag: true iff system-spawned neutral (SpawnNeutralMinionAction); survives a control change; with ownerId, sole input to §4 ⑦ graveyard routing
baseAttack, baseHealth: int   // immutable, from card definition
enchantments: StatModifier[]  // permanent buffs; Silence clears all
auraAttackBonus, auraHealthBonus: int  // recalculated each aura pass; never in enchantments
attack: int                   // = baseAttack + Σenchantments.attackDelta + auraAttackBonus
maxHealth: int                // = baseHealth + Σenchantments.healthDelta + auraHealthBonus
currentHealth: int            // takes damage; healed up to maxHealth. ≤0 (or destroy-marked) ⇒ "mortally wounded": stays ON board (targetable/countable/keyword-active) until next death-wave settle (§4 ⑦)
keywords: string[]            // EFFECTIVE view = intrinsic ∪ granted ∪ aura; what the engine queries (resolved to IKeyword at runtime)
intrinsicKeywords: string[]   // from card def at summon; Silence clears
grantedKeywords: string[]     // permanent non-aura grants; retained across RetainEnchantments bounce; Silence clears
auraKeywords: string[]        // aura-granted, recomputed each pass; never persisted; ANY keyword may be aura-granted (pulled, never a bus subscription)
intrinsicTribes: Tribe        // from card at summon; immutable; survives Silence
grantedTribes: Tribe          // permanent tribe grants; Silence clears to Tribe.None
auraTribes: Tribe             // recomputed each aura pass; never persisted
effectiveTribes: Tribe        // = intrinsicTribes | grantedTribes | auraTribes; all engine tribe checks use this
summoningSick: bool           // raw fact (replaced derived `canAttack`): entered a board this turn, not yet woken. Set by BOTH summon and control; cleared by turn-start sweep. Attack eligibility (incl. Charge-any / Rush-minions-only) PULLED from effective keywords at §4 ③
attacksUsedThisTurn: int      // raw counter; reset at turn-start. ONLY stored attack-state — per-turn BUDGET is PULLED at §4 ③ (Windfury→2 else 1), never a field (can't desync from keyword). Not consulted for a commanded neutral
summonOrder: int              // monotonic per session; trigger fire-order disambiguation
isFrozen: bool                // frozen → cannot attack (§4 ③ AttackerFrozen)
frozenOnTurn: int             // turn.number when freeze landed; drives end-of-turn thaw rule (Turn Lifecycle step 3)
isDamaged: bool
rebornAvailable: bool         // one-time Reborn charge (distinct from the persistent "reborn" keyword tag); init at summon to keywords.Contains("reborn"); reborn-summon path is false; consuming THIS (not the keyword) is what reborn does (§4 ⑦)
// TRIGGERS: not fields. ITrigger(s)/Deathrattle/keyword-hooks registered into IEventBus by ICardHandler (via definitionKey) at summon (④), dropped on leave. Bus carries only ITrigger; keywords are pulled from the effective view
```

### Card
```
id, name: string
definitionKey: string         // only cross-entity link into card-def library; distinct from per-instance `id`
type: CardType                // Minion | Spell | Artifact   (Artifact replaced HeroPower; Weapon REMOVED — both 2026-06-11)
rarity: CardRarity
tribes: Tribe                 // [Flags] intrinsic taxonomy from definition; Tribe.None = tribeless (valid); NOT cleared by Silence
baseManaCost: int
baseAttack?, baseHealth?: int // Minion only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
grantedKeywords: string[]     // carried from a RetainEnchantments bounce/shuffle; migrate to the new minion's grantedKeywords on play; not part of base definition
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // the card's definition body (flat — inversion's normal/inverted split is v2)
handlerKey?: string
// REACTIVE TRIGGERS: a card carrying one is a SIGIL (§3 Sigils) — same trigger, zone decides: HAND → may be INVOKED (player-choice window); INSCRIPTIONS zone → auto-resolves (choices degrade to random). Every sigil supports both deployments. (Most cards have none.)
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
  turnPlayed: int
  // NO stored Card. Card form is FABRICATED on demand (§1 Fabrication rule) at point of use — minting a fresh
  // Card.id as a stage-④ action. A pre-stored originalCard would be derived data that can only drift. Each
  // subtype keeps its own ENTITY snapshot, the single source of truth the fabrication reads.

GraveyardMinion : GraveyardEntry
  snapshot: MinionOnBoard   // full death state; carries definitionKey → card fully derivable
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  definitionKey: string     // a spell has no board snapshot → stores its own identity → card fabricated from this on recast

GraveyardArtifact : GraveyardEntry
  artifactState: ArtifactOnBoard // full snapshot; both DISCARD and DESTROY land here (2026-06-11)
  destroyedOnTurn: int
```

### Supporting types
```
ArtifactOnBoard     — artifactId, definitionKey, ownerId: string; baseActivationCost: int? (null = passive-only; effective cost PULLED
                      from the definition's formula — starter = base + usesThisTurn); durability: int? (null = infinite; ONE counter,
                      definition declares consuming moments: activation / trigger-fire / both); charges: int (generic accumulator,
                      via ModifyArtifactChargesAction only); usesThisTurn: int (reset at controller's turn start); equippedOnTurn: int
                      // no attack/health — never combat/death-wave entities, not Silence-able, not aura-receivers (may HOST auras)
TurnState           — activePlayerId: string, number: int
TimerState          — secondsRemaining: int
PendingChoice       — waitingPlayerId: string, choiceType: ChoiceType, options: ChoiceOption[], context: JsonElement
PendingIntervention — respondingPlayerId: string, heldAction: GameAction?, candidates: InterventionCandidate[], timeoutSeconds: int
                      // SET-VALUED window (2026-06-09): candidates = responder's matching + affordable reactive
                      // hand cards. Player plays ONE (card + targets) or skips. heldAction null = post-reaction
                      // (re-offer loop); non-null = declaration-hold (re-declare loop). Re-opens after each PLAY, never after a skip.
InterventionCandidate — cardId: string, targetIds: string[]   // offered card + engine-computed legal targets (2026-06-10);
                      // single source for submit-validation AND the responder-directed InterventionPromptEvent
MulliganState       — player1Completed: bool, player2Completed: bool
EffectContext       — sourceId: string, sourcePlayerId: string, targetId: string?, state: GameState (read-only), SpellDamageBonus: int
                      // BASE resolution context (IEffect.Execute / ITargetSelector.Select / keyword pulls). sourceId = unified entity ref (minion OR card), set by the enqueuer from the action's source fields
CardPlayContext : EffectContext — + card: Card   // SUPERSET for ICardHandler.OnPlay only (adds the resolved Card; card.id == sourceId)
```

---

## Key files

| File | Purpose |
|---|---|
| `docs/superpowers/specs/2026-05-25-ccg-system-design.md` | System design doc (approved) |
| `docs/superpowers/plans/2026-05-26-game-server.md` | Game Server plan — **SUPERSEDED, do not execute** until new mechanics spec is written and a new plan is created |
| `docs/brainstorm/` | Brainstorm HTML artifacts |
| `.superpowers/brainstorm/` | Visual companion session files |
