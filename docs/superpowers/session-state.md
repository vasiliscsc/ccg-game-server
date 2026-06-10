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
**Minion, Spell, Weapon, Artifact**. Hero/class is tied to the deck (like Hearthstone). *(Revised 2026-06-11: the original fourth type, Hero Power, was replaced by the **Artifact** system — stakeholder/playtest feedback; see session 8 part 2 + spec §1 `ArtifactOnBoard`.)*

### Keywords (all selected, system must be extensible)
Taunt, Divine Shield, Charge, Rush, Lifesteal, Windfury, Poisonous, Stealth, Spell Damage +X, Reborn, Enrage, Freeze — plus any future keywords added via new `IKeyword` implementations.

### Trigger types (all selected, system must be extensible)
Battlecry, Deathrattle, Start of Turn, End of Turn, Aura (continuous), On Damage Taken, On Friendly Minion Death, On Spell Cast, On Artifact Activated (renamed from Inspire, 2026-06-11), Combo, On Invert — plus any future triggers added via new `ITrigger` implementations.

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

## Approved data model (Section 1)

> **Provenance (synced 2026-06-09):** this block is a convenience mirror of spec §1 (`specs/2026-05-26-game-mechanics.md`). The **spec is authoritative** — on any conflict, the spec wins. Re-synced through end of session 6 (`a9cdb97`): field renames `cardId`→`definitionKey`, `canAttack`→`summoningSick`, dropped `attacksAllowedThisTurn`; added neutral-zone/origin/3-field tribe+keyword/freeze/reborn fields; `GraveyardEntry.originalCard` removed; `PendingIntervention` set-valued; turn-end mana refresh; **session 7** added `PlayerState.fatigueCounter` + dropped `cause` from the two mortally-wounded events. Comments trimmed vs. spec; read the spec for full rationale.

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
health, mana, maxMana, heroAttack: int
hand: Card[]
deck: Card[]
board: MinionOnBoard[]        // player's own side only
graveyard: GraveyardEntry[]   // unified — minions + spells + weapons
weapon?: WeaponOnHero
artifacts: ArtifactOnBoard[]  // artifact row, cap 3 incl. the starter (hero-power REPLACEMENT, 2026-06-11); heroPower/heroPowerUsedThisTurn REMOVED
cardsPlayedThisTurn: int      // Combo tracking
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
isInverted: bool
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
type: CardType                // Minion | Spell | Weapon | Artifact   (Artifact replaced HeroPower, 2026-06-11)
rarity: CardRarity
tribes: Tribe                 // [Flags] intrinsic taxonomy from definition; Tribe.None = tribeless (valid); NOT cleared by Silence
baseManaCost: int
baseAttack?, baseHealth?: int // Minion / Weapon only
modifiers: StatModifier[]     // in-hand cost/stat changes; attack/healthDelta migrate to enchantments on play
grantedKeywords: string[]     // carried from a RetainEnchantments bounce/shuffle; migrate to the new minion's grantedKeywords on play; not part of base definition
effectiveCost: int            // max(0, baseManaCost + Σmodifiers.costDelta)
definition: JsonElement       // has "normal" and "inverted" sections
handlerKey?: string
isInverted: bool              // intrinsic state; preserved across all minion↔card transitions
// REACTIVE TRIGGERS: in a trigger-hosting zone, ICardHandler registers reactive ITrigger(s), live by zone, dropped on play/discard/return — HAND → player-choice intervention; SECRET zone → auto-resolve. (Most cards have none.)
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
  snapshot: MinionOnBoard   // full death state; carries definitionKey + isInverted → card fully derivable
  diedOnTurn: int

GraveyardSpell : GraveyardEntry
  definitionKey: string     // a spell has no board snapshot → stores its own identity
  isInverted: bool          // → card fabricated from these on recast

GraveyardWeapon : GraveyardEntry
  weaponState: WeaponOnHero // carries definitionKey → card derivable
  destroyedOnTurn: int

GraveyardArtifact : GraveyardEntry
  artifactState: ArtifactOnBoard // full snapshot; both DISCARD and DESTROY land here (2026-06-11)
  destroyedOnTurn: int
```

### Supporting types
```
WeaponOnHero        — definitionKey: string, attack: int, durability: int
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
