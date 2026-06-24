# Spec Review #3 — game-mechanics.md

**Date:** 2026-06-21
**Reviewer pass:** third full read-only review of `specs/2026-05-26-game-mechanics.md`, after spec-review #1 (session-9, 33 findings, all applied) and spec-review #2 (F1–F19 + D1–D4, all applied/resolved). Read end-to-end with extra scrutiny on the freshly-landed **F19** (Battlecry/summon simplification) and **D2** (`SpellCastEvent` → resolution) seams, since those are newest.
**Status:** findings raised, **none applied yet** — this doc is the input queue for a walk (one-by-one, present-tradeoff-+-recommend), exactly like reviews #1/#2.
**Method:** logical-bug / inconsistency / hole hunt, plus a prior-art **game-design recommendations** chapter at the end (web-assisted, per the request).

> **Calibration:** two prior thorough reviews + a fix pass have already drained the hard contradictions, so this pass is mostly **subtle holes + freshly-introduced seams + under-specifications**, not headline bugs. Findings are `G`-tagged (`G1…`) to avoid collision with the `#`-series (review #1) and `F`/`D`-series (review #2). Severity H/M/L; category CONTRADICTION / BUG / HOLE / DRIFT / LIMITATION.

---

## ▶ Walk progress

**Session 19 (2026-06-22) — walking one-by-one, confirm-each.**
- **G1 ✅ APPLIED** — Combo predicate stated as `cardsPlayedThisTurn ≥ 2` (not `> 0`). Off-by-one note added at §1:79 (`cardsPlayedThisTurn`) + the predicate stated at §3:692 (`ITriggerCondition`, Combo). `ComboTriggeredEvent` needs no change (the predicate lives in the condition, not the event). The heavier "snapshot cards-played-before-this into `CardPlayContext`" alternative was rejected — the threshold encodes it for free.
- **G2 ✅ APPLIED** (clean fold, user's pick) — `SpellCastEvent` now carries the **full `card`** (mirrors `CardPlayedEvent`), threaded from commit via a `card` snapshot on `ResolveCardAction`. `ResolveCardAction`'s former separate `cardId`/`definitionKey` fields were **dropped** (subsumed by `card.id`/`card.definitionKey`) — no redundant data. Touch-points applied: §2A:318 (enqueue signature), §2A:319 (`ResolveCardAction` row + `SpellCastEvent` emit note), §2B:462 (`SpellCastEvent` row). `GraveyardSpell` deliberately **unchanged** (the whole point of the clean fix: cost rides the event, not the graveyard). Traces at :967/:968/:971/:972 already render `"card"` compactly, so no trace edits.
- **G3 ✅ NO-OP (reviewer miss)** — the finding's premise is false against the live spec: the `MinionOnBoard` block **already** enumerates all four tribe fields at §1:101–104 (`intrinsicTribes`/`grantedTribes`/`auraTribes`/`effectiveTribes`), right after the keyword trio. Verified there's only one `MinionOnBoard` block and no edits landed between the review and the walk. No change made.
- **G4 ✅ APPLIED — but the finding's recommended fix was CORRECTED.** The finding said "reject `BoardFull` → **fizzle** (full-board-summon pattern)." The "reject is wrong (effect-issued, no submitter)" half is right, but **fizzle is also wrong**: a user clarifying question ("fizzle the effect or the minion?") prompted HS verification — a take-control onto a **full** board **destroys the target** (both targeted *and* random/deathrattle), and its Deathrattle fires **as the *original* owner's** ([Take control, hearthstone.wiki.gg](https://hearthstone.wiki.gg/wiki/Take_control)). So the correct precedent is the **`ReturnToHandAction` full-hand-bounce pattern (#21)** — enqueue `DestroyMinionAction` (the minion still has its original `ownerId` because the full-board check precedes the re-home, so original-owner graveyard + Deathrattle fall out free), **not** the full-board-summon fizzle (which never creates an entity; TakeControl moves a *pre-existing* one). Applied at §2A:351. `BoardFull` rejection code is **not orphaned** (still used by the player minion-play ③ check, §4:1055 + enum §2C:488). **Lesson reaffirmed (F18/D2):** verify HS against sources, not the finding's framing.
- **G5 ✅ APPLIED (delete both)** — `InterventionPlayedEvent` + `InterventionWindowClosedEvent` removed (§2B). Gated on a **full invocation walkthrough** (user-requested) that proved the spec is bulletproof without them: "played in response" = `CardPlayedEvent`(+`SpellCastEvent`)/`SigilRevealedEvent`; window-close is inferable in every close path (skip/skipall/timeout/autoskip/back-out/play/⑥′) from the next record + `phase → InProgress`. Lone residual = a minor non-responder-client render ergonomic, outweighed by the unconditional-window flood (a paired close doubles the noisiest tier) and the thin-public-event ethos; if ever needed, a generic phase-change signal beats two bespoke events. Borrow-list note's old "stays public" line annotated SUPERSEDED.
- **G6 ✅ APPLIED — but DISSOLVED, not "stored" (user's leaner call).** Rather than pick a storage slot, the user dropped the **`Enrage` keyword + `EnrageStateChangedEvent`** entirely (both deleted, §3:581 / §2B:429). Enrage is now authored as a **self-conditional `IAura`** — a `Calculate` reading its own host (`SourceEntityId`) and returning a self-targeting, condition-gated `AuraEffect`; the bonus rides the ordinary `auraAttackBonus/auraHealthBonus/auraKeywords` channel (recomputed each ⑥, Silence-safe), so no dedicated storage/toggle. Verified machinery-free against `IAura` (scope "is entirely its Calculate"; `AuraEffect` already carries attack/health/tribe/keyword grants). Added an `IAura` authoring note (§3, before `ICardHandler`) documenting the pattern **+ a convergence caveat** (per the user's "1+2"): single-pass recalc means a self-**referential** aura whose output feeds its own predicate (e.g. "while Attack ≤ 3, +2 Attack") is non-convergent/undefined — Enrage is safe because its predicate is on *health*, output on *attack*, and the damage gap is #13-invariant. **A user challenge corrected my Example B mid-walk** (an external health aura keeps the damage gap, so Enrage fires without any snapshot rule — the snapshot idea was decoupled as a separate general-`IAura` matter, NOT bundled here).
- **G7 ✅ APPLIED** — Match Setup step 1 (§4:1149) now pins the setup-equipped starter's triggers/auras to **`birthEpoch = 0`** (match-start epoch, same as step-7 opening-hand sigils; not the normal ④ stamp, since setup is outside the turn pipeline) → live from turn 1, no reaction to setup's own events. Genuinely moot for the v1 active-only starter, but pins the rule for a future passive/hero artifact. No plan impact beyond a one-line note on the `EquipArtifactAction` setup path.
- **G8 ✅ RESOLVED (session 20, 2026-06-24) — via DELETING the pre-halt death rule, not the carve-out.** The session-19 ruling proposed half (b) as a *carve-out* (`pendingIntervention != null` ⇒ a response-nested `PendingChoice` inherits deferred-death, doesn't pre-settle). The user's mandated Discover/Modal review **generalized the problem past the carve-out**: the pre-halt death rule (force-settle ⑦ before a type-A halt) was the **only** place in the engine that violated the cascade-settle cadence, and it gave *choices* a privileged board no other action gets — making ③′ ≠ ⑥′ for the same responder, and a Discover-summon luckier than a battlecry-summon (freed slot). It is **feel-only** (hide "noise"), it **contradicts** the still-on-board targeting rule (§4 ③ :1044: a `Dead…`/`All…` selector may target a *still-on-board* mortally-wounded minion), and it **silently breaks** an obviously-intended card class — *"When a friendly minion is Mortally Wounded, discover an effect"* (a save-via-Discover = R1 through a choice), at every link of a chained Discover→Target. So instead of carve-out (b), we **removed the pre-halt rule entirely**: ⑦/⑧ now run **only** at queue-empty (⑨), uniformly — no halt or window force-settles early. Half (a) (a response may open a `PendingChoice`) stands as a one-liner and dissolves into "it just works" (the deeper inconsistency is gone). Applied: spec §4 PendingChoice Interruption (rule → uniform-cadence statement + the G8(a) note + chained-choice note), §4 ⑥′ step 2 (dropped the now-defunct contrast), §4 "Response resolution" (justification re-cites the real queue-empty ⑦, not the removed early settle). *(Immediate-resolution rule survives on its own merits — a FIFO-appended save still loses its target to the normal queue-empty ⑦.)*
- DR1–DR6: not yet walked.

Suggested order at the bottom. Headliners: **G1** (Combo self-counting — DONE) and **G2** (`SpellCastEvent` can't read the cast card's cost) are the two with real gameplay teeth.

---

## A. Findings

### G1 — [BUG / M] Combo counts the card that triggers it (off-by-one from the commit/resolve split) — ✅ APPLIED (session 19, 2026-06-22)

`cardsPlayedThisTurn` is incremented at **`PlayCardAction` ④ (commit)** ([§1:79](../specs/2026-05-26-game-mechanics.md#L79), [§2A:318](../specs/2026-05-26-game-mechanics.md#L318)). A **Combo** battlecry is evaluated at **`ResolveCardAction` ④ (resolution)** — strictly *after* the commit. So by the time the Combo condition runs, `cardsPlayedThisTurn` **already counts the Combo card itself** (≥ 1 even if it's the first card of the turn).

If anyone implements the Combo predicate as the obvious `cardsPlayedThisTurn > 0`, **Combo is always active** — a real bug. HS-correct Combo is "you played *another* card first," i.e. the threshold must be **≥ 2** (this card + ≥1 prior), or the predicate must read `cardsPlayedThisTurn − 1`. The spec **never states the Combo predicate**, so the off-by-one is latent and undefended.

- **Fix:** spell out the Combo condition explicitly (e.g. `Subject` n/a — it's a turn-context condition; define it as `cardsPlayedThisTurn ≥ 2`), and add a one-line note at [§1:79](../specs/2026-05-26-game-mechanics.md#L79) that the current card is already counted by resolution time. Alternative (heavier): snapshot "cards played before this one" into `CardPlayContext` at commit. The threshold fix is the lean one.
- **Touch-points:** §1 `cardsPlayedThisTurn`, §3 `ITriggerCondition` (Combo is listed as a turn-context condition at [:692](../specs/2026-05-26-game-mechanics.md#L692) but never given a predicate), `ComboTriggeredEvent`.

### G2 — [HOLE / M (H if many cost-gated cast cards)] `SpellCastEvent` can't evaluate the cast spell's cost/stats — and `CardPlayedEvent` can — ✅ APPLIED (session 19, 2026-06-22)

`SpellCastEvent` carries only `{ playerId, cardId, targetId? }` ([§2B:462](../specs/2026-05-26-game-mechanics.md#L462)), whereas its sibling `CardPlayedEvent` carries the full `{ playerId, card, targetId? }` ([§2B:391](../specs/2026-05-26-game-mechanics.md#L391)). A `Subject(p)` trigger condition resolves `p` against the **cast card** ([§3:690](../specs/2026-05-26-game-mechanics.md#L690)), but by the time `SpellCastEvent` publishes the card has **left hand and become a `GraveyardSpell`** (which stores `definitionKey` only — no `modifiers`, no `effectiveCost`; [§1:182](../specs/2026-05-26-game-mechanics.md#L182)). So a cost/stat-gated cast trigger — `Subject(CostAtLeast(5))`, "whenever you cast a **expensive** spell" — has nothing to read its `effectiveCost` from, and silently evaluates against base cost or fails.

**D2 sharpened this into a genuine expressiveness gap:** the two events now split the needed information:
- `CardPlayedEvent` has the full card (cost readable) but fires at **commit**, so it fires even for a **countered** spell (post-D2 we *want* cast-synergy to skip countered spells).
- `SpellCastEvent` fires at **resolution** (counter-respecting) but **lacks the card**.

So *"whenever you cast a 5-cost-or-more spell that actually resolves"* — a bread-and-butter HS archetype — is **currently inexpressible correctly**.

- **Fix (lean):** give `SpellCastEvent` the full `card` (snapshot at commit, carried through to the resolution emit) like `CardPlayedEvent` does — then it has both the cost *and* the counter-respecting timing. One field on one event.
- **Touch-points:** §2B `SpellCastEvent`, §3 `ITriggerCondition` `Subject`/cost predicates, §1 `GraveyardSpell`.

### G3 — [DRIFT / L] `MinionOnBoard` field block omits the tribe fields it's said to own — ✅ NO-OP / reviewer miss (session 19, 2026-06-22): fields already present at §1:101–104

The Tribes section states the three-way split (`intrinsicTribes` / `grantedTribes` / `auraTribes`, plus derived `effectiveTribes`) "exists only on `MinionOnBoard`" ([§1:247](../specs/2026-05-26-game-mechanics.md#L247)), and `TransformMinionAction`/`SilenceMinionAction` mutate them by name. But the `MinionOnBoard` field block ([§1:84–112](../specs/2026-05-26-game-mechanics.md#L84)) lists the *keyword* trio and never enumerates the tribe fields. Pure doc-completeness drift — a reader building the struct from the block alone would miss them.

- **Fix:** add the four tribe fields to the `MinionOnBoard` block (mirroring the keyword trio), or add a one-line "+ the tribe trio, see Tribes."

### G4 — [DRIFT / L → behavioral fix] `TakeControlAction` says "reject `BoardFull`" but, as an effect-issued action, should **fizzle** — ✅ APPLIED w/ CORRECTION (session 19, 2026-06-22): HS *destroys* the target (→ #21 pattern), not fizzle

`TakeControlAction` is always effect-issued (a card's "take control of…" effect), yet its handler says "**reject** `BoardFull` if the destination is full" ([§2A:351](../specs/2026-05-26-game-mechanics.md#L351)). But "reject" produces an `ActionRejection` returned to a submitting player (§2C) — there is no player submitting an effect-issued action. The established **full-board-summon pattern** is that effect-issued actions onto a full board **fizzle** at ④ re-validation (e.g. `SpawnNeutralMinionAction` [:359](../specs/2026-05-26-game-mechanics.md#L359), `EquipArtifactAction`'s effect path [:345](../specs/2026-05-26-game-mechanics.md#L345)). `TakeControlAction` should follow it.

- **Fix:** change "reject `BoardFull`" → "**fizzle** at ④ re-validation (full-board-summon pattern)" for `TakeControlAction`.

### G5 — [DRIFT / L] Orphaned intervention events: `InterventionPlayedEvent` (and maybe `InterventionWindowClosedEvent`) — ✅ APPLIED / DELETED BOTH (session 19, 2026-06-22; gated on the invocation walkthrough)

`InterventionPlayedEvent { respondingPlayerId, cardId }` ([§2B:478](../specs/2026-05-26-game-mechanics.md#L478)) is defined, but **no action or §4 flow emits it** — `SubmitInterventionAction` ([§2A:333](../specs/2026-05-26-game-mechanics.md#L333)) describes resolving the card without it, and the worked traces emit `CardPlayedEvent`/`SpellCastEvent` for an invocation instead ([:971](../specs/2026-05-26-game-mechanics.md#L971), [:982](../specs/2026-05-26-game-mechanics.md#L982)) — never `InterventionPlayedEvent`. Likewise `InterventionWindowClosedEvent` ([:477](../specs/2026-05-26-game-mechanics.md#L477)) has no stated emit point in §4's `PendingIntervention Interruption`.

- **Fix:** either wire these into §4's intervention flow (where exactly each fires) **or** delete them if `CardPlayedEvent` + `SigilRevealedEvent` + `InterventionWindowOpenedEvent` already cover the wire. Right now they read as vestigial.

### G6 — [HOLE / L] Enrage's conditional-buff storage is under-specified — ✅ APPLIED / DISSOLVED (session 19, 2026-06-22): `Enrage` keyword + `EnrageStateChangedEvent` DROPPED → self-conditional `IAura`

Enrage is listed as a keyword "recomputed in the stage-⑥ aura/stat pass from `currentHealth < maxHealth`" with an `EnrageStateChangedEvent` toggle ([§3:583](../specs/2026-05-26-game-mechanics.md#L583), [§2B:429](../specs/2026-05-26-game-mechanics.md#L429)), but **where the enrage attack/stat bonus lives is never stated** — it's neither an `enchantment` (those are permanent) nor an `auraBonus` (those come from `IAura.Calculate`), yet it's a conditional stat delta recomputed each ⑥. A first Enrage card can't be built without deciding this.

- **Fix (likely):** route Enrage through the same `aura*Bonus` recompute channel (it *is* a continuous, board-state-conditional stat effect — i.e. model it as an `IAura` keyed on the host's own `currentHealth`), or add an explicit `enrageBonus` recompute slot. Worth one sentence even if the full design defers to the first Enrage card.

### G7 — [HOLE / L] Trigger-registration epoch for **setup-equipped** artifacts is unspecified — ✅ APPLIED (session 19, 2026-06-22): `birthEpoch = 0` pinned at Match Setup step 1

Match Setup step 1 equips each player's starter artifact via `EquipArtifactAction` ([§4:1149](../specs/2026-05-26-game-mechanics.md#L1149)), which registers the definition's triggers/auras with a stamped `birthEpoch` ([§2A:345](../specs/2026-05-26-game-mechanics.md#L345)). But setup runs partly **outside the pipeline** (opening-hand sigils register at `birthEpoch = 0` explicitly, step 7 [:1155](../specs/2026-05-26-game-mechanics.md#L1155)). It's unstated whether the setup equip runs through ④ (getting some epoch) or directly. **Moot for v1** (the shared starter is active-only — "pay mana, draw" — with no triggers/auras to register at a meaningful epoch), but a future **passive** starter/hero artifact would need the rule.

- **Fix:** one line at Match Setup step 1: setup-equipped artifact triggers register at `birthEpoch = 0` (same as opening-hand sigils), so they're live from turn 1 and don't react to setup's own events.

### G8 — [HOLE / L] `PendingChoice` opening *inside* an intervention response is unaddressed — ✅ RESOLVED (session 20, 2026-06-24): pre-halt death rule REMOVED → uniform cadence; (a) "it just works", (b) dissolved (nothing pre-settles)

The depth-1 cap bars a further **player intervention window** on a response ([§3:809](../specs/2026-05-26-game-mechanics.md#L809), [§4:1066](../specs/2026-05-26-game-mechanics.md#L1066)), and the pre-halt death rule covers `PendingChoice` as a "type-A halt" ([§4:1190](../specs/2026-05-26-game-mechanics.md#L1190)). But if a **responder's** invoked sigil has a Discover/targeting effect, its resolution issues `StartChoiceAction` → `PendingChoice` **mid-response** — a player choice that isn't an intervention window, so the depth-1 cap doesn't bar it. The interaction (does the pipeline halt for the responder's choice while a held action is suspended? does the pre-halt death rule fire here?) isn't stated.

- **Fix:** confirm and state that a response *may* open a `PendingChoice` (it's the responder's own choice, not a nested intervention), that the pipeline halts for it normally, and how it composes with the suspended held action. Likely "it just works," but it should be on the record (inscribed sigils already sidestep this by degrading choices to random — §3:849; only *invoked* sigils hit it).

---

## B. Checked and consistent (coverage — what was verified clean)

So you know the breadth, not just the failures:

- **F19 simplification is internally consistent** across its four touch-points (§1:111, §2A:319, §3:731, §3:548) + the Unaddressed entry — register-at-summon, enqueue-Battlecry-after, equal-epoch self-summon exclusion, later-epoch self-Battlecry reaction all agree. The "summon-triggered removal doesn't fizzle the Battlecry" property holds (Battlecry enqueued ahead + deferred death).
- **D2 is consistent** at every `SpellCastEvent` mention (§2A:318 disavows commit, §2A:319/§2B:462 = resolution, the "Fireball countered" trace #2 correctly shows no `SpellCastEvent`); `OnSpellCast`/`Subject` are timing-agnostic. *(The one residue is G2 — a pre-existing payload gap that D2 surfaced.)*
- **Directed/public event split** is complete: all five directed events pair with a public sibling, `MulliganStartedEvent` is the deliberate no-sibling case (F12); the `ChoicePromptEvent`/`ChoiceResolvedEvent` Discover-leak fix holds.
- **Epoch model** (creation-epoch filter, per-action RNG derivation, `birthEpoch`/`originEpoch`) is coherent across §3 `IEventBus`, `IRandom`, and the death-wave Phase-2 summon note (§4:1103).
- **Lane-based Friendly/Enemy** predicate is stated identically in `IEntityPredicate` (§3:666), `ITargetSelector` (§3:748), `IAura` scope (§3:887), and the `ITriggerCondition` sugar table — no controller-based drift remains.
- **Death cadence** (⑦/⑧ as settle stages, mortally-wounded vs destroy-mark savability, `All…/Alive…/Dead…` trichotomy) is consistent §1/§3/§4.
- **Rejection-code enum** (§2C) covers every code used in §4 ③ validation; the F8 intervention codes are present.
- **Combat-as-two-actions**, armor/Divine-Shield absorb precedence, fatigue-through-`DealDamageAction`, and the `GraveyardEntry` lazy-fabrication model are all self-consistent.

---

## C. Game-design recommendations (prior-art-grounded)

Not defects — design-space calls worth weighing before implementation hardens, each anchored to prior art. The user invited these explicitly.

### DR1 — Reactive-window pacing: borrow LoR's **spell-speed tiers** to stop *every* action opening a window

The intervention model opens an **unconditional** Stage-1 window at **every** ③′ and ⑥′ of the active player's cascade (for hidden-information reasons). This is the most LoR-like part of the design — and LoR is the cautionary tale: in LoR *"almost every single action allows the opponent to answer… gameplay is divided into mini-turns,"* which is interactive but **paces slowly**, and LoR mitigates it with **spell speeds** — *burst* spells resolve with **no response window at all**, *fast* don't pass priority, only *slow* opens the full window ([Mastering Runeterra: Priority](https://masteringruneterra.com/mastering-lor-priority/), [Dexerto: LoR vs HS](https://www.dexerto.com/league-of-legends/5-biggest-differences-between-hearthstone-and-legends-of-runeterra-1168130/)). HS deliberately ships **no** reactive windows at all for speed. The spec currently has *no* "burst" tier — even a deterministic, uninterceptable action (a pure draw, a stat buff with no legal answer in the format) still opens a window.

- **Recommendation:** add a **non-interceptable / "burst" class** of action (or a per-action `interceptable: bool` derived from whether any reactive card *could* answer it), so windows open only where a response is meaningful. This is the single biggest UX/pacing risk in the design; the `autoSkipAll` toggle + short timers (already in) are palliative, but a speed tier is the structural fix LoR validated. Worth heavy playtesting either way. (Complements the already-recorded "decision-window real-time budget" Unaddressed entry.)

### DR2 — The universal "pay mana, draw" starter artifact is a **Life-Tap-without-the-life-cost**, given to everyone

The shared starter artifact is *"pay mana, draw"* with an **escalating** per-turn cost and **no life cost**, equipped on **every** hero. Compare HS's **Life Tap** (Warlock: 2 mana **+ 2 health** → draw), *"widely considered one of the strongest Hero Powers in the game"* precisely because *"the ability to draw more cards than your opponent is always extremely powerful"* — so strong that Blizzard *"cannot give great cards to a class with the best Hero Power"* and **deliberately keep Warlock's cards middle-of-the-road** to compensate ([Life Tap, Liquipedia](https://liquipedia.net/hearthstone/Life_Tap), [Warlock balance, TheGamer](https://www.thegamer.com/hearthstone-warlock-hero-power-replacement-cancelled/)). The spec's starter is arguably **stronger** (no health downside) and **universal** (so there's no class-identity lever to balance around — every deck gets the best card-advantage engine).

- **Recommendation:** add a real cost beyond mana — a **life cost** (Life-Tap-faithful), a **v2 aether** cost, or a hard per-turn use cap — so it's a meaningful decision, not a free default. And reconsider whether the *card-advantage* engine should be the universal starter at all: HS's lesson is that "free repeatable draw" warps the whole format. A tempo/utility starter (a 1-damage ping, a 1/1 token) is lower-variance to balance than universal draw.

### DR3 — The neutral lane rewards **whoever's turn it is** more than skill — curate the pool and consider a contest mechanic

The neutral lane is a genuinely novel third side with no direct mainstream-CCG prior art; the closest analogues are **contested neutral objectives** in MOBAs/board games, where the lesson is that contested shared resources **reward initiative/tempo heavily** and can amplify variance. Two risks: (a) the **random** `spawnPool` draw (uniform with replacement) injects swing — a lucky high-value neutral spawn that the upcoming player gets "first crack" at (Interturn = turn-end, so the incoming player acts first) is a turn-order windfall; (b) there's no *contest* interaction — control is "whoever can spend a control card / attack it first."

- **Recommendation:** (1) **curate `spawnPool`** tightly (narrow stat band, no format-warping bodies) to bound the RNG swing — this is a content lever already in `NeutralZoneConfig`. (2) Playtest the **first-crack** asymmetry (turn-end handover gives the incoming player priority on fresh neutrals); if it's too strong, consider spawning at turn-*start*-of-the-player-who-just-acted, or a shared "both may contest before either's turn" beat. (3) Consider whether the lane needs a **skill-expressing contest** (e.g. last-hit/secure mechanics) rather than pure tempo, à la jungle objectives, so it's not just "free value to the tempo player."

### DR4 — Going-second compensation is HS-exact (Coin + extra card) — fine default, but monitor and consider LoR-style alternating initiative

Setup gives P2 **4 cards** + **The Coin** (+1 mana one-shot) vs P1's 3 ([§4:1152–1156](../specs/2026-05-26-game-mechanics.md#L1152)) — exactly HS. HS shows this *roughly* balances but going-first retains a small edge in aggressive metas. With this game's **extra** first-mover swing vectors (the intervention press economy refreshes off-turn; the neutral first-crack from DR3), the first/second balance may not match HS's.

- **Recommendation:** ship the HS model (it's a known-good baseline), but **instrument first/second win-rate from day one** and keep **LoR's alternating-initiative** model (the attack token alternates each round) on the table if the intervention/neutral systems prove to skew first-mover advantage beyond what the Coin offsets.

### DR5 — Reserve an explicit **alternate-win-condition** seam in ⑧

Win check ⑧ is hardcoded to "hero health ≤ 0" ([§4:1131](../specs/2026-05-26-game-mechanics.md#L1131)); deck-out/fatigue route through it as damage. Prior art (MTG mill/poison/"you win the game" cards, HS quests + Mecha'thun/Uther-DK alt-wins) shows alternate win conditions are a high-value design space — and the spec already gestures at counters (`ArtifactOnBoard.charges` for "quest-counter designs") and v2 aether/crafting. Retrofitting an alt-win later means surgery on ⑧.

- **Recommendation:** make ⑧ consult a small **win-condition predicate list** (default: the two hero checks) rather than a hardcoded health test — a cheap seam now that keeps "quest reward = win" / "mill = win" expressible later without re-touching the settle loop. Low effort, high optionality.

### DR6 — Revisit `MaxDeathWaves = 16` against *legitimate* deep boards before it voids real games

The 16-wave cap voids the match (`NoContest`) to protect MMR ([§4:1107](../specs/2026-05-26-game-mechanics.md#L1107)) — correct as a runaway backstop. But a **legal** board (e.g. many deathrattle-summon-deathrattle minions across 7+7+neutral) could plausibly chain a deep-but-finite cascade. Voiding a game a player legitimately built toward would feel terrible.

- **Recommendation:** keep the abort, but (a) treat 16 as a **tunable** to revisit with playtest telemetry on real cascade depths, and (b) ensure card-load validation (the stated primary mitigation) is genuinely catching the self-feeding patterns so the runtime cap only ever bites true bugs, not deep-but-finite legal boards.

---

## Suggested triage order

1. **G1** (Combo self-counting) — small, real, define the predicate.
2. **G2** (`SpellCastEvent` carries the card) — small fix, unblocks a whole cost-gated cast-synergy archetype.
3. **G4, G5** (TakeControl fizzle wording; orphaned intervention events) — cheap consistency cleanups, batch them.
4. **G3, G7** (MinionOnBoard tribe fields; setup-artifact epoch) — doc-completeness one-liners.
5. **G6, G8** (Enrage storage; PendingChoice-in-response) — one-sentence rulings, or defer to first card with a recorded note.
6. **DR1–DR6** — design calls; DR1 (window pacing) and DR2 (universal draw engine) are the two worth deciding before content design hardens.

---

## Plan impact (fold into the end-of-pass plan reconciliation)

Per the standing workflow, these review passes amend the **spec only**; the `plans/2026-05-27-ccg-game-logic/` epic+ticket files are swept in the one deferred reconciliation. Applied-finding plan touches so far:

- **G1** — the trigger-condition library ticket (Epic 08 T8.10 `ITriggerCondition`) must implement **`Combo` as `cardsPlayedThisTurn ≥ 2`** (not `> 0`). Epic 01 data-model `cardsPlayedThisTurn` carries the off-by-one note.
- **G2** — Epic 01/02: `SpellCastEvent` payload `{playerId, card, targetId?}` (was `cardId`); `ResolveCardAction` fields `{card, playerId, targetId?}` (drop separate `cardId`/`definitionKey`, fold into the carried `card` snapshot). The `PlayCardAction`/`ResolveCardAction` handler tickets (Epic ~05 card-play) snapshot `card` at commit and thread it to the resolution `SpellCastEvent`.
- **G4** — the `TakeControlAction` handler ticket: full destination board ⇒ enqueue `DestroyMinionAction` (original-owner death + Deathrattle), the `ReturnToHandAction` #21 pattern — **not** a `BoardFull` reject. `BoardFull` code stays (player minion-play ③).
- **G5** — **delete `InterventionPlayedEvent` + `InterventionWindowClosedEvent`** from **Epic 13** (`epic-13-intervention.md` T13.2 stops emitting `InterventionPlayedEvent`; T13.3 stops emitting `InterventionWindowClosedEvent` — window-close inferred, no event) and from the **README** T13.2/T13.3 lines + `Events/ChoiceEvents.cs` create-list. T13.3's "Skip / timeout path" stays; only the event is dropped.
- **G8** — the **pipeline / death-resolution epic** (the ⑦/⑧ settle + `PendingChoice` Interruption tickets, Epic ~03/04 engine): **do NOT implement any pre-halt early-settle.** ⑦ death / ⑧ win run **only** at queue-empty (⑨); a `PendingChoice` halts in place with the live (pending-death-inclusive) board. The `SubmitChoiceAction`/continuation ticket must allow a `PendingChoice` to coexist with a parked `pendingIntervention` (orthogonal §1 fields) for the response-nested case, and chained `Discover→Target` choices each open their own `PendingChoice`. A scenario test should cover **save-via-Discover** (AoE mortally wounds → invoked "Discover an effect" sigil → Discovered heal targets the still-on-board dying minion → it survives ⑦). Deletes whatever "pre-halt death rule / amendment B" wording exists in any pipeline ticket prose.
- **G6 (sizable)** — **Epic 08 T8.9** is currently "On Damage Taken trigger **+ Enrage**" and creates `Keywords/EnrageKeyword.cs` + `EnrageStateChangedEvent` + `EnrageScenarios.cs` with an "enchantment-vs-aura representation" decision. Rework: **keep** the generic **On-Damage-Taken trigger** (still needed for real damage-reaction cards), but **drop** `EnrageKeyword.cs`, `EnrageStateChangedEvent`, and the representation decision. Enrage moves to the **`IAura` ticket** as a *self-conditional aura* scenario test ("while damaged → +Attack"). **README T8.9** drops `+ EnrageStateChangedEvent`. **Epic 16** (`epic-16-remaining-effects.md`:24) Silence×Enrage note → reframe: Silence stops the minion emitting its **own aura** (bonus drops) — verify the aura-silence interaction, not an enchantment. Historical borrow-list lines ("Enrage as a stage-⑥ recompute participant", Epic 07 re-scope) are superseded by this self-conditional-aura model. **Forward-note (v2):** `notes/2026-06-14-inversion-v2.md` treats Enrage as a keyword in its offense↔defense pairing (Enrage↔Lifesteal) — revisit when inversion-v2 is taken up; Enrage is no longer a keyword.
