# Spec Review #2 — post-fix-pass audit

**Date:** 2026-06-17
**Reviewer:** read-only full audit of `docs/superpowers/specs/2026-05-26-game-mechanics.md` (1250 lines, §1–§4 + Unaddressed Features).
**Purpose:** catch inconsistencies, logical bugs, stale text, and wording problems **before implementation** — cheaper to fix the spec than thousands of lines of code.
**Relationship to the prior review:** distinct from `notes/2026-06-12-spec-review-findings.md` (the 33-finding doc, all since applied in the fix pass). This is a fresh read of the *current* spec. Findings here are numbered **F1…** to avoid collision with the old `#1…#33`.

**Status legend:** each finding is `[CATEGORY / Severity]`.
Categories: **CONTRADICTION** (two parts of the spec disagree) · **BUG** (logic produces a wrong/unintended outcome) · **HOLE** (underspecified / missing rule) · **DRIFT** (stale text from a superseded decision) · **WORDING** (clarity only) · **DESIGN** (a question or improvement, not a defect).

**Summary:** 3 high, 7 medium, 7 low/wording, plus a design-suggestions section. None are show-stoppers; the headline (F1) is a real gameplay bug that would surface in the very first match.

---

## ▶ Walk progress (as of 2026-06-19, end of session 16)

**Applied to spec:** F1 ✅, F2 ✅, F3 ✅, F4 ✅ (B), F5 ✅ (a), F6 ✅, F7 ✅, F8 ✅ (session 15); **F9 ✅ (lock-now), F10 ✅ (a, remove-from-game), F11 ✅, F12 ✅, F13 ✅, F14 ✅, F15 ✅, F16 ✅, F17 ✅ (session 16).**
**F18 — RAISED then RETRACTED (session 16), no formula change.** During the F16 walk I mis-stated HS and proposed a "signed-Δ" fix to the #13 fall formula. **The user corrected the HS rule:** removing a Health buff **never kills** in HS — current Health is reduced only down to the printed stat, and if already below it, unchanged. That is *exactly* what the spec's existing `min(currentHealth, newMaxHealth)` already does, so **F18 is invalid; the formula stays `min(currentHealth + max(0, Δmax), newMax)`.** I did apply a **prose tightening** to §1 line 95 (replaced the misleading "existing damage persists" with the correct clamp-down/never-kills wording; sole death path = `maxHealth ≤ 0` from a *negative* buff/aura).

**F19 ✅ RESOLVED + APPLIED 2026-06-21 (session 18) — the SIMPLIFICATION.** The sub-drain mechanism (decided 2026-06-20) was **rejected by the user as unclean and reverted.** Replacement: the **existing summon flow** — `ResolveCardAction` ④ summons the body (place + register `OnSummon` + publish `MinionSummonedEvent`, all inline, *identical to a token's `SummonMinionAction`*) and then **enqueues the Battlecry after**. Because the Battlecry is enqueued before the summon event publishes (⑤), it sits ahead of summon-reactions → resolves first → other minions' "after you summon" reactions fire after the Battlecry (HS-faithful, unchanged). The cost: the minion's triggers are live during its own Battlecry, so it **does** react to its own Battlecry's events — line 731's "Battlecry before triggers go live" is **dropped as a deliberate, documented divergence from HS** (HS verified 2026-06-21: an HS minion never triggers off its own Battlecry). Own-*summon*-event self-exclusion still holds via the equal-epoch filter; keywords (pulled) and Deathrattle (snapshot) unaffected. Zero new machinery — no `FinalizeSummon`, no DFS, no sub-drain. The reversible path back to the HS rule (`FinalizeSummonAction` + `EnqueueFront`-generalized DFS) is recorded in the spec's **Unaddressed Features → "A minion triggers off its own Battlecry."** Touch-points reworked: §2A `ResolveCardAction` ④, §3 line 731, §1 `MinionOnBoard` TRIGGERS comment, §3 `IEventBus` note + the new Unaddressed entry. See the F19 section for the full write-up.

**✅ D1–D4 DONE (2026-06-21).** D1 = active player can't reactively save self on own turn (by design) → **corollary note added** at §3:786. D2 = `SpellCastEvent` timing → **premise overturned on HS verification; event MOVED commit → resolution** (countered spell fires no cast trigger; current-HS-correct, symmetric with `MinionSummonedEvent`). D3 = Reborn simplification → no edit (subsumed by F5). D4 = press cap a balance lever → no edit. See the D-section below for full records.

**✅ SPEC-REVIEW-2 COMPLETE (F1–F19 + D1–D4 all applied/resolved).** Next: end-of-pass plan reconciliation (fold in F1–F17 + F19 + D2's SpellCastEvent move) → Epic 01 / T1.1.

Each resolved finding carries a `✅ DECIDED + APPLIED` line under its heading with the touch-points.

---

## A. High severity

### F1 — [BUG / H] The going-second player begins their first turn with 0 mana   ✅ DECIDED + APPLIED (2026-06-17)

**▶ Resolution:** seed **both** players `mana = maxMana = StartingMana (1)` at setup. Applied to spec §1 `GameConstants.StartingMana` comment, §1 `PlayerState.mana/maxMana` comment, and §4 Match Setup step 8. Carry-on-opening-turn behavior (P2's seed doubles as their off-turn intervention pool during P1's turn 1) accepted as consistent with the model.

**Where:** Match Setup step 8 ([2026-05-26-game-mechanics.md:1144](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1144)) + Turn Lifecycle step 4 ([:1155](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1155)) + `GameConstants.StartingMana` ([:53](docs/superpowers/specs/2026-05-26-game-mechanics.md#L53)).

**Problem.** The mana model was moved to **turn-END refresh** (session 6): step 4 refreshes *"the player whose turn is ENDING"*, pre-loading their carry pool. Setup step 8 then seeds **only the first player**: *"Seed the **first player's** `mana = maxMana = StartingMana` (1) directly."* The second player is left at the `mana = maxMana = 0` from setup step 1.

Trace it through:

| Moment | P1 mana | P2 mana |
|---|---|---|
| Setup (step 1) | 0/0 | 0/0 |
| Setup (step 8, seed first player) | **1/1** | 0/0 |
| P1 turn 1 plays | 1 | (holds 0 for interventions) |
| End P1 turn 1 → step 4 refreshes **P1** | → **2/2** | 0/0 |
| **P2 turn 1 begins** | 2 (carry) | **0** ← bug |
| End P2 turn 1 → step 4 refreshes P2 | 2/2 | → 1/1 |

The going-second player has **0 mana on their first turn** (only The Coin lets them do anything). The turn-end refresh never targets P2 before P2's first turn, and setup never seeds P2. This is not an edge case — it happens every match.

**Why it slipped in:** the turn-end refresh works for every turn *except each player's opening turn*, which needs an explicit seed; setup seeds only one of the two.

**Recommendation.** Seed **both** players at setup: `mana = maxMana = StartingMana (1)` for player 1 **and** player 2. This is symmetric and correct (each ramps 1→2→3… on its own turn-end): P1 turn1 = 1, P2 turn1 = 1 (+Coin), P1 turn2 = 2, P2 turn2 = 2, … — HS-faithful. (Reverting to turn-start refresh is the wrong fix; it would break the off-turn intervention-pool carry the turn-end model exists for.) Update step 8 and the `StartingMana` comment accordingly.

---

### F2 — [CONTRADICTION / H] `SubmitInterventionAction` is defined twice, with conflicting signatures; the two-stage responder actions are filed under "System/effect-issued"   ✅ DECIDED + APPLIED (2026-06-17)

**▶ Resolution:** stale single-stage row deleted; `RespondInterventionAction` + `SubmitInterventionAction` relocated into a new **"Responder-initiated (off-turn)"** sub-table in §2A (player-submitted, exempt from ③ turn-ownership), `StartInterventionAction` left under System/effect-issued.

**Where:** §2A — stale row ([:323](docs/superpowers/specs/2026-05-26-game-mechanics.md#L323)) vs current rows ([:352-354](docs/superpowers/specs/2026-05-26-game-mechanics.md#L352-L354)).

**Problem.** The session-13 two-stage rework split the responder flow into `RespondInterventionAction` (Stage-1 decision) + `SubmitInterventionAction` (Stage-2 play) and added those rows to the **System/effect-issued** table (lines 352–354). But the **old single-stage `SubmitInterventionAction` row was never removed** — it still sits in the **Player-initiated** table at line 323 with the pre-session-13 contract:

- line 323: field `playerId`; `chosenCardId? (null = **skip** — ends the window loop)`; *"A play re-opens the window over the remaining offer; a skip ends it."* — the **single-stage** model (skip and play both via this one action).
- line 354: field `respondingPlayerId`; `chosenCardId? (null = **back out** … the intervene-press was already counted)`; Phase Guard `InterventionExtension` **only**; **Stage-2** model (skip is now `RespondInterventionAction{Skip}` at Stage-1).

These disagree on the field name (`playerId` vs `respondingPlayerId`), on what `chosenCardId = null` means (skip vs back-out), and on which phase accepts it. The §3/§4 prose and all three trace examples ([:955-971](docs/superpowers/specs/2026-05-26-game-mechanics.md#L955-L971)) use the two-stage flow, so **line 323 is the stale outlier.**

Two secondary issues fall out:
- **Mis-categorization:** `RespondInterventionAction` (353) and `SubmitInterventionAction` (354) are **player responses**, but they sit in the *System/effect-issued* table. Only `StartInterventionAction` (352) is genuinely system-issued.

**Recommendation.** Delete the stale row 323. Move `RespondInterventionAction` + `SubmitInterventionAction` into the Player-initiated table (or a small "Responder-initiated (off-turn)" sub-table), keeping `StartInterventionAction` under System/effect-issued. No semantic change — pure de-duplication of a superseded row.

---

### F3 — [CONTRADICTION / H] Intervention iteration described as "bounded by mana, not a hard cap" — but the per-turn press cap *is* a hard cap   ✅ DECIDED + APPLIED (2026-06-17)

**▶ Resolution:** confirmed the strict reading — **each chained play is a fresh intervene-press**, so a responder answers at most `MaxInterventionsPerTurn` (3) cards across the entire opponent turn. Reworded lines 992, 1056, 1189 to "bounded by **both** the per-turn press budget **and** next-turn mana, whichever binds first."

**Where:** the "not a hard cap / bounded by next-turn mana" claims at [:992](docs/superpowers/specs/2026-05-26-game-mechanics.md#L992), [:1056](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1056), [:1189](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1189) vs the press-cap model at [:780](docs/superpowers/specs/2026-05-26-game-mechanics.md#L780) and `MaxInterventionsPerTurn` ([:59](docs/superpowers/specs/2026-05-26-game-mechanics.md#L59)).

**Problem.** Under the two-stage model, **each played intervention costs one press**: after a `SubmitInterventionAction` (Stage-2) resolves, the loop *"re-opens a fresh **Stage-1** decision window"* (lines 354, 1186), and reaching Stage-2 again requires pressing **intervene** (`turn.interventionsUsed++`). Line 780 confirms: *"each re-declare / re-offer play is a fresh press."* With `MaxInterventionsPerTurn = 3`, a responder can play **at most 3 intervention cards per active-player turn**, across all of that turn's windows.

But three places still carry the **session-6 single-stage** framing, where one press allowed unlimited re-declare plays bounded only by mana:
- line 992 (Deterministic Ordering): *"Iteration … is bounded by next-turn mana, **not capped**."*
- line 1056 (③′ re-declare loop): *"The loop is bounded by the responder's next-turn mana, **not a hard cap**."*
- line 1189 (PendingIntervention): *"chain several of their *own* held cards (**bounded by next-turn mana**)."*

So the spec simultaneously says iteration is uncapped (mana-only) and capped at 3 presses/turn. The press cap wins; the "not a hard cap" lines are stale.

**Recommendation.** Reconcile to: *iteration is bounded by **both** the per-turn press budget (`MaxInterventionsPerTurn`, a hard cap) **and** the responder's available mana — whichever binds first.* Update lines 992, 1056, 1189. (This also clarifies F-adjacent reasoning: the Unaddressed-Features "decision-window real-time budget" entry already correctly notes the press cap bounds *extension* windows.)

---

## B. Medium severity

### F4 — [HOLE / M] `ITargetSelector.Filter(selector, ITriggerCondition)` reuses an event-shaped predicate as an entity predicate — the interfaces don't line up   ✅ DECIDED + APPLIED (2026-06-17, option B)

**▶ Resolution:** chose **B (unify)** for maximum design space — factored a shared **`IEntityPredicate`** (new §3 subsection) holding the entity-property checks (`Friendly`, `Enemy`, `IsSelf`, `IsDamaged`, `IsMortallyWounded`, `HasTribe`, `HasKeyword`, `MinionTypeIs`, `CardTypeIs`, `CostAtLeast/AtMost`, …) + `All`/`Any`/`Not`. `ITargetSelector.Filter(selector, IEntityPredicate)` uses them directly (negation now expressible inside a Filter); `ITriggerCondition` is reframed as the **event** layer = event-structural primitives (`SelfIs*`) + `Subject(p)` over an `IEntityPredicate`. The lane-based Friendly/Enemy formalism **moved** into `IEntityPredicate`; `MinionsWithTribe`/`MinionsWithKeyword` reduced to named sugar over `Filter(…, HasTribe/HasKeyword)`; `Not` promoted to a first-class combinator; the old `HasTribe`(condition)/`MinionsWithTribe`(selector) duplication dissolved. **Plan impact:** the Epic-08 `ITargetSelector` ticket and the T8.10 `ITriggerCondition` ticket now both build on a new `IEntityPredicate` unit (reconciliation).

**Where:** `Filter` factory + `IsDamaged` example ([:740](docs/superpowers/specs/2026-05-26-game-mechanics.md#L740)); `ITriggerCondition` interface ([:651](docs/superpowers/specs/2026-05-26-game-mechanics.md#L651)); condition catalog ([:674-690](docs/superpowers/specs/2026-05-26-game-mechanics.md#L674-L690)).

**Problem.** Two coupled gaps:

1. **Signature mismatch.** `ITriggerCondition.Matches(GameEvent evt, GameState state, string sourceId)` is **event-shaped** — every catalog entry inspects *"the event's relevant entity."* But `Filter(selector, ITriggerCondition)` needs to test **each candidate entity** ("is *this minion* damaged?"). There is no event to hand. The spec says *"reuse the condition library as the entity predicate"* but never reconciles how an event-matching predicate evaluates against a bare entity. An implementer hits this on the first `Filter` call.

2. **`IsDamaged` is undefined.** The worked example `Filter(AllFriendlyMinions, IsDamaged)` (line 740) and the Enrage "damaged" reads (line 576) reference an `IsDamaged` predicate, but it is **not in the condition catalog** (which lists only `AlwaysFires`, `SelfIsSource`, `SelfIsTarget`, `SelfIsRelated`, `FriendlyOnly`, `EnemyOnly`, `MinionTypeIs`, `HasTribe`, `CardTypeIs`, `CostAtLeast/AtMost`).

**Recommendation.** Decide the abstraction and write it once:
- **Option A (recommended):** introduce a distinct `IEntityPredicate { bool Matches(EntityId id, GameState state, string sourceId); }` for `Filter`, and add `IsDamaged` (≡ `currentHealth < maxHealth`) to it. Clean separation: conditions gate *events*, predicates filter *entities*. Costs one small interface.
- **Option B:** generalize `ITriggerCondition` so entity-only conditions can be evaluated against an entity (e.g. a synthetic `EntitySubjectEvent` wrapper), and add `IsDamaged`. Keeps one interface but muddies the "conditions filter events" story.

Either way, add the missing `IsDamaged` (or `HealthBelowMax`) entry. Note this is the **`ITargetSelector` library** ticket (Epic 08) the reconciliation will create — fold the resolution into its spec.

---

### F5 — [BUG / M] Reborn granted *after* summon never arms — `rebornAvailable` is snapshotted at summon   ✅ DECIDED + APPLIED (2026-06-17, option a)

**▶ Resolution:** `rebornAvailable` now means "charge unspent" — **initialized `true` for every minion**, firing gated on `keywords has reborn && rebornAvailable`. Reborn copy summoned with it `false` (Phase-3 explicitly overrides the default-true init). `TransformMinionAction` resets to `true`. **Silence (option a) leaves it untouched** — stripping the `reborn` keyword already gates off reborn, so silence-then-re-grant now fires. Applied at §1 `rebornAvailable`, §2A `TransformMinionAction` + `SilenceMinionAction` (×2), §4 ⑦ Phase 3.

**Where:** `MinionOnBoard.rebornAvailable` ([:110](docs/superpowers/specs/2026-05-26-game-mechanics.md#L110)); IKeyword Reborn read ([:575](docs/superpowers/specs/2026-05-26-game-mechanics.md#L575)); §4 ⑦ Phase 3 ([:1092](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1092)).

**Problem.** `rebornAvailable` is *"Initialized at summon to `keywords.Contains("reborn")`,"* and Phase 3 fires reborn only when **`keywords has reborn && rebornAvailable`**. Walk the cases:

| Scenario | `rebornAvailable` at death | `keywords has reborn` | Reborns? | Correct? |
|---|---|---|---|---|
| Summoned with Reborn | true | true | yes | ✓ |
| Reborn copy (summoned `rebornAvailable=false`) | false | true | no | ✓ (no loop) |
| Silenced before death | false (Silence clears it) | false | no | ✓ |
| **Granted Reborn *after* summon** (buff/`grantedKeywords`) | **false** (snapshot at summon) | true | **no** | ✗ |

A *"Give a friendly minion Reborn"* effect adds `reborn` to `grantedKeywords`/`keywords` but never touches `rebornAvailable` (already snapshotted `false`), so the minion will **not** reborn. HS has Reborn-granting effects; even absent one in v1 content, the rule is silently wrong.

**Recommendation.** Make `rebornAvailable` mean *"reborn charge unspent"* rather than *"had reborn at summon"*: **initialize it `true` for all minions**, keep the firing gate `keywords has reborn && rebornAvailable`, and keep the reborn-summon path setting it `false` (and Silence clearing it `false`). Then granted-Reborn works, the no-loop guard is unchanged, and there's no per-source bookkeeping. (Trivially refactorable; a one-line init change.)

---

### F6 — [HOLE / M] The Coin is added before the mulligan, so it can be mulliganed away   ✅ DECIDED + APPLIED (2026-06-17, with F7)

**Where:** Match Setup steps 5 (The Coin) and 7 (Mulligan) ([:1141-1143](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1141-L1143)).

**Problem.** Step 5 puts The Coin in the second player's hand; step 7 then runs the mulligan, where the player *"may replace **any subset**"* of their hand — with nothing excluding The Coin. As written, the second player could mulligan The Coin (shuffling a 0-cost card into the deck, or losing it). In HS the Coin is added **after** mulligan precisely so it can't be mulliganed.

**Recommendation.** Move The Coin to **after** the mulligan completes (i.e. between current steps 7 and 8). Order becomes: deal → sigil registration → mulligan → **Coin** → turn 1. (Minor, but it's a real rules deviation and a tidy reorder.)

---

### F7 — [HOLE / M] Mulligan's interaction with sigil (hand-zone) trigger registration is unspecified   ✅ DECIDED + APPLIED (2026-06-17, with F6)

**▶ Resolution (F6+F7 together):** reordered Match Setup to **deal → mulligan (5) → Coin (6) → sigil registration (7)**. Coin after mulligan ⇒ unmulliganable (F6). Sigil registration once on the *final* post-mulligan hand ⇒ no unregister/re-register on swaps (F7), justified by nothing reactive firing during the Mulligan phase. No numeric cross-refs broken.

**Where:** Match Setup steps 6–7 ([:1142-1143](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1142-L1143)); `Card` reactive-trigger lifecycle ([:154](docs/superpowers/specs/2026-05-26-game-mechanics.md#L154)); hosting table hand row ([:789](docs/superpowers/specs/2026-05-26-game-mechanics.md#L789)).

**Problem.** Step 6 registers opening-hand sigils' hand-zone triggers at `birthEpoch = 0`. Step 7 mulligan can **replace** any of those cards: replaced cards are shuffled back into the deck, and replacement cards are drawn first. But:
- A **mulliganed-away sigil** leaves the hand (shuffle-into-deck) — its hand trigger must **unregister** (the hosting table says a sigil's trigger drops when the card "leaves the hand by any route … shuffle-into-deck"). The mulligan procedure doesn't say it does this.
- A **replacement** card that is itself a sigil must **register** its hand trigger. But the mulligan deal happens "outside the action pipeline" like the opening deal, so it won't go through the `DrawCardAction` handler that normally registers hand triggers — it needs the same explicit registration step 6 does.

Neither is specified, so opening-mulligan sigils could end up with stale or missing registrations.

**Recommendation.** Add a sentence to step 7: the mulligan unregisters hand triggers for replaced (deck-bound) sigils and registers hand triggers (at `birthEpoch = 0`) for replacement sigils, mirroring step 6 — so post-mulligan hand-trigger state is exactly "the sigils actually in the kept/replacement hand."

---

### F8 — [DRIFT / M] `ActionRejectionCode` is missing `NoInterventionBudget` and `NoMatchingCard`   ✅ DECIDED + APPLIED (2026-06-17)

**▶ Resolution:** both added to the §2C enum. Clarified (per the F8 discussion) that they are **server-authoritative guards** — a correct client suppresses the Intervene option when `candidates` is empty / the press budget is spent, so neither fires in normal play; they exist to reject out-of-contract submissions. Cross-ref added in the `RespondInterventionAction` row.

**Where:** §2C enum ([:479-486](docs/superpowers/specs/2026-05-26-game-mechanics.md#L479-L486)) vs §4 ② Phase Guard ([:1020](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1020)).

**Problem.** §4 ② rejects an illegal `Intervene` with `NoInterventionBudget` or `NoMatchingCard`. The §2C enum — explicitly a *"closed set; grows in lockstep with the §4 ②③ validation rules"* — does not list either. The session-13 two-stage rework added the rejection paths but not the codes.

**Recommendation.** Add `NoInterventionBudget` and `NoMatchingCard` to the enum (and double-check the reconciliation's "structured error codes" Epic-01 ticket carries the full closed set).

---

### F9 — [HOLE / M] `ChoiceType` values, `ChoiceOption` shape, and `SubmitChoiceAction.choiceId` are undefined   ✅ DECIDED + APPLIED (2026-06-19, lock-now)

**▶ Resolution:** **lock the shape in §1 now** (chosen as the design-space-maximizing option — the record is already maximal: all payload fields optional + all resolution logic lives in the unconstrained `PendingChoice.context` continuation, so a new mechanic never needs a new option field). Added to §1 Supporting types: `enum ChoiceType { Discover, Target, Modal }` (mulligan is NOT a member — rides `SubmitMulliganAction`) and `record ChoiceOption(string optionId, string? entityId = null, string? definitionKey = null, string? label = null)`. **Dropped `choiceId`** from `SubmitChoiceAction` → `{ playerId, selectedOptionId }`. **Invariant baked in:** `ChoiceOption` is a DISPLAY + ANSWER-ROUTING token, never the effect; `optionId` is the stable per-mode dispatch key the continuation switches on (a Modal mode's effect-spec lives in the card definition's `modes[]`, reached by `optionId` — never carried on the wire). The three patterns were worked through with payloads in the session reply (Discover→definitionKey, Target→entityId, Modal→optionId-only; the "deal 2 damage" number lives in the card definition, not the option). Touch-points: §1 `PendingChoice` block (new `ChoiceType`/`ChoiceOption`), §2A `SubmitChoiceAction` row.

**Where:** `PendingChoice` ([:195](docs/superpowers/specs/2026-05-26-game-mechanics.md#L195)); `SubmitChoiceAction` ([:322](docs/superpowers/specs/2026-05-26-game-mechanics.md#L322)); `StartChoiceAction` ([:351](docs/superpowers/specs/2026-05-26-game-mechanics.md#L351)).

**Problem.** (Already flagged in the session-14 Discover Q&A for the Epic-01 data-model ticket; recording here so the spec audit is complete.) The spec references `choiceType: ChoiceType` and `options: ChoiceOption[]` but never enumerates `ChoiceType`'s values (Discover? mid-effect targeting? mulligan rides its own action — so probably just `Discover` + `Target`?) nor defines `ChoiceOption`'s fields. Additionally, **`SubmitChoiceAction` carries `choiceId`** but nothing else defines a choice id (`PendingChoice`, `ChoicePromptEvent`, `ChoiceStartedEvent` have none); with at most one pending choice at a time it's likely vestigial, and `selectedOptionId` implies `ChoiceOption` has an `id`.

**Recommendation.** Nail down in the Epic-01 data-model ticket: the `ChoiceType` enum members, the `ChoiceOption` record (`{ id, … }` — at minimum the option's payload/target ref), and whether `SubmitChoiceAction.choiceId` is needed at all (drop it if a single in-flight choice makes it redundant; the `selectedOptionId` references `ChoiceOption.id`).

---

### F10 — [DESIGN / M] A fizzled (countered) minion card lands in the graveyard as a `GraveyardMinion` — making it a resurrection/recursion target   ✅ DECIDED + APPLIED (2026-06-19, option a)

**▶ Resolution:** **(a) removed-from-game.** A fizzled minion card writes **no** graveyard entry — it never reached the board and never *died*, so it must not be a resurrection/recursion target, and there's no `diedOnTurn` to define. The instance simply ceases to exist (present in no zone; no tracked exile zone needed). HS-faithful (a countered card is not recyclable). Touch-point: §2A `ResolveCardAction` fizzle branch.

**Where:** `ResolveCardAction` fizzle branch ([:316](docs/superpowers/specs/2026-05-26-game-mechanics.md#L316)).

**Problem.** *"A **minion** card that fizzles goes to the owner's graveyard **here** (built from its `definitionKey` + base stats — it never reached the board)."* That entry is a `GraveyardMinion`, which is exactly what *"resurrect a friendly minion that died"* / *"add the last dead minion's card to hand"* effects read. So a **countered minion that never hit the board and never died** becomes a valid resurrection/recursion target — almost certainly unintended (in HS a countered card is removed from game, not recyclable). It also leaves `GraveyardMinion.diedOnTurn` undefined for a thing that didn't die.

**Recommendation.** Decide explicitly. Options:
- **(a)** Route a fizzled minion card to a **removed-from-game** sink (not the graveyard) — cleanest, matches HS, and avoids the `diedOnTurn` ambiguity.
- **(b)** Keep it in the graveyard but mark it non-resurrectable (e.g. a flag the resurrection selectors exclude). More machinery.
- **(c)** Accept it as a deliberate divergence (countered minions are recyclable). Cheapest, but should be a conscious call.

Recommend (a). Either way, pin `diedOnTurn`/death-metadata semantics for the fizzle case.

---

## C. Low severity / wording

**✅ ALL APPLIED (2026-06-19, session 16) — batch sweep.** Per-finding touch-points below.

### F11 — [WORDING / L] "Battlecry resolves synchronously inside the PlayCard pipeline" predates the #34 commit/resolve split   ✅ APPLIED
**Done:** §3 line 731 reworded to "the **card-resolution pipeline (`ResolveCardAction` ④** — not the now-commit-only `PlayCardAction`, per #34)." *(NB: this reword is what surfaced **F19** — see below.)*

**Where:** [:719](docs/superpowers/specs/2026-05-26-game-mechanics.md#L719). After #34, a minion card's summon + Battlecry run at **`ResolveCardAction` ④** (line 316), not at `PlayCardAction`. The sentence reads as if Battlecry runs in the (now commit-only) PlayCard handler. Reword to "inside the card-resolution pipeline (`ResolveCardAction` ④)".

### F12 — [WORDING / L] Directed-events "each paired with a public thin sibling" is off by one   ✅ APPLIED
**Done:** reworded so `MulliganStartedEvent` is called out as directed **per-player with NO public sibling**, and the **other five** directed events each pair with a sibling.

**Where:** [:362](docs/superpowers/specs/2026-05-26-game-mechanics.md#L362). Six directed events are listed but only five public siblings in the "respectively" mapping; `MulliganStartedEvent` has no sibling (it's directed *per-player*, each seeing their own — no public form needed). Reword so `MulliganStartedEvent` isn't implied to have a thin sibling.

### F13 — [WORDING / L] `turn.number` increment is not in the Turn Lifecycle steps   ✅ APPLIED
**Done:** increment added at Turn Lifecycle **step 6** (with a note that steps 1–5 deliberately read the *ending* turn's number — Freeze `frozenOnTurn`, Interturn `firstSpawnTurn`); Match Setup **step 8** now seeds `turn.number = 1` + `turn.activePlayerId`.

**Where:** Turn Lifecycle ([:1148-1164](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1148-L1164)). The 12 steps advance the active player (step 6) but never state where `turn.number` increments. It's referenced by Freeze (`frozenOnTurn == turn.number`) and the neutral-handover gate (`turn.number ≥ firstSpawnTurn`), so its advance point matters. Add it explicitly (presumably at step 6, "advance the active player").

### F14 — [WORDING / L] `EntityId[]` selector return type vs `string` ids everywhere else   ✅ APPLIED
**Done:** §2A ID-convention note now defines `EntityId` as a **documented alias for `string`** in v1 (same instance ids; selector/predicate boundary uses the alias for intent only); a typed wrapper is a flagged possible Epic-01 refinement, not v1 — "treat `EntityId` ≡ `string` everywhere."

**Where:** `ITargetSelector.Select` returns `EntityId[]` ([:732](docs/superpowers/specs/2026-05-26-game-mechanics.md#L732)), but every id field in §1/§2 is `string` (`minionId: string`, `m0`/`c1`/…). `EntityId` is never defined. Clarify whether it's an alias for `string` or a typed wrapper (the old Unity project used a typed-int wrapper — a deliberate choice worth restating). Pick one and use it consistently.

### F15 — [WORDING / L] Naming collision: `PlayerState.autoSkipAll` vs `RespondInterventionAction.response = SkipAll`   ✅ APPLIED
**Done:** kept both names (rename avoided to not touch many refs); added a one-line "distinct from…" disambiguator at **both** sites — `autoSkipAll` = turn-spanning standing flag over ALL windows; `SkipAll` = a single response covering only the CURRENT action's ③′ + ⑥′.

**Where:** [:81](docs/superpowers/specs/2026-05-26-game-mechanics.md#L81) and [:353](docs/superpowers/specs/2026-05-26-game-mechanics.md#L353). `autoSkipAll` (a standing, all-windows-this-turn preference) and `SkipAll` (a one-action response covering its ③′ + ⑥′) are different scopes but read alike — an implementer will conflate them. Consider renaming the response (`SkipRest`? `SkipThisAction`?) or adding a one-line "distinct from `autoSkipAll`" note at each site.

### F16 — [WORDING / L] "Savable" overclaims for a maxHealth-collapse mortal wound   ✅ APPLIED
**Done:** added the savability caveat at the `MinionMortallyWoundedEvent` row and §1 line 96 — only a **damage-domain** wound (`currentHealth ≤ 0 < maxHealth`) is **heal**-saveable; a **maxHealth-collapse** wound (`maxHealth ≤ 0`) is not (heal caps at `maxHealth ≤ 0`), only restoring `maxHealth` saves it. **"Savable" ≠ "heal-saveable."** *(This walk also triggered the F18 raise→retract + the §1 line-95 prose tightening — see the F18 record above.)*

**Where:** `currentHealth` / aura-loss path ([:95-96](docs/superpowers/specs/2026-05-26-game-mechanics.md#L95-L96)); ⑥ ([:1068](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1068)). A minion driven to `maxHealth ≤ 0` by aura/buff loss fires `MinionMortallyWoundedEvent` (the "savable" kind) — but `HealAction` caps at `maxHealth` (≤ 0), so **no heal can save it**; only restoring `maxHealth` (re-buff / the aura returning) can. The save window opens but the obvious save (heal) is a no-op. Worth a clarifying clause so the "savable" framing isn't misread as "heal-saveable."

### F17 — [WORDING / L] `EffectContext.SpellDamageBonus` casing is inconsistent with sibling fields   ✅ APPLIED
**Done:** `SpellDamageBonus` → `spellDamageBonus` (camelCase) at all 5 occurrences.

**Where:** [:217](docs/superpowers/specs/2026-05-26-game-mechanics.md#L217). In the pseudo-record, `SpellDamageBonus` is PascalCase while `sourceId`/`sourcePlayerId`/`targetId`/`state` are camelCase. Cosmetic, but pick one convention for the data-model blocks so the eventual C# doesn't inherit ad-hoc casing.

---

## C′. Findings raised during the walk (session 16)

### F18 — [non-finding] Health-buff removal and the #13 fall formula   ❌ RETRACTED (no change)

**Origin:** raised by me during the F16 walk; I claimed HS *kills* a damaged, health-buffed minion when the buff is removed (e.g. silence a 2/4-at-current-2 → dies), and that the spec's `min(currentHealth, newMaxHealth)` fall formula (which *survives*) was therefore a bug needing a "signed-Δ" fix.

**Why retracted:** **the user corrected the HS rule** — in Hearthstone, **removing a buff can never kill a minion.** Current Health is reduced only **down to the printed (base) stat**, and if it's *already* at/below that, it's **unchanged**. That is precisely what `min(currentHealth, newMaxHealth)` already computes (2/4-at-2 → silenced → `min(2,2)=2`, survives 2/2). So the spec was **already HS-correct**; my proposed signed-Δ fix would have *introduced* a divergence (killing minions HS never kills). **The formula `min(currentHealth + max(0, Δmax), newMax)` is unchanged.**

**What WAS applied:** a **prose tightening** of §1 line 95 only — replaced the misleading "*existing damage persists*" with the correct clamp-down/never-kills wording, and pinned the sole death path as `maxHealth ≤ 0` from a **negative** buff/aura (the collapse / aura-loss-death path). No formula or behavior change.

**Lesson (for the record):** verify HS rules before asserting them — the "damage-taken is preserved on buff removal" model is **not** how HS works for buff loss.

### F19 — [HOLE / M, NEW] Battlecry / minion-summon ordering is under-pinned vs. the per-action epoch model   ✅ RESOLVED + APPLIED (2026-06-21, the simplification) — superseded the 2026-06-20 sub-drain (reverted)

**✅ RESOLVED 2026-06-21 (session 18) — THE SIMPLIFICATION (use the existing summon flow).** After the user rejected the sub-drain (below) as unclean, we worked the redesign by tracing the placement-event question: it surfaced that the sub-drain's deferral of the *`MinionSummonedEvent` publish* (not just the registration) left the body placed with **no announcing event** — the Battlecry's `DealDamageAction` would carry `sourceId = Newcomer` before any event introduced Newcomer (violating §2 "every state change is announced" / §3 `IActionQueue`). The fix dissolved the whole mechanism:

- **Mechanism:** `ResolveCardAction` ④ does the **ordinary summon** — place body + register `OnSummon` triggers/Deathrattle/keyword-hooks + return `MinionSummonedEvent`, all inline at this ④ (epoch N), **identical to `SummonMinionAction` for a token** — and then **enqueues the Battlecry's effect actions** (ordinary queue citizens, each with its own ③′/⑥′). No sub-drain, no `FinalizeSummon`, no deferred epoch.
- **Why the ordering still works:** the Battlecry is enqueued at ④ **before** `MinionSummonedEvent` publishes at ⑤, so it sits ahead of any summon-reaction in the queue → resolves first → **(iii) other minions' "after you summon" reactions fire after the Battlecry** (HS-faithful, preserved). **(i) intervenable** — Battlecry actions keep their windows, unchanged. **On-time placement** — the summon event publishes before the Battlecry, so no dangling `sourceId`.
- **The trade — (ii) dropped as a documented divergence.** The minion's triggers are live during its own Battlecry (registered at summon, epoch N; Battlecry events at N+1, …), so it **does** react to its own Battlecry. Line 731's "Battlecry before triggers go live" is dropped. **HS verified 2026-06-21** (Advanced Rulebook): HS keeps a minion's triggered effects inactive until its Battlecry resolves, so this *is* a real divergence — accepted in exchange for engine uniformity (no "summoned-but-dormant" sub-state; triggers live by board zone like everything else). Blast radius is narrow: only a minion's own `ITrigger`s on its own Battlecry — own-summon-event exclusion (equal-epoch), keyword hooks (pulled), Deathrattle (snapshot) are all unaffected. Recorded with its reversible path (`FinalizeSummonAction` + DFS via generalized `EnqueueFront`) in spec **Unaddressed Features → "A minion triggers off its own Battlecry."**
- **Stress-tested:** summon-triggered removal (a summon-reaction that mortally wounds the new minion) does **not** fizzle the Battlecry — the Battlecry is enqueued ahead of the reaction and the minion lingers (death deferred to ⑦), so it gets its Battlecry off, then dies. Only ③′-cancel of `ResolveCardAction` itself, or a Battlecry effect's own target leaving, fizzles.
- **Touch-points reworked (grep-clean):** §2A `ResolveCardAction` ④, §3 line 731 (trigger-timing note), §1 `MinionOnBoard` TRIGGERS comment, §3 `IEventBus` note, + new Unaddressed-Features entry. The 2026-06-20 sub-drain edits are reverted.

**▶ SUPERSEDED — Resolution as applied 2026-06-20 (option (a), sub-drain + summon event deferred) — REVERTED 2026-06-21.** *Retained for the record only; replaced by the simplification above.* Both open calls were decided:
1. **Honor line 731 via the sub-drain (a)** — *not* dropping 731. `ResolveCardAction` ④ runs a minion card in three steps: **(1)** place the body (on board for the Battlecry's targeting/AoE; `MinionSummonedEvent` not yet published); **(2)** run the Battlecry, whose effect actions **sub-drain to completion as ordinary fully-windowed queue citizens** — each keeps its ③′/⑥′, so the Battlecry is **intervenable, never swallowed** (the user's explicit requirement); **(3)** after the sub-tree drains, publish `MinionSummonedEvent` **and** register `OnSummon` triggers/Deathrattle/keyword-hooks at a **fresh post-Battlecry epoch**. The deferred epoch is what makes the creation-epoch filter actually deliver "Battlecry before the minion's triggers go live" (the minion's `birthEpoch` > every Battlecry sub-action's `originEpoch` ⇒ no self-reaction to its own Battlecry; the co-located summon-event publish ⇒ no self-reaction to its own summon either). Rider: if an intervention during the Battlecry removes the minion, step 3 is skipped.
2. **`MinionSummonedEvent` publishes AFTER the Battlecry** (reversing my earlier "before/placement" lean). HS-faithful per **Razorfen Hunter (Battlecry: summon a Boar) + Knife Juggler** — HS summons the Boar first, *then* throws the knives, i.e. summon-triggers resolve after the Battlecry. The body is still *placed* before the Battlecry (board presence/AoE), only the *event publish* defers. Also mechanically cleaner: HS order falls out of the epoch stamp, no sub-queue-vs-main-queue assumption. *(HS claim flagged for verification per the F18 lesson; user accepted on the Razorfen evidence.)*

**Touch-points applied:** §2A `ResolveCardAction` ④ (the three-step order spelled out + the token-path contrast); §3 line 731 (mechanism, not just intent); §1 `MinionOnBoard` TRIGGERS comment (card-played defer vs token inline). The token-summon path (`SummonMinionAction`: no Battlecry ⇒ publish + `OnSummon` inline at the summon's ④) is now stated as the deliberate difference.

**Origin:** surfaced while applying F11 (Battlecry runs at `ResolveCardAction` ④).

**Where:** `ResolveCardAction` ④ "summon + Battlecry" ([:316](docs/superpowers/specs/2026-05-26-game-mechanics.md#L316)); Battlecry-before-triggers-live ([:731](docs/superpowers/specs/2026-05-26-game-mechanics.md#L731)); `OnSummon` "at summon / entering board" ([:111](docs/superpowers/specs/2026-05-26-game-mechanics.md#L111), [:800](docs/superpowers/specs/2026-05-26-game-mechanics.md#L800)); the per-action epoch filter `birthEpoch < originEpoch` ([:539-548](docs/superpowers/specs/2026-05-26-game-mechanics.md#L539-L548)); `ICardHandler.OnPlay`/`OnSummon` ([:894-899](docs/superpowers/specs/2026-05-26-game-mechanics.md#L894-L899)).

**The intended ordering** when a minion card is played (two pipelined actions, #34 split):
- **`PlayCardAction`** (commit, epoch N): pay/leave-hand/`CardPlayedEvent`, enqueue `ResolveCardAction`. ③′ here = *pre-cast* counter.
- **`ResolveCardAction`** ④: (1) place body + emit `MinionSummonedEvent`; (2) **`OnPlay` (Battlecry)** resolves; (3) **`OnSummon`** registers the minion's persistent `ITrigger`s/Deathrattle/keyword-hooks → live. Line 731 intent: a minion must **not** trigger off its own Battlecry (HS-faithful). ③′ at `ResolveCardAction` = the **Counterspell** window.

**The seam (why line 731 isn't actually delivered as written):**
1. Battlecry effects are **enqueued actions** (a "Battlecry: deal 3" must be a real `DealDamageAction` so it gets its own ④ precedence + ③′/⑥′ windows — the locked all-damage-one-channel rule). So they run at **later epochs** N+1, N+2, ….
2. The epoch model stamps a subscriber's `birthEpoch` with the epoch of the action whose ④ calls `Subscribe` (lines 542, 548). The natural reading "`OnSummon` at the summon's ④" stamps the minion's triggers `birthEpoch = N`.
3. Then `N < N+1` is **true** ⇒ the minion **would** react to its own Battlecry's sub-actions — violating line 731. (The epoch filter only cleanly stops self-reaction to the *same-epoch* `MinionSummonedEvent`.)
4. For line 731 to hold, `OnSummon` registration must be **deferred until the Battlecry sub-tree fully drains** and stamped with an epoch **past all of it**. Two things the spec never states: **(a)** that the Battlecry **sub-drains to completion** inside `ResolveCardAction` ④ ("synchronous-in-pipeline" gestures at it; a plain FIFO "register last" action doesn't guarantee it beats Battlecry *grandchildren*); **(b)** what epoch the deferred `OnSummon` gets.
5. Unstated sub-question: does `MinionSummonedEvent` fire at **placement (before Battlecry)** — so other minions' "after you summon" reactions (Knife-Juggler-style) precede the Battlecry — or after? It's observable and currently undefined.

**My recommendation (presented, awaiting the user):** option **(a)** — `ResolveCardAction` ④ = place body + emit `MinionSummonedEvent` → **Battlecry sub-drains to completion** (the response-resolution idiom, §4 line 1180) → **then** `OnSummon` registers triggers with a **fresh post-Battlecry epoch** (so `birthEpoch >` every Battlecry epoch ⇒ no self-trigger; Battlecry stays a fully-pipelined set of actions with windows; HS-faithful). Alternative = accept the minion *can* react to its own Battlecry and **drop line 731** (simpler, diverges from HS/intent).

**Two open calls — both RESOLVED 2026-06-20 (see the ▶ Resolution block at the top of this section):**
1. ✅ Confirmed **(a)** — synchronous Battlecry sub-drain (fully windowed/intervenable) + deferred post-Battlecry `OnSummon` epoch. Line 731 honored, *not* dropped.
2. ✅ `MinionSummonedEvent` publishes **after** the Battlecry (reversed my "before" lean; Razorfen Hunter + Knife Juggler evidence). Body placed before, event published after.

---

## D. Design suggestions & deliberate consequences to confirm

These are not defects — they're calls worth confirming on the record before they're baked into code.

- **D1 — Active player cannot save their own hero/minions on their own turn (by design). ✅ APPLIED 2026-06-21.** Under Model B′ (responder = always non-active), if the active player crosses to lethal on their *own* turn (own-board AoE clipping their hero, self-damage, fatigue), no ③′/⑥′ window opens for them, and their own inscriptions are off-turn-only. So self-inflicted lethal on your turn is **unsaveable by reactive cards** — HS-consistent (Ice Block is a Secret = off-turn). Added as a one-line **Corollary (D1)** to §3 "Who responds" (`:786`), the user confirming it's in line with expectations.

- **D2 — `SpellCastEvent` timing. ✅ RESOLVED 2026-06-21 — original premise overturned, event MOVED to resolution.** The finding originally read "`SpellCastEvent` fires at commitment even when countered → cast-triggers fire for a countered spell (HS Counterspell)." The user challenged the **consistency**: `MinionSummonedEvent` (the minion's resolution event) fires at `ResolveCardAction` ④, so why does `SpellCastEvent` sit at `PlayCardAction` ④ (commit)? Investigation + **HS verification (2026-06-21, Advanced Rulebook + Counterspell wiki)** flipped the answer: **current HS suppresses *all* cast triggers on a countered spell** (Patch 11.2.0, June 2018, unified "whenever you cast" and "after you cast" — Counterspell fires before both); the original premise matched only *pre-2018* HS. Fix: **`SpellCastEvent` moved from `PlayCardAction` ④ (commit) → `ResolveCardAction` ④ (resolution)** — the spell-side analog of `MinionSummonedEvent`. A **countered** spell now fires `CardPlayedEvent` (played, mana spent, → `GraveyardSpell`) but **no** `SpellCastEvent`, so cast-synergy doesn't trigger — current-HS-correct and symmetric with minions. **Pre vs post nuance (user's follow-up):** the pre-resolution moment already exists as the ③′ `ActionDeclaredEvent` (per §3 pillar 2 it's the *interception* phase, e.g. Counterspell — and it fires *before* the counter, so it's not where counter-respecting synergy belongs); cast-synergy is a post-reaction → `SpellCastEvent`. No new event, no Unaddressed deferral needed (the two-phase ③′-declaration / ⑤-resolution model already spans pre and post). **Touch-points:** §2A `PlayCardAction` ④ (drop), `ResolveCardAction` ④ (add + countered note), §2B `SpellCastEvent` row, the "Fireball countered" trace example (#2), `:786` D1 corollary.

- **D3 — Reborn model simplification. ✅ No edit (subsumed by F5).** Initializing `rebornAvailable = true` for all minions (F5) was both the bug fix and this cleanup.

- **D4 — Press cap of 3 is a balance lever. ✅ No edit.** `MaxInterventionsPerTurn = 3` is a deliberate tempo ceiling for playtest tuning (with `MaxInscriptions`/`HandCap`), not a correctness issue; the #33 cap-interaction note already gestures at it.

- **D3 — Reborn model simplification.** See F5's recommendation — initializing `rebornAvailable = true` for all minions (charge = "unspent") is both the bug fix and a conceptual cleanup.

- **D4 — Press cap of 3 is a balance lever, not a correctness issue.** Once F3 is reconciled, note that `MaxInterventionsPerTurn = 3` means a defender can answer at most 3 of the active player's actions per turn regardless of cards/mana held. That's a deliberate tempo ceiling; just worth surfacing for playtest tuning alongside `MaxInscriptions`/`HandCap` (the #33 cap-interaction note already gestures at this).

---

## E. Checked and consistent (coverage notes)

So the user knows what was verified, not just what failed:

- **Turn Lifecycle step-number cross-references** (§1 → §4): `frozenOnTurn`→step 3, `interventionsUsed`→step 6, `summoningSick`/`attacksUsedThisTurn`→step 7, `cardsPlayedThisTurn`/artifact `usesThisTurn`→step 8 — **all match** §4's renumbered (post-session-12) lifecycle. The Interturn insertion renumbering held.
- **Rejection codes** used in §4 ③ (`NotActivePlayer`, `NotEnoughMana`, `InvalidTarget`, `TargetStealthed`, `MustTargetTaunt`, `AttackerNotAlive`, `AttackerNotControlled`, `AttackerCannotAttack`, `RushCannotTargetHero`, `AttackerFrozen`, `AttackerExhausted`, `BoardFull`, `ArtifactSlotsFull`, `ArtifactNotActive`, `InscriptionSlotsFull`, `SigilAlreadyInscribed`) all appear in the §2C enum — the only gap is F8 (the two Intervention codes).
- **Lane-based friendly/enemy** predicates are stated identically in `ITriggerCondition` (lines 664–665) and `ITargetSelector` (line 736) and `IAura` scope (line 875) — consistent.
- **Death cadence** (⑦/⑧ as settle stages, pending-death lingering, mortally-wounded vs destroy-mark) is consistent across §1, §3, §4, and Unaddressed Features.
- **Directed/public event split**, `eventId`, `originEpoch`/`birthEpoch` epoch-filter, and the bus-only `ActionDeclaredEvent`/`AttackDeclaredEvent` exclusions are internally consistent.
- **Trace examples** (1)–(3) correctly use the two-stage `RespondInterventionAction`→`SubmitInterventionAction` flow (confirming F2's line 323 is the stale one).

---

## Suggested triage order

1. **F1** (mana seeding) — correctness, every match, one-line fix.
2. **F2, F3, F8** (intervention-model drift trio) — all are session-13-rework cleanup; batch them.
3. **F5** (Reborn) — small logic fix + cleanup.
4. **F4** (selector `Filter`/predicate) — needs a design call; lands in the Epic-08 `ITargetSelector` ticket.
5. **F6, F7** (match-setup ordering / mulligan-sigil) — setup robustness.
6. **F9, F10** (choice types / fizzled-minion graveyard) — F9 rides the Epic-01 ticket; F10 is a design call.
7. **F11–F17** (wording) — sweep in one pass.
8. **D1–D4** — confirm-on-the-record, no edits required unless you want the clarifying notes.
