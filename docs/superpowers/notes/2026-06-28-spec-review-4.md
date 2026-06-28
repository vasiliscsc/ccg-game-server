# Spec Review #4 — game-mechanics.md (two-lens: systems architect + game designer)

**Date:** 2026-06-28
**Reviewer pass:** fourth full read-only review of `specs/2026-05-26-game-mechanics.md`, after review #1 (session-9, 33 findings, applied), #2 (F1–F19 + D1–D4, applied), and #3 (G1–G8 + DR1–DR6, applied). Read end-to-end (§1–§4 + Unaddressed Features), with two explicit lenses per the request.
**Status:** ▶ **WALK IN PROGRESS** (started session 23, 2026-06-28) — one-by-one, present-tradeoff-+-recommend, like reviews #1/#2/#3. See "▶ Walk progress" for per-finding records. **A2 + A3 APPLIED.** A4 clarified (attack action/event walkthrough recorded — supports drop), decision deferred to session 24. A1 deferred to session 24. A5/GD1–GD4 not yet walked.
**Method:**
- **Lens 1 — senior systems architect:** contradictions that block implementation; things speced as a *one-off exception* that the engine already generalizes; logical inconsistencies / under-specified seams.
- **Lens 2 — game designer:** limitations, fun-killers, and decisions whose *gameplay consequence* may not match intent.

> **Calibration.** Three prior thorough reviews + a fix pass have drained the hard contradictions, so this pass is mostly **classification/terminology contradictions, one redundant-exception, one unspecified mechanism, and two real game-feel consequences of already-locked mechanisms.** Findings are `A`-tagged (architect) and `GD`-tagged (game designer) to avoid collision with the `#` / `F`/`D` / `G`/`DR` series. Severity H/M/L; category CONTRADICTION / REDUNDANT-EXCEPTION / UNDER-SPEC / LIMITATION / FUN-RISK.

---

## ▶ Walk progress

**Session 23 (2026-06-28) — walking one-by-one, triage order.**
- **A2 + A3 ✅ APPLIED (one shared fix — mechanical correctness, no fork).** §4 ③′ step 1 rewritten: the ③′-exemption now names **two disjoint classes** — (i) system/effect-issued *openers* (`StartInterventionAction`/`StartChoiceAction`, which also bypass Phase Guard) and (ii) player-submitted, phase-gated *responses* (`RespondInterventionAction`/`SubmitInterventionAction`/**`SubmitChoiceAction`**, which do **not** bypass Phase Guard). Unifying property = "window/choice machinery, not a state-mutating board move," **not** "system action." `SubmitChoiceAction` (A3) folded into the exemption (previously omitted → would have opened a spurious window when resolving a Discover); `SubmitMulliganAction`/`SurrenderAction` noted as also non-declaring. Cross-notes added at §2A `SubmitChoiceAction` row + the "Responder-initiated (off-turn)" header (player-submitted, not system; ③′-exempt). Grep-clean.
- **A4 ⏸ CLARIFICATION PROVIDED — decision deferred to session 24 (user's call).** Before deciding, the user asked for the full action+event sequence of an attack to verify every step stays interceptable and the UI has enough to render. Walked it: **`AttackAction`** (③′ = "redirect/cancel the whole attack") → ④ (`AttackPerformedEvent` + `StealthBrokenEvent`, enqueue strike + retaliation `DealDamageAction`s) → **strike** (own ③′ = "prevent/redirect/immune this hit", ④ `DamageTakenEvent`) → **retaliation** (own ③′, ④ `DamageTakenEvent`) → ⑦ death wave (`MinionDiedEvent`, deathrattles each fully-pipelined). **Conclusion (recorded, supports DROP):** (a) interception is complete & granular via the **general** `ActionDeclaredEvent` filtered by carried action type — `AttackAction`'s ③′ for attack-redirect, each `DealDamageAction`'s ③′ for damage-prevent; `AttackDeclaredEvent` only duplicates the `AttackAction` ③′. (b) The UI never receives *either* declaration event (both **bus-only**) — it renders from `AttackPerformedEvent` (swing anchor) + the two `DamageTakenEvent`s + status events + the public `InterventionWindowOpenedEvent`. So dropping `AttackDeclaredEvent` costs nothing on either axis. **User to confirm drop-vs-demote next session.**
- **A1 ⏸ DEFERRED to session 24 (user's call — stopping for the day).** Fork unchanged: nested drain (recommended) vs front-priority queue. No edit.
- A5, GD1–GD4: not yet walked.

---

## A. Architect findings

### A1 — [UNDER-SPEC / **M–H**] The "immediate response resolution" mechanism is unspecified and conflicts with the documented queue primitive

**Where:** §4 "Response resolution" (:1222–1224); §3 `IActionQueue` (:559–568); §2A `ResolveCardAction` F19 note ("No sub-drain", :331).

"Response resolution" is stated to be **load-bearing, not an optimization**: a window response's effects "resolve to completion immediately … **before returning to drain the pre-existing queue**" — otherwise the pre-existing cascade drains to the ⑦ settle and removes the very minion a save targeted. So a response's enqueued actions must execute **ahead of** whatever was already queued (e.g. actions a ⑥′-triggering action enqueued at ⑤).

But the only front-of-queue primitive, `IActionQueue.EnqueueFront`, is annotated **"high priority (front of queue — death resolution only)."** Two problems:

1. **The real consumer isn't named.** Immediate response resolution is exactly a "jump ahead of the pre-existing queue" operation, yet it is never said to use `EnqueueFront` (or any named mechanism). It reads as a *nested sub-drain* ("processes the response's enqueued actions … fully, before returning") — but the F19 resolution explicitly **deleted** the Battlecry sub-drain and says "No sub-drain," so the spec now has no sanctioned sub-drain vocabulary while a different sub-drain is silently required here.
2. **The annotation looks stale.** Death resolution (⑦) runs **only at queue-empty**, so its Phase-2 deathrattles / Phase-3 reborns enqueue into an *empty* queue and need no front-priority. So "death resolution only" describes a use that doesn't obviously need front-priority, while the use that *does* (response resolution) is excluded by that very annotation.

**Why it matters (architect):** this is the single most load-bearing ordering rule in the engine (the save-before-settle guarantee), and its realization is left to a primitive whose documented contract forbids the use. An implementer cannot build it from the interface as written.

**Recommendation:** name the mechanism explicitly. Either (a) generalize `EnqueueFront` to "front-of-queue priority — used by death resolution **and** immediate window-response resolution," and specify that a response front-enqueues its actions and the engine drains them (and their inline chains) before resuming the parked queue; or (b) specify a bounded **nested drain** for responses (and reconcile the F19 "no sub-drain" wording so it's clearly scoped to the *Battlecry summon* case, not a blanket ban). Pick one and pin the epoch behavior (response sub-actions take fresh `currentActionEpoch`s).

### A2 — [CONTRADICTION / **M**] `RespondInterventionAction` / `SubmitInterventionAction` are called "system actions," but they are player-submitted and phase-gated

**Where:** §4 ③′ step 1 (:1097) vs §2A "Responder-initiated (off-turn)" (:340, :344–345) and §4 ② / line 1069.

§4 ③′ lists *"The `Start*` / `Respond*` actions (`StartInterventionAction`, `StartChoiceAction`, `RespondInterventionAction`, `SubmitInterventionAction`) are **system actions** that … have no ③′."* But:

- §2A explicitly classifies `RespondInterventionAction`/`SubmitInterventionAction` under **"Responder-initiated (off-turn) — submitted by the non-active player … Player-submitted"** (exempt only from the *turn-ownership* check).
- §4 ② line 1069 defines **"System actions bypass Phase Guard"** — yet these two are **phase-gated** (the ② table accepts `RespondInterventionAction` only in `InterventionDecision`, `SubmitInterventionAction` only in `InterventionExtension`). So they demonstrably do **not** bypass Phase Guard.

So the same two actions are simultaneously "system actions (bypass Phase Guard)" and "player-submitted (phase-gated)" — a direct classification contradiction. (`StartInterventionAction`/`StartChoiceAction` *are* genuinely system/effect-issued; only the two `Respond*`/`Submit*` ones are misfiled.)

**Recommendation:** introduce a clean third category — **"player-submitted, phase-gated, ③′-exempt"** — and move `RespondInterventionAction`/`SubmitInterventionAction` into it. Their real property is "do not publish `ActionDeclaredEvent` / have no ③′," **not** "system / bypass Phase Guard." One sentence in §4 ③′ + a label tweak in §2A.

### A3 — [CONTRADICTION / **L–M**] `SubmitChoiceAction` is omitted from the ③′-exemption list, so by the letter it opens a spurious intervention window

**Where:** §4 ③′ step 1 exemption list (:1097); §2A `SubmitChoiceAction` (:337).

The window-machinery exemption (#27) names `StartInterventionAction`, `StartChoiceAction`, `RespondInterventionAction`, `SubmitInterventionAction` as not publishing `ActionDeclaredEvent`. **`SubmitChoiceAction` is missing.** It is a player-initiated action (§2A), so by the rule "every game action declares at ③′" it would publish `ActionDeclaredEvent` and open an (unconditional) decision window when a player resolves a Discover/Target/Modal choice — i.e. the opponent gets a window to "intercept the act of picking option B." That is incoherent (the *continuation* actions it re-enqueues are the interceptable units, and each declares on its own) and asymmetric with its sibling `SubmitInterventionAction`, which **is** exempt.

**Recommendation:** fold `SubmitChoiceAction` into the same "player-submitted, ③′-exempt" category as A2 (it is window/choice machinery, not a state-mutating board move). Same one-line fix resolves A2 and A3 together. (Worth also stating the obvious for `SubmitMulliganAction`/`SurrenderAction` while there — they should not declare at ③′ either.)

### A4 — [REDUNDANT-EXCEPTION / **M**] `AttackDeclaredEvent` is a per-action-type `*Declared` event that contradicts the spec's own "no `*Declared` taxonomy" principle

**Where:** §2B `AttackDeclaredEvent` (:427) and `ActionDeclaredEvent` (:477); §4 ③′ (:1094–1096).

`ActionDeclaredEvent`'s own definition states the design principle: *"subscribers filter on the carried action's runtime type/params via the condition library, so **no per-action-type `*Declared` taxonomy is required**."* Yet `AttackDeclaredEvent` is exactly such a per-type declaration. And because ③′ publishes `ActionDeclaredEvent` for **every** action that reaches dispatch (:1096) and `AttackAction` is not exempt, an attack **double-declares** at its ③′: both `ActionDeclaredEvent { action: AttackAction }` *and* `AttackDeclaredEvent` fire at the same checkpoint, carrying the same attacker/target. The spec asserts (:477) that **both** are "mechanically load-bearing" — but a "when this attacks" interception is already expressible as `ActionDeclaredEvent` filtered to `is AttackAction` (the general mechanism the spec mandates everywhere else), so the typed event is redundant, not load-bearing.

**Why it matters (architect):** it's a generalization speced as an exception — the one surviving member of a taxonomy the spec elsewhere says it doesn't need. It also forces two declaration events on the hot combat path and invites the question "why not `DamageDeclaredEvent`, `SummonDeclaredEvent`, …?" (the answer the spec gives for those is "use the condition filter" — which applies to attacks too).

**Recommendation:** drop `AttackDeclaredEvent`; express "when this attacks" / redirect triggers as an `ActionDeclaredEvent` condition (`Subject`/type filter on `AttackAction`, reading `attackerId`/`targetId`). If a typed event is retained purely as **performance sugar** (avoid every attack-redirect trigger scanning all `ActionDeclaredEvent`s), say *that* explicitly and demote it from "load-bearing" to "a routing optimization the bus may use" — and note the same optimization is available to any hot action type. Either way, remove the internal contradiction.

### A5 — [UNDER-SPEC / **L**] ⑥ (aura recalc) publishes events, but ⑤ is "the publish stage" — the publish-timing/visibility of ⑥-emitted events is unpinned

**Where:** §4 ⑥ (:1113) vs ⑤ (:1107–1109); §2B event header (:376); §3 `IEventBus` epoch model (:548–555).

⑤ is defined as *the* event-publication stage (events stamped `originEpoch`, dispatched with the `birthEpoch < originEpoch` filter). But ⑥ aura recalc *also* publishes: *"A minion newly at ≤0 `maxHealth` (an aura-loss death) enters pending-death (**publishing `MinionMortallyWoundedEvent`**)."* The spec never says what `originEpoch` a ⑥-published event carries, whether it goes through the same snapshot/epoch-filtered bus dispatch, or whether the **⑥′ gather** (which collects "this action's events") picks it up so a save window opens for the aura-collapsed minion. (F16 already says such a wound is heal-*unsavable*, but a window may still open — that interaction depends on whether the ⑥ event is gathered.)

**Recommendation:** one clause: ⑥-published events take the current action's `originEpoch` and ride the same ⑤-style dispatch + ⑥′ gather (so they're treated identically to handler-emitted events), or explicitly carve them out. Low severity (no card breaks today) but it's an unstated hole in the otherwise-tight epoch model.

---

## B. Game-designer findings

### GD1 — [FUN-RISK / **M–H**] The carry-mana model silently taxes hand-invoked interventions into your *own* next turn

**Where:** Turn Lifecycle step 4 (:1206); §3 "The press is the rationed resource" (:800); §4 "Response resolution" (:1222).

Mana is refreshed **only at turn-end** (for the ending player) and there is **no turn-start refresh** (Turn Lifecycle has no such step). The ending player carries their full refreshed pool through the opponent's turn for interventions, and — per the spec — *"the player's own next turn simply begins with **whatever interventions didn't consume**."*

**Consequence (likely under-weighed):** every mana you spend invoking a hand sigil during the opponent's turn is mana you **do not get back** at the start of your own turn — you only refill at *your* next turn-end. So a 1-mana invoked intervention effectively costs **the card + 1 tempo on your following turn**, and in the extreme a player can invoke themselves down to a crippled (even 0-mana) own turn. This is the deliberate flip-side of "no `reservedMana` field" (chosen session 6), but the *gameplay* cost curve is steep, opaque, and asymmetric:

- **Asymmetric with inscriptions**, which are pre-paid on your own turn and never touch your future mana — so the "invoke from hand" mode is strictly more expensive in a way that isn't surfaced anywhere a player sees.
- **Counter-intuitive for HS-trained players**, who expect a full mana bar at turn start; discovering turn-3 mana is short because they reacted on turn 2 reads as a bug, not a cost.

**Recommendation (designer call, present-tradeoff):** confirm this is the intended cost of hand-invocation. If yes, it's a strong knob (interventions are *real* commitments) but needs heavy UI telegraphing ("reacting now spends from next turn"). If the intent was only "the off-turn pool is your ramped pool" (the *ceiling* framing in :1206) without the next-turn penalty, the model needs a turn-start top-up to `maxMana` (re-introducing something `reservedMana`-like) — i.e. the session-6 simplification has a gameplay tail worth re-examining now that content is about to be designed against it.

### GD2 — [FUN-RISK / **M–H**] Unconditional-window flood, worst in combat — the deferred "decision-window budget" undercounts it

**Where:** §3 "How the window is delivered — unconditional + two-stage" (:796); §4 ③′/⑥′ (:1098/:1121); Unaddressed "Intervention decision-window real-time budget" (:1274–1280); combat = two `DealDamageAction`s (§2A `AttackAction` :332).

A Stage-1 **decision** window opens at **every** ③′ **and** every ⑥′, **unconditionally** (even with zero matching cards), so window *existence* leaks nothing. The press cap (`MaxInterventionsPerTurn`) limits only Stage-2 *escalations*, not these decision windows. The Unaddressed entry acknowledges the resulting real-time waste but frames it as *"one short window per action."*

**That undercounts the real number.** A single attack is **three** actions — `AttackAction` + strike `DealDamageAction` + retaliation `DealDamageAction` — each with a ③′ **and** a ⑥′ → up to **six** decision windows for one attack. `skip-all` is scoped to one action's ③′+⑥′, so even a player spamming skip-all faces ~**3 prompts per attack**; `autoSkipAll` removes them all but is all-or-nothing (you either never get to intervene, or you wade through the storm). A board trading several attacks per turn becomes a prompt gauntlet for the non-active player, with the active player's clock paused throughout.

**Why it's a real fun risk:** the entire value of the *hand-invoke* intervention layer (vs. pure auto inscriptions, the HS-Secret model) is bought with this window cost — and DR1's `Uncounterable`/`Inexorable` only elides ③′ for *flagged card plays*, not for combat/damage actions or any ⑥′, so DR1 does **not** relieve the combat case.

**Recommendation:** weigh three levers and decide before content hardens — (a) the deferred per-turn real-time budget on decision windows (auto-skip once exhausted); (b) collapsing combat's per-action windows (e.g. one declaration window per *attack* rather than per strike/retaliation — but this reopens the "combat is two independent actions" ⑥′-uniformity decision); (c) a smarter `autoSkipAll` that only auto-skips **empty-candidate** windows (accepting the small "I might hold something" inference, which the unconditional design was specifically built to deny — so this trades a sliver of hidden-info for pacing). At minimum, correct the Unaddressed entry's "one window per action" to reflect the combat multiplier, since that estimate is what justified deferring the fix.

### GD3 — [LIMITATION / **L**] Pending-death (mortally-wounded) bodies linger longer than players expect — the design leans on UI the engine can't enforce

**Where:** §1 `currentHealth` (:103); §4 cadence (:1047); §3 `All…/Alive…/Dead…` (:764–772).

The cascade-settle model keeps a 0-HP minion **on the board — targetable, buffable, countable, keyword-active** — until the *whole* cascade drains (⑦), which can span many actions and multiple windows. This unlocks the genuinely good "dying-matters" design space (overkill, save-from-lethal, count-the-doomed). But it also means players routinely see a 0-HP minion still sitting on the board and still interactable — a state HS never shows for more than an instant (HS death-checks fire after almost every action). The distinction between a *mortally-wounded* (savable, fires `MinionMortallyWoundedEvent`) and a *destroy-marked* (silent, final) 0-HP body is invisible without UI help.

**Why note it (designer):** the mechanic's fun depends entirely on players *reading* the dying state correctly, and `CCG.GameLogic` is UI-agnostic — so the spec is implicitly committing the client to a clear "mortally wounded / will die at settle" telegraph (and to distinguishing it from destroy-marked). That's a real content/UX dependency riding on an engine choice; flag it so the client/content plan owns it rather than discovering it in playtest.

**Recommendation:** no engine change. Record a client/UX requirement ("render pending-death state, and the savable-vs-marked distinction") in the plan, and have v1 content introduce dying-window interactions gradually so the state is taught, not sprung.

### GD4 — [LIMITATION / **L**, informational — already decided] Heroes cannot attack at all; an entire HS axis (weapons / hero aggression / equip archetypes) is cut

**Where:** §1 `PlayerState` (:74); Unaddressed "Hero combat (RESOLVED — concept dropped)" (:1309–1311).

Heroes have no attack, no weapons (the `Weapon` card type is gone), no frozen state; the artifact row is the whole hero kit. This was decided and re-affirmed (session 9), so it is **not** a bug — but as a *design-space* observation it removes: weapon classes, the "equip + swing face" aggressive plan, weapon-durability/charge synergies, and the tactical decision of hero-into-minion trades. The artifact row (passive/active/durability/charges) is expressive, but it is a *different* axis (mostly value/utility), not a substitute for board-presence-via-hero-attack.

**Why note it (designer):** flagging so the content plan consciously confirms the artifact row carries enough *aggressive* and *tempo* design space to replace what weapons gave HS — rather than assuming parity. Purely informational; no spec change implied.

---

## C. Checked and consistent (coverage — verified clean this pass)

- **Mana ramp under the carry model** — traced setup-seed → per-player turn-end refresh: P2's first turn = 1 mana, each player's Nth own-turn = N mana (cap 10). The ramp itself is correct (F1's both-player seed is what makes it work); GD1 is about the *off-turn-spend* tail, not the ramp.
- **Epoch / creation-visibility model** (§3 `IEventBus`) — equal-epoch self-exclusion, `birthEpoch < originEpoch`, the F19 self-Battlecry divergence — all internally consistent and correctly documented as a deliberate HS divergence (Unaddressed Features).
- **Death-cadence settle loop** (§4 ⑦–⑨, amendment B; G8 uniform cadence) — consistent; no stale per-action-death or pre-halt language survives.
- **`IWinCondition` seam** (§3, §4 ⑧; DR5) — registration shape, pull-at-settle rationale, mutual-winner→draw all consistent; v1 ships zero registered conditions.
- **Friendly/Enemy lane algebra** (§3 `IEntityPredicate`) and its reuse by `ITriggerCondition`/`ITargetSelector` — coherent across all three; neutral-source vs player-source cases correct.
- **`All…/Alive…/Dead…` trichotomy**, the health-only predicate choice, destroy-mark-in-`Alive…` rationale — consistent.
- **Directed/public event split** (§2B) and the no-leak invariants (Discover options, drawn-card identity, intervention candidates) — consistent with the visibility model (§3).
- **Source Attribution** (`sourceId` never inherited) vs **keyword-effects-on-dead-source** (effective-keywords pulled from live board) — the split is consistent and its limitation correctly deferred.
- **Reborn charge** (`rebornAvailable` init-true, keyword-independent, copy=false, Transform=true), **fizzle→RFG for minion cards**, **full-hand/full-board/full-lane** (#21 destroy vs full-board-summon fizzle) — all consistent.
- **RNG** (counter-based per-action `mix(seed, epoch)`), **replay** (command-log canonical, event-log wire), **trace** (`ITraceSink`) — consistent and self-contained.

---

## Suggested triage order (for the walk)

1. **A2 + A3** — one shared fix (a "player-submitted, ③′-exempt, phase-gated" category); cheap, removes a real classification contradiction. Do them together.
2. **A4** — decide: drop `AttackDeclaredEvent` (generalize) or demote it to documented perf-sugar. Removes an internal "load-bearing" contradiction.
3. **A1** — name the immediate-response-resolution primitive + reconcile `EnqueueFront`'s annotation. Highest implementation-risk item; worth a careful call.
4. **A5** — one clause on ⑥-published-event timing/visibility.
5. **GD1, GD2** — the two design calls worth deciding **before content design hardens** (mirror DR1/DR2's priority): the intervention mana cost-curve, and the window-flood pacing. Both are consequences of *locked* mechanisms, so the question is "confirm/tune," not "redesign."
6. **GD3, GD4** — record as plan/UX/content notes; no engine decision.

---

## Plan impact (fold into the end-of-pass plan reconciliation, if/when applied)

- **A1** — the pipeline/queue epic: `IActionQueue` ticket documents the front-priority primitive used by **both** death resolution and window-response resolution (or specifies the nested response-drain); the ⑥′/③′ response-resolution tickets reference it. Scenario test: a save resolves before a same-cascade `DealDamageAction` already queued ahead of it.
- **A2 + A3** — Epic ~02 action-classification / Epic ~03 pipeline: the ③′ checkpoint ticket's exemption set is **{`StartInterventionAction`, `StartChoiceAction`, `RespondInterventionAction`, `SubmitInterventionAction`, `SubmitChoiceAction`, `SubmitMulliganAction`, `SurrenderAction`}** — and only `Start*`/`Spawn*`-style truly system actions bypass Phase Guard; `Respond*`/`Submit*` are player-submitted + phase-gated. No "system action" label on the latter.
- **A4** — Epic ~02 events / Epic ~03 ③′: either delete `AttackDeclaredEvent` (and the combat redirect tickets express the hook via an `ActionDeclaredEvent` condition on `AttackAction`), or keep it explicitly as a bus-routing optimization with a note that it is not a distinct semantic hook.
- **A5** — pipeline epic ⑥ ticket: pin ⑥-published `MinionMortallyWoundedEvent` to the current action's `originEpoch` + normal dispatch + ⑥′ gather.
- **GD1** — turn-lifecycle / mana ticket + a balance/telemetry note: instrument how often interventions cut into the invoker's next turn; if the penalty is unintended, add a turn-start top-up. Client must telegraph "reacting spends next-turn mana." Playtest-watch.
- **GD2** — pipeline/intervention epic: correct the Unaddressed "decision-window budget" estimate; if pursued, the per-turn real-time budget (one counter + `GameConstant`) and/or empty-candidate auto-skip is the additive fix. Playtest-watch.
- **GD3** — client/content plan: render pending-death (mortally-wounded vs destroy-marked) state; stage v1 dying-window content to teach it.
- **GD4** — content plan: confirm the artifact row covers the aggressive/tempo design space weapons/hero-attack would have. Informational.
