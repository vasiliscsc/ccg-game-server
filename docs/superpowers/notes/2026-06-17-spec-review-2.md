# Spec Review #2 — post-fix-pass audit

**Date:** 2026-06-17
**Reviewer:** read-only full audit of `docs/superpowers/specs/2026-05-26-game-mechanics.md` (1250 lines, §1–§4 + Unaddressed Features).
**Purpose:** catch inconsistencies, logical bugs, stale text, and wording problems **before implementation** — cheaper to fix the spec than thousands of lines of code.
**Relationship to the prior review:** distinct from `notes/2026-06-12-spec-review-findings.md` (the 33-finding doc, all since applied in the fix pass). This is a fresh read of the *current* spec. Findings here are numbered **F1…** to avoid collision with the old `#1…#33`.

**Status legend:** each finding is `[CATEGORY / Severity]`.
Categories: **CONTRADICTION** (two parts of the spec disagree) · **BUG** (logic produces a wrong/unintended outcome) · **HOLE** (underspecified / missing rule) · **DRIFT** (stale text from a superseded decision) · **WORDING** (clarity only) · **DESIGN** (a question or improvement, not a defect).

**Summary:** 3 high, 7 medium, 7 low/wording, plus a design-suggestions section. None are show-stoppers; the headline (F1) is a real gameplay bug that would surface in the very first match.

---

## ▶ Walk progress (as of 2026-06-17, end of session 15)

**Applied to spec:** F1 ✅, F2 ✅, F3 ✅, F4 ✅ (option B), F5 ✅ (option a), F6 ✅, F7 ✅, F8 ✅.
**⏳ RESUME HERE — F9 + F10 presented, awaiting the user's decision** (user stopped before deciding):
- **F9** — proposed shape to lock (or defer `ChoiceOption` fields to Epic-01): `enum ChoiceType { Discover, Target, Modal }`; `record ChoiceOption(string OptionId, string? EntityId = null, string? DefinitionKey = null, string? Label = null)`; **drop `choiceId`** from `SubmitChoiceAction` → `{ playerId, selectedOptionId }` (single `PendingChoice` at a time ⇒ unambiguous). Decision pending: lock-now vs defer-fields.
- **F10** — fork pending: **(a)** fizzled minion → removed-from-game (recommended, HS-faithful, avoids resurrection + undefined `diedOnTurn`); **(b)** graveyard + non-resurrectable flag; **(c)** accept recyclable.

**Not yet walked:** F11–F17 (wording/clarity sweep — likely batch-apply), D1–D4 (design notes to confirm on the record).

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

### F9 — [HOLE / M] `ChoiceType` values, `ChoiceOption` shape, and `SubmitChoiceAction.choiceId` are undefined   ⏳ AWAITING DECISION (resume here)

**Where:** `PendingChoice` ([:195](docs/superpowers/specs/2026-05-26-game-mechanics.md#L195)); `SubmitChoiceAction` ([:322](docs/superpowers/specs/2026-05-26-game-mechanics.md#L322)); `StartChoiceAction` ([:351](docs/superpowers/specs/2026-05-26-game-mechanics.md#L351)).

**Problem.** (Already flagged in the session-14 Discover Q&A for the Epic-01 data-model ticket; recording here so the spec audit is complete.) The spec references `choiceType: ChoiceType` and `options: ChoiceOption[]` but never enumerates `ChoiceType`'s values (Discover? mid-effect targeting? mulligan rides its own action — so probably just `Discover` + `Target`?) nor defines `ChoiceOption`'s fields. Additionally, **`SubmitChoiceAction` carries `choiceId`** but nothing else defines a choice id (`PendingChoice`, `ChoicePromptEvent`, `ChoiceStartedEvent` have none); with at most one pending choice at a time it's likely vestigial, and `selectedOptionId` implies `ChoiceOption` has an `id`.

**Recommendation.** Nail down in the Epic-01 data-model ticket: the `ChoiceType` enum members, the `ChoiceOption` record (`{ id, … }` — at minimum the option's payload/target ref), and whether `SubmitChoiceAction.choiceId` is needed at all (drop it if a single in-flight choice makes it redundant; the `selectedOptionId` references `ChoiceOption.id`).

---

### F10 — [DESIGN / M] A fizzled (countered) minion card lands in the graveyard as a `GraveyardMinion` — making it a resurrection/recursion target   ⏳ AWAITING DECISION (resume here)

**Where:** `ResolveCardAction` fizzle branch ([:316](docs/superpowers/specs/2026-05-26-game-mechanics.md#L316)).

**Problem.** *"A **minion** card that fizzles goes to the owner's graveyard **here** (built from its `definitionKey` + base stats — it never reached the board)."* That entry is a `GraveyardMinion`, which is exactly what *"resurrect a friendly minion that died"* / *"add the last dead minion's card to hand"* effects read. So a **countered minion that never hit the board and never died** becomes a valid resurrection/recursion target — almost certainly unintended (in HS a countered card is removed from game, not recyclable). It also leaves `GraveyardMinion.diedOnTurn` undefined for a thing that didn't die.

**Recommendation.** Decide explicitly. Options:
- **(a)** Route a fizzled minion card to a **removed-from-game** sink (not the graveyard) — cleanest, matches HS, and avoids the `diedOnTurn` ambiguity.
- **(b)** Keep it in the graveyard but mark it non-resurrectable (e.g. a flag the resurrection selectors exclude). More machinery.
- **(c)** Accept it as a deliberate divergence (countered minions are recyclable). Cheapest, but should be a conscious call.

Recommend (a). Either way, pin `diedOnTurn`/death-metadata semantics for the fizzle case.

---

## C. Low severity / wording

### F11 — [WORDING / L] "Battlecry resolves synchronously inside the PlayCard pipeline" predates the #34 commit/resolve split

**Where:** [:719](docs/superpowers/specs/2026-05-26-game-mechanics.md#L719). After #34, a minion card's summon + Battlecry run at **`ResolveCardAction` ④** (line 316), not at `PlayCardAction`. The sentence reads as if Battlecry runs in the (now commit-only) PlayCard handler. Reword to "inside the card-resolution pipeline (`ResolveCardAction` ④)".

### F12 — [WORDING / L] Directed-events "each paired with a public thin sibling" is off by one

**Where:** [:362](docs/superpowers/specs/2026-05-26-game-mechanics.md#L362). Six directed events are listed but only five public siblings in the "respectively" mapping; `MulliganStartedEvent` has no sibling (it's directed *per-player*, each seeing their own — no public form needed). Reword so `MulliganStartedEvent` isn't implied to have a thin sibling.

### F13 — [WORDING / L] `turn.number` increment is not in the Turn Lifecycle steps

**Where:** Turn Lifecycle ([:1148-1164](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1148-L1164)). The 12 steps advance the active player (step 6) but never state where `turn.number` increments. It's referenced by Freeze (`frozenOnTurn == turn.number`) and the neutral-handover gate (`turn.number ≥ firstSpawnTurn`), so its advance point matters. Add it explicitly (presumably at step 6, "advance the active player").

### F14 — [WORDING / L] `EntityId[]` selector return type vs `string` ids everywhere else

**Where:** `ITargetSelector.Select` returns `EntityId[]` ([:732](docs/superpowers/specs/2026-05-26-game-mechanics.md#L732)), but every id field in §1/§2 is `string` (`minionId: string`, `m0`/`c1`/…). `EntityId` is never defined. Clarify whether it's an alias for `string` or a typed wrapper (the old Unity project used a typed-int wrapper — a deliberate choice worth restating). Pick one and use it consistently.

### F15 — [WORDING / L] Naming collision: `PlayerState.autoSkipAll` vs `RespondInterventionAction.response = SkipAll`

**Where:** [:81](docs/superpowers/specs/2026-05-26-game-mechanics.md#L81) and [:353](docs/superpowers/specs/2026-05-26-game-mechanics.md#L353). `autoSkipAll` (a standing, all-windows-this-turn preference) and `SkipAll` (a one-action response covering its ③′ + ⑥′) are different scopes but read alike — an implementer will conflate them. Consider renaming the response (`SkipRest`? `SkipThisAction`?) or adding a one-line "distinct from `autoSkipAll`" note at each site.

### F16 — [WORDING / L] "Savable" overclaims for a maxHealth-collapse mortal wound

**Where:** `currentHealth` / aura-loss path ([:95-96](docs/superpowers/specs/2026-05-26-game-mechanics.md#L95-L96)); ⑥ ([:1068](docs/superpowers/specs/2026-05-26-game-mechanics.md#L1068)). A minion driven to `maxHealth ≤ 0` by aura/buff loss fires `MinionMortallyWoundedEvent` (the "savable" kind) — but `HealAction` caps at `maxHealth` (≤ 0), so **no heal can save it**; only restoring `maxHealth` (re-buff / the aura returning) can. The save window opens but the obvious save (heal) is a no-op. Worth a clarifying clause so the "savable" framing isn't misread as "heal-saveable."

### F17 — [WORDING / L] `EffectContext.SpellDamageBonus` casing is inconsistent with sibling fields

**Where:** [:217](docs/superpowers/specs/2026-05-26-game-mechanics.md#L217). In the pseudo-record, `SpellDamageBonus` is PascalCase while `sourceId`/`sourcePlayerId`/`targetId`/`state` are camelCase. Cosmetic, but pick one convention for the data-model blocks so the eventual C# doesn't inherit ad-hoc casing.

---

## D. Design suggestions & deliberate consequences to confirm

These are not defects — they're calls worth confirming on the record before they're baked into code.

- **D1 — Active player cannot save their own hero/minions on their own turn (by design).** Under Model B′ (responder = always non-active), if the active player crosses to lethal on their *own* turn (own-board AoE clipping their hero, self-damage, fatigue), no ③′/⑥′ window opens for them, and their own inscriptions are off-turn-only. So self-inflicted lethal on your turn is **unsaveable by reactive cards**. This is HS-consistent (Ice Block is a Secret = off-turn) and follows from the model — flagging only so it's a conscious acceptance, not a surprise.

- **D2 — `SpellCastEvent` fires at commitment even when the spell is then countered.** A spell's `CardPlayedEvent` + `SpellCastEvent` fire at `PlayCardAction` ④ (commitment); if `ResolveCardAction` is countered, the effects never run but those cast events already fired — so "whenever you cast a spell" triggers fire for a countered spell. This matches HS Counterspell semantics (the spell *was* cast, then countered). Confirm it's intended; if so, a one-line note at `CardPlayedEvent`/`SpellCastEvent` would pre-empt the "why did my cast-trigger fire on a countered spell" question.

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
