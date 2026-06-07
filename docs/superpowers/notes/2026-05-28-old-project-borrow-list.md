# Old Unity Project — Borrow List for `CCG.GameLogic` Spec

**Date:** 2026-05-28
**Source:** `C:\Users\vasil\UnityProjects\ccg` — earlier proof-of-concept implementation.
**Purpose:** Discrete items to discuss one at a time. For each: what's there, what it would add to our spec, recommendation, open questions. Items the new spec already covers (per-zone collections, Phase 1/2/3 death wave, trigger ordering by active player + board index, scenario test Spine/Exact traces, EntityId-style typed IDs, DeathProcessor/Service split) are deliberately omitted.

Items are roughly ordered by impact on the spec, not by topic.

---

## Item 1 — Stabilization loop iteration cap & explicit wave events

**What the old project does:**
The `Stabilize` method in `PlayCardResolver` (PlayCardResolver.cs:62–92) runs an outer loop that processes triggers, then a death wave, then re-checks for new events, until no new events appear. **No iteration cap. No wave-start/wave-end markers.** Relies on the assumption that "effects eventually stop creating new events."

**Why it might matter for our spec:**
Our spec §4 ⑦ describes the death-wave loop and says "back to step 1 (next wave)" but doesn't say what stops a runaway. A card designer could write something that loops indefinitely (e.g., deathrattle summons a 1/0 minion → dies → triggers itself → infinite). Right now nothing in the spec prevents this. Also: clients and tests have no signal for "the cascade just settled" — they have to infer it from event-list quiescence.

**Recommendation: Adopt.** Two small additions to §4 ⑦:
1. **Iteration cap:** define a `GameConstants.MaxDeathWaves` (e.g. 32). On exceeding, abort the action, roll back, emit a `StabilizationAbortedEvent`. Forces designers to think about termination; protects production.
2. **Wave markers:** emit `DeathWaveStartedEvent { waveIndex }` and `DeathWaveEndedEvent { waveIndex, minionsResolved }`. Cheap, audit-friendly, and the client can use them to gate animations.

**Open questions:**
- Cap value — 32 feels generous; what's reasonable?
- Should `StabilizationAbortedEvent` end the game, or just reject the action?

### ✅ DECISION (2026-05-29): ADOPTED, adapted

- **Cap = 16** (not 32). It's a runaway *detector*, not a budget — set just above legitimate worst case. Named constant `GameConstants.MaxDeathWaves`.
- **Cap trip → abort the match as `NoContest`** (not a draw — would unfairly hit MMR; not reject+rollback — rollback restores the exact state the bug re-triggers from, so the player could just replay it into the same hang). Publish `StabilizationAbortedEvent` then `GameEndedEvent { reason: NoContest }`, set `phase = Ended`. No rollback/snapshot dependency.
- **Wave markers adopted:** `DeathWaveStartedEvent { waveIndex }` / `DeathWaveEndedEvent { waveIndex, minionsResolved }`.
- **`reason: NoContest`** added to `GameEndedEvent`.
- **Telemetry = `StabilizationAbortReport { rngSeed, preActionState, triggeringAction, wavesReached, cascadeTrace[] }`** — formatted to drop straight into a scenario fixture (Build/Script/Assert) so the runaway is reproduced and the abort never regresses.
- **Reborn confirmed non-looping** (self-consuming in Phase 3) — discussed and ruled out as a loop source; runaway is always Phase 2 deathrattle/summon-driven.
- **Reborn keyword vs. charge split (related amendment, 2026-05-29):** the Reborn *keyword* is a persistent tag (counts for auras, survives bounce/copy, only Silence clears it); the one-time reborn *charge* is separate, modeled as `MinionOnBoard.rebornAvailable: bool`. Reborn firing consumes the bool, NOT the keyword. Charge initializes at summon to `keywords.Contains("reborn")`; the reborn-summon path summons with it false; a *copy* of a spent reborn minion is a fresh summon → charge re-initializes true. Considered a generic charge-container (`Dictionary<string,int>` + `IKeyword.InitialCharges`) but chose the bool (YAGNI — population of one, no other charge mechanics planned; trivially refactorable later). Spec amended: §1 `MinionOnBoard`, §3 Reborn keyword row, §4 ⑦ Phase 3.
- Real fix is upstream content validation at card-load time; runtime cap is the backstop (termination is undecidable).

Spec amended: §2B (events), §4 ⑦ (Death Resolution rewritten).

**Plan impact:**
- **Epic 04 T4.5** (`DeathResolutionService` wave loop) — add `GameConstants.MaxDeathWaves = 16`, `DeathWaveStartedEvent`/`DeathWaveEndedEvent`, and the abort path (`StabilizationAbortedEvent` → `GameEndedEvent{NoContest}` → `phase=Ended`; no rollback).
- **Epic 04 T4.1** (`MinionOnBoard`) — add `rebornAvailable: bool`.
- **Epic 03 T3.5** (`GameEndedEvent` / win check) — add `NoContest` reason.
- **Epic 08 T8.5** (Reborn / Phase 3) — REWRITE: consume `rebornAvailable`, **keep** the keyword; reborn-summon path sets `rebornAvailable=false`; a copy re-initializes it from the keyword. (Current ticket text says "reborn removed" — wrong under the new model.)
- **NEW ticket needed** (no telemetry concept in the plan) — `StabilizationAbortReport` sink + scenario-reproducible report (`rngSeed`/`preActionState`/`triggeringAction`/`wavesReached`/`cascadeTrace`). Suggest Epic 04 **T4.8** (depends on Item 6 `IRandom` for `rngSeed`).

---

## Item 2 — Pre-built `ITriggerCondition` singletons (composable filters)

**What the old project does:**
`ITriggerCondition` (ITriggerCondition.cs:6–45) has a single `Matches(...)` method. Four pre-built singletons exist: `AlwaysCondition`, `SelfIsRelatedEntityCondition` ("the dying minion is me"), `FriendlyRelatedEntityCondition`, `EnemyRelatedEntityCondition`. Card definitions reference these by reference, not by string. No allocation per check, no JSON DSL.

**Why it might matter for our spec:**
Our spec defines `ITrigger.ShouldFire(GameEvent, GameState)` (§3) as an open method. Card definitions go through `definition: JsonElement` (Section 1). The implication is that every card with a trigger writes its own predicate logic — which is fine for unique cards, but the 80/20 case ("fire only on friendly minion death," "fire only when self is the source") will repeat across dozens of cards. Without canned conditions, every card author re-implements those filters and we lose a regression-test target.

**Recommendation: Adopt.** Define a small library of `ITriggerCondition` building blocks in the spec under §3:
- `AlwaysFires`
- `SelfIsSource` / `SelfIsTarget` / `SelfIsRelated`
- `FriendlyOnly` / `EnemyOnly` (by ownerId vs trigger source's owner)
- `MinionTypeIs(string definitionKey)` (for tribal triggers)

`ITrigger` implementations compose one or more of these via `And`/`Or` combinators. `DefaultCardHandler` reads `definition.condition` JSON (e.g. `"FriendlyMinionDeath"`) and maps to a canned condition; custom cards still implement `ITrigger` directly.

**Open questions:**
- Should this live in the spec or be left as an implementation detail of `DefaultCardHandler`? Putting it in the spec gives every card the same vocabulary.

### ✅ DECISION (2026-05-30): ADOPTED, in the spec

- **`ITriggerCondition` added to §3** as a first-class interface — `bool Matches(GameEvent evt, GameState state, string sourceId)`. `sourceId` is the trigger's host entity; conditions evaluate the event relative to it. `ITrigger.ShouldFire` typically delegates to a condition (or tree).
- **Single-condition library, two shapes:**
  - *Parameterless singletons:* `AlwaysFires`, `SelfIsSource`, `SelfIsTarget`, `SelfIsRelated`, `FriendlyOnly`, `EnemyOnly`.
  - *Parameterized factories:* `MinionTypeIs(definitionKey)`, `CardTypeIs(CardType)`, `CostAtLeast(n)`/`CostAtMost(n)`. (The parameterized shape was surfaced by the user's "2-mana-or-higher spell" example — a numeric threshold predicate that the flat-singleton list would have missed.)
- **Combinators adopted:** `All` (AND) / `Any` (OR), nestable. Rated low-effort/high-leverage: the combinator *classes* are trivial (~6 lines, `All`/`Any` over children); the only real (still small) work is the recursive JSON descent in `DefaultCardHandler`, which the compound-condition requirement forces regardless of whether it's called a "combinator." `Not` deliberately deferred (same shape, add on first need — e.g. "other friendly minion died").
- **Multiplicity confirmed as an orthogonal axis.** A card = handler = N triggers (the spec already supported this via "ICardHandler registers triggers, plural"); now made explicit with a `definition.triggers` array. Combinators compose conditions *within* one trigger; multiple effects watching different events are *separate* triggers. The user's two-line example (summon-on-spell + draw-on-beast-death) exercises both axes at once and is embedded in the spec as the worked example.
- **Battlecry / Deathrattle confirmed to fit the same `{ type, condition, effect }` shape** (Battlecry = `MinionPlayed`+`SelfIsSource`; Deathrattle = `MinionDied`+`SelfIsRelated`). Key clarification recorded in spec: **conditions gate (*whether*), trigger `type` routes (*when/in what order*).** Battlecry resolves synchronously in the PlayCard pipeline (may pause for targeting); Deathrattle resolves in death-wave Phase 2 off the death snapshot — neither fires at raw publish time. The condition library does not alter these resolution semantics (§4 owns them).
- **JSON encoding of the condition tree left as `DefaultCardHandler` implementation detail** — explicitly out of the spec; settles when the first compound card is authored.

Resolved the open question above (spec vs. impl detail): **the interface + library + combinators go in the spec** (shared vocabulary, shared test target); only the wire encoding stays in the handler.

Spec amended: §3 (`ITrigger` cross-reference + new `ITriggerCondition` subsection).

**Plan impact:**
- **Epic 08 T8.1** (`ICardHandler` + `ITrigger` infra + `DefaultCardHandler`) — add `Interfaces/ITriggerCondition.cs`, the single-condition library (parameterless singletons + parameterized factories), and `All`/`Any` combinators; introduce the `definition.triggers[]` array and the `DefaultCardHandler` JSON→condition mapping. T8.1 is already dense — recommend splitting the condition library into a **NEW ticket T8.10** (`ITriggerCondition` library + combinators, with its own unit tests as the shared regression target).
- **Epic 08 T8.6 / T8.8** (Start-of-turn / On-Friendly-Death) — restate their `ShouldFire` predicates as conditions (`FriendlyOnly`, `Not(SelfIsRelated)`, etc.) rather than ad-hoc inline checks; note `Not` is added here on first need.

---

## Item 3 — Snapshot triggers before processing (registration-during-cascade safety)

**What the old project does:**
`TriggerProcessor` (TriggerProcessor.cs:65–107) **snapshots the trigger list at the start of an event batch.** Triggers registered mid-cascade (e.g. a deathrattle that summons a minion with its own trigger) do **not** fire on events already in flight — they only catch *subsequent* events in the next batch.

**Why it might matter for our spec:**
Our spec §3 `IEventBus` says "publish ordering — current player's list first, then opponent's; sorted by current board index at publish time." It doesn't say *what list*: the live subscriber list at the moment of publish, or a snapshot taken at the start of the publish batch. These differ exactly when a trigger handler installs a new trigger while the bus is iterating. Worth being explicit in the spec.

**Recommendation: Adopt and document.** Add a sentence to §3 `IEventBus`: *"Subscribers for a given event are snapshotted at publish time. Listeners registered during dispatch of that event do not receive it; they receive subsequent publishes only."* Same applies during the death-wave Phase 2 — new triggers from reborn minions don't retroactively fire on deaths already published this wave.

**Open questions:**
- Does the spec want the snapshot at *individual event publish* or at *action-handler batch start*? Different answer for "does an event fired during the action see triggers from a minion summoned earlier in the same action."

### ✅ DECISION (2026-05-31): ADOPTED — per-event snapshot + creation-epoch filter

Chose **per-event snapshot at publish time, plus a creation-epoch visibility filter** — which delivers the old project's "registered mid-cascade → subsequent events only" behavior (and Hearthstone's "a new minion never triggers off the event that introduced it") *uniformly*, without the heavier per-batch threading.

- **Mechanism:** `GameEngine` keeps a monotonic `currentActionEpoch`, incremented as each action enters stage ④. Subscribers are stamped `birthEpoch`; events are stamped `originEpoch`; `Publish(E)` dispatches to listener `L` iff `L.birthEpoch < E.originEpoch`. Equal epoch → excluded, so a listener never sees events from the action that created it (incl. its own `MinionSummonedEvent`).
- **Why epoch over per-batch threading** (resolves the open question — chose per-event, not per-batch): same observable behavior, but per-batch threading is *heavy* — it copies the subscriber registry per action, leaks a "batch" concept into the `IEventBus` contract (extra param or `BeginBatch`/`EndBatch` ambient state), introduces temporal coupling (pairing must survive the stabilization-abort tear-down), forces a batch-boundary definition (action vs. wave vs. cascade) that is itself a bug surface, and is non-local to debug. Epoch is a pointwise integer compare on the *live* list: lighter **and** strictly more robust (no stale frozen list can fire a removed subscriber; no intra-handler ordering dependency). Rejected the earlier "register-after-own-summon-event" deferral idea too — it was ad-hoc per event type (fragile); epoch is uniform.
- **Robustness driver (the deciding factor):** new entity-introducing events (transform, resurrect, copy, …) are covered by the *same* rule with zero extra code, vs. the deferral approach where each must individually replicate the ritual or silently misbehave.
- **Supporting invariant added to spec:** *all bus subscriptions happen inside action handlers* (stage ④ — `OnSummon`, `IKeyword.OnApplied`); nothing subscribes during publish (⑤) or aura recalc (⑥). This is what makes the epoch comparison total.

Spec amended: §2B (event base gains `originEpoch`), §3 `IEventBus` (snapshot + epoch rule + invariant) and Deterministic Ordering table (new row), §4 ④ (epoch increment + stamping), §4 ⑤ (snapshot + filter on publish), §4 ⑦ Phase 2 (death-wave cross-reference).

**Plan impact:**
- **Epic 01 T1.3** (`GameEvent` base) — add `originEpoch: int` alongside `OccurredAt`.
- **Epic 01 T1.4** (`IEventBus` interface) — subscribers carry a `birthEpoch`; document the epoch contract.
- **Epic 01 T1.5** (`EventBus` impl) — snapshot subscriber lists at publish + filter by `birthEpoch < originEpoch`; add an epoch-visibility scenario test (a listener does not receive the event from the action that created it). Folds naturally into the existing ordering tests.
- **Epic 01 T1.8** (`GameEngine` pipeline) — maintain `currentActionEpoch`, increment as each action enters stage ④, stamp returned events' `originEpoch` and new subscriptions' `birthEpoch`.

---

## Item 4 — Effect-op result chaining + variable amounts (Spell Damage +X)

**What the old project does:**
Ops (`IOp.Execute(EffectContext)`) take a context, mutate state, return nothing. **No result passed between ops.** Damage amount is hardcoded in the op constructor (`DealDamageOp.cs:10` — `new DealDamageOp(3)`), so Spell Damage +X cannot bump it without rewriting the op. This is a known limitation the old project flagged for "future work."

**Why it might matter for our spec:**
Spell Damage +X is in our locked keyword list (§ "Decided feature scope"), so the spec needs to resolve this somewhere. Two questions our spec doesn't answer:
1. **Variable amounts:** how does an effect that says "deal 3 damage" actually become "deal 4 damage" when the caster has Spell Damage +1? The spec mentions in §3 `IKeyword` row that "Spell Damage +X" listens to `SpellCastEvent` and "bumps spell damage amount in context" — but `EffectContext` in §3 `IEffect` doesn't define a `SpellDamage` field. Loose end.
2. **Op chaining:** can a later op reference the *outcome* of an earlier one (e.g. "if any damage was dealt, draw a card"; "summon a copy of the minion you just killed")? Old code can't; some cards in any CCG benchmark require this.

**Recommendation: Adapt.** Two additions to §3:
- **`EffectContext` gains a mutable `SpellDamageBonus: int` and a `Results: List<OpResult>` ledger.** Damage ops read the bonus before applying and append `DamageOpResult { targetId, dealt, killed }` to `Results`. Subsequent ops in the same effect can inspect prior `OpResult`s.
- **Spell-damage application is centralised** in `DealDamageAction`'s handler (not the keyword listener): the handler reads `EffectContext.SpellDamageBonus` if the source action is a spell, adds it to `amount`, then applies. The keyword listener only sets the bonus before the spell resolves. Keeps the math in one place.

**Open questions:**
- Should `Results` be cleared between effects or persist across the entire action's cascade?
- Are there spell-damage interactions we want to *forbid* (Spell Damage on Hero Power damage, e.g.) and where would those filters live?

### ✅ DECISION (2026-05-31): ADAPTED — split in two; leaned down hard from the note's synchronous-op framing

The note's recommendation was written against the old project's **synchronous op model** (ops mutate state inline, could hand results to the next op). **That model is not ours** — our spec (§3 `IActionQueue`) has effects *enqueue* actions; they never mutate state or get a synchronous result back. That single difference splits the item:

**(A) Variable amounts / Spell Damage +X → ADOPTED, via a read-only *pulled* snapshot; Spell Damage demoted from active keyword to declarative.**
- `EffectContext` gains **`SpellDamageBonus: int`**, computed **once at spell-resolution start** and held for the whole cast. Spell-scoped, not hit-scoped (a two-hit spell applies it to both; killing your own Spell-Power minion mid-cascade doesn't drop it for the second half). The shared `DealDamageEffect` bakes it into the enqueued `DealDamageAction`; the handler stays dumb.
- **Spell Damage +X moves out of the Active-keyword (listener) table into the Declarative list.** No `SpellCastEvent` listener, no mutable shared context field, no fire-before-read ordering coupling. The bonus is *pulled* (aggregated on demand), not *pushed*. This is strictly leaner **and** closes the "forbidden interactions" open question for free: the bonus is `0` for non-spell sources, so it structurally cannot leak into combat or hero-power damage.
- **Magnitude lives in the keyword string** (`spell_damage:1`) — no new model field (population is tiny).
- **Aura-granted spell power** reuses the existing `grantedKeywords` rail (an aura grants `spell_damage:X`, rewritten each recalc pass); intrinsic and aura-granted spell power flow through the same aggregation. No new `AuraEffect` channel needed.
- **v1 source list = intrinsic keyword + aura-granted keyword.** That covers the locked "Spell Damage +X" scope.

**(B) Op result chaining / `Results` ledger → REJECTED for v1; redirected to the event bus.**
- "React to an outcome" in our architecture *is* the event bus, hardened by Item 3's epoch filter. "If damage dealt → draw" = an `ITrigger` on `DamageTakenEvent`; "summon a copy of what you killed" reads the death snapshot off `MinionDiedEvent`. A synchronous `Results` ledger is a foreign synchronous-model concept that duplicates the bus. Consistent with **Item 10**, already deferred. Kept on radar only if a card needs same-action *synchronous* inspection the bus genuinely can't express (none expected).

**Player-scoped spell-damage modifiers ("+1 all spells this turn", "+3 next spell") → DEFERRED to the future Modifier System** (`docs/superpowers/notes/2026-05-31-modifier-system.md`). Surfaced during discussion: spell damage and **mana-cost reduction** are the *same shape* (scoped/filtered/lifecycle-bound modifier to a named quantity, aggregated at a read point). The only forward-compatibility constraint mana reduction places on spell damage today is "don't bake a bespoke player-scoped modifier store into spell damage" — honored by simply not building one. The minion-bound + snapshot parts (above) are local to spell resolution, share nothing with cost, and cannot paint us into a corner, so Item 4 resolves now without waiting on the Modifier System. **No reserved `spellPowerModifiers` field** — that would itself be the premature commitment.

Resolved open questions: spell-damage scope = **direct spell damage only** (snapshot at cast; cascade damage attributed to its own source per Item 9); forbidden interactions handled structurally by the `0`-for-non-spell rule. The `Results`-clearing question is moot (rejected).

Spec amended: §3 (Declarative-keywords paragraph — Spell Damage added + pull/aura explanation; Active-keyword table — Spell Damage row removed; `IEffect` — `EffectContext.SpellDamageBonus` defined with snapshot/aggregate/scope rules).

**Plan impact:**
- **Epic 08 (keywords)** — Spell Damage implemented as a **declarative** keyword (no listener): a `spell_damage:X` parser + the friendly-board aggregate. Remove any "Spell Damage registers a `SpellCastEvent` listener" framing from the keyword ticket.
- **Epic 08 / aura ticket** — Spell-Power aura grants `spell_damage:X` via `grantedKeywords` on recalc (no new `AuraEffect` field).
- **Spell-resolution / `DefaultCardHandler` ticket** — compute `EffectContext.SpellDamageBonus` once at cast start; `DealDamageEffect` reads it and bakes it into `DealDamageAction`. Add a scenario test: two-hit spell applies bonus to both hits; self-killing-Spell-Power-minion spell keeps bonus for second half; non-spell damage gets `0`.
- **No new ticket for `Results` ledger** (rejected). **No player-scoped-modifier work** here (deferred to Modifier System — see that note for its own plan impact when scheduled).

> **Correction (2026-06-01, via Item 8 follow-on):** aura-granted `spell_damage:X` was specified here to flow into `grantedKeywords`. That was wrong — `grantedKeywords` is the permanent, bounce-retained set, so an aura grant would have survived a `RetainEnchantments` bounce. The keyword model was expanded to a 4-field form (`intrinsic`/`granted`/`aura`/effective `keywords`); aura `spell_damage` now flows into **`auraKeywords`** (recomputed each pass, never retained). Spell Damage remains declarative, so the aura grant is permitted. See Item 8 → "Follow-on amendment".

---

## Item 5 — `GetLegalActions(playerId)` API surface

**What the old project does:**
**No such API exists.** Bots would need it; the bot folder is stubs. The client UI also benefits from a "which moves can I make right now" query — cards in hand that can't be afforded would be greyed out, targets that aren't legal can't be selected, etc.

**Why it might matter for our spec:**
Our spec defines a strict pipeline that *rejects* illegal actions, but exposes no positive answer to "what's legal?" Two consumers will want it: bots (Epic for AI isn't on the plan but is on the radar), and the client (Platform API will eventually need it for client-side validation pre-flight). Building it as a first-class engine method is much cheaper than re-implementing the validator outside the engine.

**Recommendation: Adopt.** Add to §3 a new top-level interface (or method on `GameEngine`):
```
IReadOnlyList<GameAction> GetLegalActions(string playerId);
```
Implementation: enumerate every plausible action shape (PlayCard for each card in hand at each viable target; AttackAction for each minion at each legal target; UseHeroPower, EndTurn, etc.) and filter through PhaseGuard + ActionValidator without applying. This makes the action validator the single source of truth for "is this legal," which is exactly the property you want.

**Open questions:**
- This call could be expensive on large boards × full hands × many targets. Do we cache, or accept O(hand × targets) latency? Probably fine for 1v1 CCG sizes.
- Do we expose this in `CCG.GameLogic` or only at the Game Server layer? Recommend: in the library.

### ✅ DECISION (2026-05-31): ADOPTED — seam now, enumerator deferred to a future bot-support epic

Split the item the way Item 4's player-scoped modifiers were split: lock the cheap architectural seam in the spec now; defer the speculative machinery to where its consumer lives.

- **Lock now (spec, v1): validation (②③) is a pure, side-effect-free, standalone-invokable predicate** `(action, state) → Ok | Rejection` — no mutation/dispatch/events — and the **single source of truth for legality**. `Submit` calls it before ④; any "is this legal / what's legal" query must call the *same* predicate, so enumerated legality can never drift from what `Submit` accepts. (Our pipeline already separates ②③ from ④, so this is mostly making it explicit + a named unit.) Spec amended: §4 ③.
- **`GetLegalActions(playerId)` itself → DEFERRED to a new future bot-support epic** (see Plan impact). No consumer in core v1 scope (bots are a deferred subsystem; client pre-flight is a Game-Server/client concern; tests can hit the predicate directly). Building the enumerator now would be YAGNI — **but the user has confirmed bots are definitely coming**, so it is *deferred, not rejected*, and gets a home.
- **Home = `CCG.GameLogic`** (confirmed, not the Game Server). The library must be complete without depending on the server; the validator lives in the library, so the enumerator that reuses it does too — and tests, bots, and the Unity client (which references the library directly) all consume it. Pure enumeration + validation, no framework deps, so it doesn't violate the "no engine/framework deps" rule.
- **Approach = brute-force-then-filter** (locked): generate candidate actions (`PlayCard` per hand card × every entity; `Attack` per minion × every entity; `HeroPower`; `EndTurn`; …) and let the standalone predicate reject the illegal ones. Reuses the validator exactly, handles edge cases for free (a must-target spell on an empty board yields zero legal pairs → correctly unplayable), and **decouples this from Item 12** (no target *enumerator* needed — only the existing target *validator*). O(hand × entities) is fine at 1v1 sizes; no caching (YAGNI).
- **Phase-aware:** runs Phase Guard first, so in `PendingChoice`/`PendingIntervention` it surfaces the already-enumerated options from the pending-state object; meaningful primarily in `InProgress`. **Do not enumerate Mulligan keep-subsets** (2^handsize) — leave that to `MulliganState`.

**Plan impact:**
- **Epic 01 / validator ticket** (Phase Guard + Action Validator) — expose ②③ as a **pure, standalone-invokable predicate** `(action, state) → Ok | Rejection`, callable independently of `Submit`/dispatch. `Submit` consumes it. This is the only v1 change. (Pairs naturally with **Item 7** structured rejections when that's decided — the predicate returns the rejection code.)
- **NEW EPIC (end of plan): Bot Support.** Created during reconciliation to house `GetLegalActions(playerId)` (brute-force-then-filter, library-resident, phase-aware) **plus the other logic a bot needs** (move generation/scoring scaffolding, state evaluation hooks, a bot driver loop, etc. — to be fleshed out when the epic is written). `GetLegalActions` is its first ticket. Depends only on the ②③ predicate seam above. Add to the README epic index during reconciliation.

---

## Item 6 — Deterministic `IRandom` interface with seeded RNG

**What the old project does:**
**No RNG infrastructure.** No `IRandom`, no seeded calls, no random discards or random targets. Effects are deterministic only because none of them involve randomness yet.

**Why it might matter for our spec:**
Several actions on our list are inherently random: `DiscardCardAction { cardId? — null = random }` (§2A), discover (Epic 12), and any future "random enemy minion" target. Our spec doesn't define where the randomness comes from, and that's a determinism hole. Without a seeded `IRandom`, replay is impossible — and the spec already references replay as a future capability ("Event-log replay" in Epic 16 backlog).

**Recommendation: Adopt.** Add to §3:
```
IRandom { int Next(int maxExclusive); int Next(int minInclusive, int maxExclusive); }
```
`GameState.rngSeed: ulong` is set at game creation and threaded through `IRandom`. Every random pick uses `IRandom` — no `Random.Shared` anywhere. Document in spec: *"Replay determinism requires that every random call go through `IRandom`. Implementations must be reproducible given seed."*

Tests can pass a deterministic seed for repeatable scenarios.

**Open questions:**
- Should the seed be public (so clients can verify replay) or hidden (so opponents can't predict)? Probably hidden, but tests need access.
- One RNG per game or per player? One per game is simpler; per player avoids correlated rolls.

### ✅ DECISION (2026-06-01): ADOPTED — counter-based per-action reseed; shuffle at init from seed

The `IRandom` interface is adopted as recommended. The substance was in *how* reproducibility works, and we landed on the **counter-based / splittable-RNG** model rather than a single advancing stream.

- **Interface (spec §3 `IRandom`):** `int Next(int maxExclusive)` + `int Next(int minInclusive, int maxExclusive)`. Every random pick goes through an injected `IRandom`; no `Random.Shared`/clock-seeded/ambient RNG anywhere. Stated as the determinism contract.
- **Seed (spec, `GameState.rngSeed: ulong`):** fixed at match creation, never mutates. Both *open questions resolved by architecture, not preference*: **visibility** — the client is a pure event renderer and never receives `GameState`, so the seed is structurally hidden (can't be used to predict draws); tests inject a chosen seed. This is exactly the PvP-CCG industry pattern (Hearthstone/MTGA keep the seed server-side and ship event logs; the seed *is* hidden information). **One-RNG-per-game vs per-player** — moot under the counter-based model: there is one logical source per match, split per action by epoch; per-player streams add nothing in 1v1. YAGNI.
- **Per-action reseed by epoch (the real decision):** **no single long-lived PRNG.** At the start of each action's stage ④ the engine derives a fresh `IRandom` from `mix(rngSeed, currentActionEpoch)` (strong mixer, e.g. splitmix64). Picked over a single advancing stream because the already-locked `StabilizationAbortReport` hands you a *mid-game* `preActionState` and must reproduce a single action in isolation — a continuous stream would force serializing PRNG internal state into every snapshot; counter-based makes any action reproducible from `(rngSeed, epoch, preActionState)` with **zero PRNG-state serialization**. This is the JAX-key-split / NumPy `SeedSequence.spawn` / Slay-the-Spire-per-domain-stream model, and it **reuses the `currentActionEpoch` already introduced for Item 3's event-visibility filter** — one counter, two jobs.
- **`currentActionEpoch` moved into `GameState`** (minor refinement to Item 3, which originally placed it on `GameEngine`). Rationale: making RNG a pure function of state (`mix(state.rngSeed, state.currentActionEpoch)`) makes snapshots self-describing — `StabilizationAbortReport.preActionState` now carries the epoch, so the report needs **no new field** to be reproducible. Matches the user's articulated mental model exactly ("every action carries the gamestate, the epoch, and the seed").
- **Where randomness is consumed:** only in stage ④ handlers (the sole mutation point). Effects/triggers needing a random outcome enqueue an action whose handler rolls (`DiscardCardAction{cardId=null}`, Discover option generation, random-target actions). Selector RNG access (Item 12's open question) **deferred to Item 12**.
- **Deck shuffle → at init, from the seed (Fork B decided by user):** engine runs a Fisher-Yates over `IRandom` at match setup; **decklists are the stored input, shuffled order regenerates from `rngSeed`.** Coin flip + opening deal seeded the same way.
- **Replay model (clarified, ties into Item 13):** a full game = `rngSeed` + both **decklists** + the ordered log of **player actions** only. System actions, random rolls, and the whole event cascade regenerate; `currentActionEpoch` reconstructs by counting actions (not stored in the log). Mid-game slice repro (abort report) relies on the epoch carried in the captured `GameState`.

Spec amended: §1 `GameState` (+`rngSeed`, +`currentActionEpoch`), §3 new `IRandom` subsection + Deterministic Ordering table row, §4 ④ (per-action RNG derivation), §4 ⑦ `StabilizationAbortReport` comment (epoch-carried-in-state reproducibility).

**Plan impact:**
- **Epic 01 — NEW foundation ticket (suggest T1.9): `IRandom` + `DeterministicRandom`.** `IRandom` interface, a splittable/counter-based impl, the `mix(seed, epoch)` derivation helper, and a `Shuffle` (Fisher-Yates) extension over `IRandom`. Unit tests: same `(seed, epoch)` ⇒ identical sequence; adjacent epochs uncorrelated; `Shuffle` reproducible.
- **Epic 01 T1.x (`GameState`)** — add `rngSeed: ulong` and `currentActionEpoch: int` to the state record.
- **Epic 01 T1.8 (`GameEngine` pipeline)** — at stage ④, derive the per-action `IRandom` from `mix(rngSeed, currentActionEpoch)` (the epoch increment already lands here from Item 3); thread it to the dispatched handler.
- **Epic 04 T4.8 (`StabilizationAbortReport`)** — confirm `preActionState` carries `currentActionEpoch` so the report is self-contained (no separate epoch field). Dependency on this ticket (flagged in Item 1) is now satisfied.
- **Game-setup ticket (shuffle/coin/opening deal)** — perform the shuffle via the `Shuffle` extension over the per-action `IRandom`; store decklists, not shuffled order.
- **Epic 16 / replay backlog (Item 13)** — record the replay package as `rngSeed` + decklists + player-action log; do not log system actions or epochs.

---

## Item 7 — Structured error codes alongside string messages

**What the old project does:**
`Result.Fail("Card is not in hand.")` (PlayCardResolver.cs throughout). Tests assert exact string equality. Works, but hardcodes UI copy in the engine and ties tests to wording.

**Why it might matter for our spec:**
The Game Server will return rejection reasons to the client. The client wants to localise / format these. Tests want stable identifiers, not English strings. Internationalization aside, the engine should be a structured-error producer, not a UI string source.

**Recommendation: Adopt with adaptation.** §2 or §3 introduces:
```
enum ActionRejectionCode {
    NotYourTurn, NotInHand, NotEnoughMana, BoardFull,
    InvalidTarget, TauntMustBeTargeted, AttackerCannotAttack,
    PhaseDoesNotAllowAction, /* … */
}
record ActionRejection(ActionRejectionCode Code, string Detail);
```
Engine returns rejections; UI layer maps codes to localised strings; tests assert on `Code`. `Detail` carries optional diagnostic context (e.g. "needs 5, has 3 mana") for logs.

**Open questions:**
- Do we want a rejection event published on the bus, or just a return value from `Submit`? Recommend return value only — rejected actions never enter the event log.

### ✅ DECISION (2026-06-01): ADOPTED — code + optional non-contractual Detail; return value only

Structural work was already done by **Item 5** (validation = pure predicate `(action, state) → Ok | Rejection`); Item 7 just gives `Rejection` its shape. Adopted as recommended, leaned to the middle on the payload question.

- **`ActionRejectionCode` enum (closed)** — the legality vocabulary; stable identifiers tests assert on, codes the Game Server maps to localized client strings. Members derive directly from the §4 ②③ rules (`WrongPhase`, `NotActivePlayer`, `CardNotInHand`, `NotEnoughMana`, `InvalidTarget`, `TargetStealthed`, `MustTargetTaunt`, `AttackerCannotAttack`, `AttackerFrozen`, `AttackerExhausted`, `BoardFull`, …). Grows in lockstep with validation rules; single producer (the Item 5 predicate).
- **`record ActionRejection(ActionRejectionCode Code, string? Detail = null)`** — chose **code + optional Detail** over code-only and over structured payload. `Code` is the contract; **`Detail` is free-form, logs-only, explicitly non-contractual** (never asserted in tests, never shown to users, may be null). **Typed per-code payloads rejected as premature** (the deciding reasoning, recorded in discussion): the payload data is reconstructable by the caller from `(action, state)`; no current consumer (tests assert on `Code`; client localizes from `Code`; logs have `action`+`state`) is blocked without it; it would add a per-code type + wire encoding + client branch per rule against an open rule set; and the lean options keep a clean upgrade path (promote *one* code to a payload later if one is ever found that genuinely can't be reconstructed). Detail is the pragmatic middle — a cheap diagnostic without committing to a schema.
- **Delivery = return value only** (resolves the open question). Rejection is the negative arm of `Submit`'s result; **never a `GameEvent`, never on the bus, never in the event log.** Consistent with Item 5 (validation side-effect-free) and Item 6 (only real state-changing player actions are logged; rejected actions re-reject on replay). The Game Server relays the `Rejected` result to the originating client as that request's response. No `ActionRejectedEvent`.
- **`Submit` return type promoted** to a discriminated `SubmitResult = Accepted(IReadOnlyList<GameEvent>) | Rejected(ActionRejection)` (was "returns the full event list"). Falls out of the above; not a separate decision.

Spec amended: new **§2C Action Rejection** (enum + `ActionRejection` + `SubmitResult`), §4 ① (Submit returns `SubmitResult`), §4 ② (`Rejected(WrongPhase)`), §4 ③ (each precondition annotated with its code; returns `Rejected`), §4 ③ predicate paragraph (cross-ref to §2C). Also fixed the §3 lead-in that hard-counted "six interfaces" (stale since Item 2's `ITriggerCondition` and Item 6's `IRandom`) to a non-brittle extensibility/infrastructure listing.

**Plan impact:**
- **Epic 01 / validator ticket** (Phase Guard + Action Validator, the Item 5 predicate) — define `ActionRejectionCode` + `ActionRejection`; the predicate returns `Ok | Rejection`. This is the single producer of codes. Unit tests assert on `Code`, not strings.
- **Epic 01 T1.8 (`GameEngine` / `Submit`)** — change the return contract to `SubmitResult` (`Accepted` | `Rejected`); ②③ failures short-circuit to `Rejected` before ④.
- **All Epic 01–16 validation tests** — assert on `Code` rather than English strings (the old-project pattern this item replaces). Establish the `Code`-assertion convention in the first validator ticket so later epics follow it.
- **Game Server (out of `CCG.GameLogic`, future)** — owns code→localized-string mapping and relaying `Rejected` over WebSocket; library stays UI-string-free.

---

## Item 8 — Card definition hooks for art / description / sound / tribe

**What the old project does:**
`CardDefinition` carries `CardId`, `Name`, `Type`, `ManaCost`, base stats, and behaviour hooks (PlayAction, Triggers, OnDeathEffects). **No art key, description text, sound cues, tribe/race ("Beast", "Demon"), or rarity.** The Unity SO layer adds none of these either.

**Why it might matter for our spec:**
We have `rarity` on `Card`. We don't have anything else presentation-layer or gameplay-tag related. The library is supposed to be standalone — it shouldn't reference art assets — but it does need to carry an opaque art key string for the client to resolve. Same for description text (for tooltip rendering) and tribe (which is gameplay-relevant — "all Beasts get +1/+1").

**Recommendation: Adopt selectively.** Extend `Card` definition with:
- `artKey: string` — opaque identifier the client resolves
- `description: string` — pre-templated for localisation later
- `tribes: string[]` — gameplay-relevant tags ("Beast", "Demon", …); empty for non-tribal
- `sfxKey?: string` — optional, opaque

Leave the actual asset resolution entirely to the client; the library treats these as opaque strings. Tribes might want a typed enum if the set is small and locked, but `string[]` is more flexible.

**Open questions:**
- Tribes: typed enum (locked set) or open strings (any tag)? Hearthstone has ~10 tribes; we could lock them. But open strings let card-crafting (T16.6) invent new ones, which is a feature.

### ✅ DECISION (2026-06-01): ADOPTED SELECTIVELY — tribes in as a `[Flags]` enum; presentation out of the library

Split on one line: **does game logic read it, or only the client?** Tribes are read by logic → first-class typed; art/description/sfx are client-only → not in the library at all (leaner than the note's "carry an opaque artKey").

- **Tribes → `[Flags] enum Tribe`** (resolves the open question; user initially leaned enum, and on analysis the enum is the better fit — the recommendation was *revised* away from the string[] lean). The deciding insight: **keywords are strings because each resolves to an `IKeyword` *implementation*; a tribe resolves to nothing — it's an inert label, all behavior lives in the cards that reference it.** So the string-keyed-extension-point rationale doesn't transfer; tribes are a closed designer-curated taxonomy, the textbook enum case. `[Flags]` specifically because the requirement is **multi-tribe + grantable**: membership = `(tribes & Tribe.Beast) != 0`, granting = `|=`, removal = `& ~` — allocation-free, cheap in hot aura-recalc, multi-value native. The only cost (closed set → recompile to add a tribe) is a non-issue: new tribes ship with the content release that adds their cards, which is a library version bump regardless. String[] would only win for hot-adding tribes to a *running, un-redeployed* engine — not our deployment model.
- **Tribeless allowed** (user refinement, supersedes an earlier "≥1 tribe" thought): `Tribe.None = 0` is a valid state and participates correctly in all bitwise ops, so tribeless cards/minions need zero special-casing. No load-time non-empty validation.
- **Tribe-granting model** (user asked to design it): three separate fields on `MinionOnBoard`, mirroring the keyword model but kept apart so aura contributions never pollute persistent state (same discipline as `auraAttackBonus`):
  - `intrinsicTribes` — from definition, copied at summon, immutable, **survives Silence** (silenced Beast is still a Beast).
  - `grantedTribes` — permanent grants ("becomes a Beast"), **Silence clears to `Tribe.None`**.
  - `auraTribes` — from active `IAura`, recomputed each pass, never persisted, drops with the aura.
  - `effectiveTribes = intrinsicTribes | grantedTribes | auraTribes` — what the engine queries. `Card` exposes only intrinsic `tribes: Tribe`; the three-way split is minion-only. All three included now (permanent + aura granting) since `[Flags]` makes the extra field free — no YAGNI deferral needed.
- **Presentation (art / description / sfx) → OUT of `CCG.GameLogic`.** Client resolves from the Platform-API presentation catalog by `definitionKey` (which it already has on every rendered entity), exactly as it resolves art. Preserves the library's "no presentation deps" property; consistent with the borrow list's own "animation/presentation belongs to the client" omission. `rarity` stays on `Card` (plausibly gameplay-relevant — Discover-a-Legendary).

Spec amended: §1 `Card` (+`tribes: Tribe`), §1 `MinionOnBoard` (+`intrinsicTribes`/`grantedTribes`/`auraTribes`/`effectiveTribes`), new §1 **Tribes** subsection (the `[Flags] enum Tribe` + sources + Silence rule) and **Presentation data (out of library scope)** subsection, §3 `IAura` (`AuraEffect.GrantedTribes` + recompute rule).

**Plan impact:**
- **Epic 01 / data-model tickets** — add the `[Flags] enum Tribe`; add `tribes: Tribe` to `Card`; add the four tribe fields to `MinionOnBoard`; `effectiveTribes` as a computed property.
- **Epic 04 (summon) ticket** — initialize `intrinsicTribes` from `card.tribes` at summon; `grantedTribes`/`auraTribes` start `Tribe.None`.
- **Epic 08 / Silence ticket** — Silence clears `grantedTribes` (→ `None`), leaves `intrinsicTribes`. Add to the existing "Silence clears enchantments/grantedKeywords" behavior.
- **Aura ticket (`IAura`/recalc)** — `AuraEffect` gains `GrantedTribes`; recalc rewrites each target's `auraTribes` as the OR of incoming grants (alongside the attack/health bonus rewrite). Add a scenario test: tribal aura grants `Beast`, drops when the aura source leaves; Silence does not remove the *intrinsic* tribe.
- **No library work for art/description/sfx** — Platform-API catalog + client concern; out of scope here.

#### Follow-on amendment (2026-06-01): keywords expanded to the same 4-field model

Prompted by a user consistency question (tribes got 4 fields — `intrinsic`/`granted`/`aura`/`effective` — why do keywords have only 2?). On inspection the asymmetry hid a **real bug from Item 4**: that decision routed aura-granted `spell_damage:X` into `grantedKeywords`, but `grantedKeywords` is the **permanent, RetainEnchantments-bounce-retained** set — so an aura grant would wrongly survive a bounce, and it contradicted `grantedKeywords`'s own "non-aura" definition.

- **`MinionOnBoard` keywords now mirror tribes:** `intrinsicKeywords` (from def, seeded at summon, Silence clears) + `grantedKeywords` (permanent non-aura grants, bounce-retained, Silence clears, **aura grants NOT here**) + `auraKeywords` (aura-granted, recomputed each pass, never persisted/retained) → `keywords` (maintained effective union, IKeyword-resolved, what the engine queries).
- **Silence asymmetry vs tribes (deliberate, documented):** keywords — Silence clears `intrinsic` **and** `granted` (a silenced Taunt minion loses Taunt); tribes — Silence clears `granted` only (silenced Beast is still a Beast). `auraKeywords`/`auraTribes` untouched by Silence (they recompute).
- **`AuraEffect` gained `GrantedKeywords` (declarative-only)** alongside `GrantedTribes`; recalc rewrites `auraKeywords` from it.
- **Scope lever = declarative-only aura keywords.** Active keywords (listener-backed: Lifesteal/Poisonous/Enrage/Windfury/Freeze) cannot be aura-granted, because subscribing during recalc (⑥) would violate **Item 3's invariant** ("all subscriptions inside action handlers ④"). Declarative aura grants (Taunt, `spell_damage`, …) need no listener, so recalc stays a side-effect-free list rewrite. Active keywords still work via `intrinsic`/`granted` (their OnApplied/OnRemoved fire in ④ handlers). Aura-granted *active* keywords → **Unaddressed Features** (see below).
- **Item 4 correction applied:** aura `spell_damage:X` now flows into `auraKeywords`, not `grantedKeywords`. (Marked at Item 4's decision block.)

**New convention — "Unaddressed Features" registry:** added a top-level section to the spec for features the architecture deliberately does not support and that are deferred **indefinitely** (distinct from "deferred to a future epic" items like `GetLegalActions` or the Modifier System, which have a planned home). First entry: **aura-granted active keywords** (why: Item 3 subscription-timing invariant; cost to enable: deferred-subscription mechanism + reopening Item 3). Use this section for future indefinite deferrals.

**Plan impact (additions to Item 8's list):**
- **Epic 01 / `MinionOnBoard` ticket** — add `intrinsicKeywords` + `auraKeywords`; `keywords` becomes the maintained effective union; restate `grantedKeywords` as permanent/bounce-retained only.
- **Epic 08 / Silence ticket** — clear `intrinsicKeywords` + `grantedKeywords`; leave `auraKeywords`.
- **Epic 08 / bounce (T16.3) ticket** — retain `grantedKeywords` only; `auraKeywords` vanish on transition (already noted in §1 transition table).
- **Aura ticket** — `AuraEffect.GrantedKeywords` (declarative-only); recalc rewrites `auraKeywords`; assert active keywords are rejected/ignored from aura grants.
- **Epic 08 / Spell Damage** — aura path targets `auraKeywords` (Item 4 correction).

#### Follow-on amendment (2026-06-02): active/declarative split COLLAPSED — keyword model unified, both Unaddressed-Features walls dissolved

Prompted by a Fireplace-comparison question (where does our spec diverge from the open-source HS clone on shared subsystems?). The comparison exposed that **Fireplace has no active/declarative keyword split** — every keyword is a declarative tag whose behaviour is read *inline* in the relevant action (e.g. `Damage.do` handles lifesteal/poisonous), with **no event-bus listeners**. Reviewing our "active" keywords against that, **every one reduces to a read/hook at the minion's *own* action moment**, so the listener category — and both walls it produced — is eliminable:

| "Active" keyword | Own-moment | Becomes |
|---|---|---|
| Lifesteal / Poisonous | my deal-damage | role-interface hook `IOnDealtDamage`, invoked by `DealDamageAction` over the source's **effective** keywords |
| Windfury | my attack-eligibility | declarative read (`windfury ? 2 : 1`) |
| Enrage | my stat recompute | conditional buff in the stage-⑥ pass (driven by `isDamaged`) |
| Freeze | end-of-turn | already a turn-lifecycle sweep — no per-minion listener |

- **The real cut isn't active-vs-declarative; it's keyword-vs-trigger.** **Keyword** = behaviour at the minion's own action moments (pulled from declarative effective state by the relevant ④ handler). **Trigger** (`ITrigger`) = reaction to board-wide events (bus subscription + epoch filter). The `IEventBus` epoch machinery stays, now used *only* by `ITrigger`.
- **Scattering avoided via role interfaces.** A keyword that enqueues a follow-on action implements a hook interface (`IOnDealtDamage`, …); the owning handler does `EffectiveKeywords(source).OfType<IOnDealtDamage>()` — new keyword = new class, handler iterates by interface, never a central switch (unlike Fireplace's `Damage.do`). Cohesion + open-closed preserved, zero subscriptions.
- **Both Unaddressed Features dissolve.** *Aura-granted active keywords*: gone entirely — hooks read the effective view (which includes `auraKeywords`), so aura-granted Lifesteal just works; aura grants are now **unrestricted** (the `AuraEffect.GrantedKeywords` "declarative-only" constraint is dropped). *Active keywords on a dead source*: reframed (not a listener-lifecycle artifact; a dead source is off-board so `EffectiveKeywords` is empty) — kept as the one remaining Unaddressed entry, enable-path = a death-snapshot keyword fallback.
- **`IKeyword` interface change:** drop `OnApplied`/`OnRemoved` (no subscribe). `IKeyword { KeywordId }` + opt-in role hooks. Reverses the prior bullet's "active keywords cannot be aura-granted" and the §3 active-keyword listener table.

**Spec amended (2026-06-02):** §3 `IKeyword` rewritten (declarative markers + inline-read table + role-hook model); §3 `IEventBus` invariant (keywords don't subscribe; bus = `ITrigger` only); §3 Source Attribution dead-source note (effective-view framing); Unaddressed Features (aura-grant entry removed, dead-source entry reframed).

**Plan impact (supersedes the active-keyword bits above):**
- **Aura ticket** — drop "assert active keywords rejected from aura grants"; `AuraEffect.GrantedKeywords` accepts **any** keyword; recalc rewrites `auraKeywords` for all.
- **Epic 07 (active keywords)** — re-scope from "register listeners on summon/apply" to: implement Lifesteal/Poisonous as `IOnDealtDamage` hooks invoked by `DealDamageAction`; Windfury as a declarative attack-eligibility read; Enrage as a stage-⑥ recompute participant; Freeze via the existing turn-lifecycle sweep. No `IKeyword.OnApplied`/`OnRemoved`, no bus subscriptions.
- **Epic 01 / `IKeyword`** — `IKeyword { KeywordId }` + role-hook interfaces (`IOnDealtDamage` for v1); add new hook interfaces as cards require new own-moment points.
- **Test scenarios** — aura-granted Lifesteal heals; aura-granted Poisonous destroys; aura loss removes both immediately (recompute); dead-source Deathrattle damage does NOT lifesteal (effective view empty).

---

## Item 9 — `PlayContext → EffectContext` conversion / nested contexts

**What the old project does:**
`PlayContext` is the higher-level context held by `PlayCardResolver` (caster, target, services). When an effect needs to fire, `PlayContext.ToEffectContext()` (PlayContext.cs:44–45) produces a lighter `EffectContext` for the op chain. Clean separation: high-level resolvers don't pollute low-level ops with all their fields.

**Why it might matter for our spec:**
Our spec defines `EffectContext` in §3 (`sourceCardId, sourcePlayerId, targetId?, read-only GameState`). It doesn't define how this is constructed inside an action handler, or how nested effects inherit context. A trigger that fires during another effect's cascade — what's its `EffectContext.sourcePlayerId`? The original caster, the trigger source's owner, or the active player? Not specified.

**Recommendation: Adopt as a clarification.** Add to §3 a paragraph: *"Each action handler constructs an `EffectContext` from its action's fields. Triggers fired during an action's event cascade construct their own `EffectContext` from the trigger source minion (not the original action's caster). Nested effects fired by triggers always identify themselves as the trigger source, not the originating action."* This determines who gets credited (e.g., for stats), who gets the spell-damage bonus, and so on.

**Open questions:**
- This decision affects "if Yeti's deathrattle damages a minion, who is the *source* of that damage — Yeti, or the spell that killed Yeti?" Real gameplay implications (poisonous, lifesteal attribution). Recommend: Yeti.

### ✅ DECISION (2026-06-01): ADOPTED as a clarification — attribution rule locked; two latent gaps fixed

The attribution *plumbing* already existed (every action + the key events carry `sourceId`; Item 2's conditions key off the trigger host's `sourceId`). So this item is the **rule** + two fixes it surfaced. Four decisions:

1. **Source attribution rule:** `sourceId` is the entity *whose effect this is*, set by **whoever enqueues the action**, and **never inherited from the upstream cause**. A trigger's `OnFire` stamps `sourceId = its host entity`. Played card → the card; minion's triggered effect → that minion; hero power/weapon → itself. `sourcePlayerId` = the source's controller, captured at enqueue (from death snapshot if already dead). **Yeti's deathrattle damage is sourced to Yeti**, not the spell that killed it (resolves the open question). No "original caster" inheritance.
2. **Generalize `EffectContext.sourceCardId` → `sourceId`** (unified entity ref, the one every action/event already uses) + keep `sourcePlayerId`. The old `sourceCardId` couldn't express a minion-sourced effect (a deathrattle has no card). Relies only on the session-unique-id assumption `targetId` already makes.
3. **Fix the Lifesteal listener wording:** it read "`DamageTakenEvent` on self" (= *taking* damage — backwards); corrected to **caused by self** (`evt.sourceId == this minion`), mirroring Poisonous. Attribution is the field both filter on.
4. **Accept dead-source active keywords don't fire:** a dead minion's deathrattle damage is still *attributed* to it (watchers/friend-enemy resolve correctly), but its Lifesteal/Poisonous don't trigger — those are listener-backed and the listeners were unsubscribed at Phase 1, before the deathrattle resolves in Phase 2. **Key distinction recorded: attribution (data on the event) ≠ keyword application (live listeners).** Falls out of the active-keyword model with no special-casing; logged to **Unaddressed Features** (snapshot-based application is the deferred path).

Spell-damage stays consistent for free: a spell's `DealDamageAction` has `sourceId` = the spell card (cast-time bonus applies); non-spell source → `0` (Item 4, already structural).

Spec amended: §3 `IEffect` (`EffectContext.sourceId` + new **Source Attribution** subsection), §3 Active-keyword table (Lifesteal corrected), **Unaddressed Features** (new entry: active keywords on a dead source).

**Plan impact:**
- **Epic 01 / `EffectContext` ticket** — field is `sourceId` (not `sourceCardId`); `EffectContext` built from the action's own source fields.
- **Action-enqueue sites (Epics 08 triggers, effects)** — every enqueuer stamps `sourceId` = its host/source entity; trigger `OnFire` uses the host, never the upstream action's source. Add a scenario test: spell kills Yeti → Yeti's deathrattle `DealDamageAction.sourceId == Yeti`; a "when a friendly minion deals damage" watcher credits Yeti.
- **Epic 08 / Lifesteal ticket** — listener filters on `evt.sourceId == self` (caused-by-self), not self-as-target. Add a test: Lifesteal minion attacking heals owner; the minion *taking* damage does not.
- **Epic 08 / Lifesteal+Poisonous + death** — test that a dead minion's deathrattle damage does NOT lifesteal/poison (listeners gone), but the event's `sourceId` still names it.

---

## Item 10 — `IEffect` op `Results` ledger as a debugging / tooling target

**What the old project does:**
Not present. Effects fire and forget.

**Why it might matter for our spec:**
If §3 `IEffect.Execute` writes nothing observable beyond events, a "why did this card not fire its second effect" debug session is a search through the event log. A small ledger — one entry per op invocation with `{opType, succeeded, summary}` — would massively shorten that loop. Same ledger feeds tooling (replay viewer, card-design preview).

**Recommendation: Keep on radar; defer.** Don't bake it into the v1 spec; the event log is sufficient for v1 cards. Revisit when we hit a card whose debugging would have benefited.

**Open questions:** none for now.

### ✅ DECISION (2026-06-01): REJECTED for v1 — both halves; the need is served by existing mechanisms, nothing is withheld

(The recommendation's "defer / keep on radar" was incoherent on review — "deferred" and "rejected" can't both hold, and nothing is actually being withheld. Resolved cleanly as a reject. This also sharpens Item 4's loose "consistent with Item 10, already deferred" reference: read it as *rejected*.)

- **(a) Gameplay op-result chaining** — rejected in Item 4, redirected to the event bus (`DamageTakenEvent` + an `ITrigger`). Need fully met.
- **(b) Observability/tooling ledger** — also rejected. Debugging "why didn't the second effect fire?" is served by deterministically *replaying* the action with verbose tracing (Item 6 makes any action re-runnable; `StabilizationAbortReport` carries the repro inputs); a replay-viewer/card-preview tool would consume the event log, not a per-op ledger. No capability withheld.
- **Why reject, not Unaddressed Feature:** that registry is for architectural can't-dos held open with a cost-to-enable (e.g. aura-granted active keywords vs the Item 3 invariant). This is the opposite — trivially *additive* anytime (`IEffect.Execute` stays `void`; just a cross-cutting tracer the engine could wrap around effect execution) and not *needed*. No blocker, no withheld capability → registering it would be noise. Built as ordinary additive tooling if a concrete need ever appears that replay can't serve.

**Disposition test applied going forward:** need met by an existing mechanism / additive anytime → **reject** (decision note only); architecture can't do it without reopening a locked decision → **Unaddressed Features** (cost-to-enable + revisit trigger); real capability, no home yet but planned → **future-epic deferral**. Item 10 is the first.

Spec amended: none. **Plan impact:** none.

---

## Item 11 — Game engine builder pattern

**What the old project does:**
`GameBootstrapper` (GameBootstrapper.cs:32–70) wires up everything manually inside one MonoBehaviour. Order matters; services are constructed inside `GameEngine` constructor. Works fine for one-shot setup.

**Why it might matter for our spec:**
For tests, the scenario builder will need to construct an engine in many slightly different configurations (different starting mana, different RNG seeds, mocked aura services, custom card libraries). A small builder makes this readable.

**Recommendation: Keep on radar; let the test framework drive it.** Don't put a builder in the spec — that's an implementation detail. The scenario builder helpers from §11 of our plan (T1.7) will produce one organically.

**Open questions:** none.

### ✅ DECISION (2026-06-01): NOT A SPEC CONCERN — implementation detail; home = the plan's scenario builder (T1.7)

- A builder/bootstrap is construction convenience, not a behavioral contract. By the Item 10 disposition test it's the "additive / met elsewhere" bucket → stays out of the spec. Recommendation adopted as-is; no open questions.
- **Two construction paths, neither a spec concern:** *Production* — engine init seeds `rngSeed`, then runs setup as system actions (shuffle via Fisher-Yates over `IRandom`, coin flip, opening deal — §3 `IRandom` / Item 6); wiring/DI is a Game Server concern. *Tests* — the T1.7 scenario builder constructs arbitrary `GameState` directly for the Build step, bypassing init to reach specific mid-game states.

Spec amended: none.

**Plan impact:** T1.7's scenario builder grew over this pass — it must now set/handle `rngSeed` + `currentActionEpoch` (Items 6/3) and deterministic seed injection, optional deck-setup vs direct board construction, and the new per-minion fields: `intrinsicTribes`/`grantedTribes`/`auraTribes` (Item 8) and the 4-field keywords `intrinsicKeywords`/`grantedKeywords`/`auraKeywords` (Item 8 follow-on). Flag these in T1.7 so the builder exposes them ergonomically.

---

## Item 12 — Targeting strategy registry vs. switch

**What the old project does:**
`EffectResolver.Resolve` (EffectResolver.cs:7–21) switches on `TargetingRule` enum to call hardcoded `TargetingService` methods. Adding a new targeting rule requires editing both the enum and the resolver.

**Why it might matter for our spec:**
Our spec doesn't say how targeting is structured. The locked feature list includes mid-effect targeting (an interactive pause), `Discover`, and likely random/multi-target effects. The old project's enum-switch will not scale — every new rule edits the central switch and the central service. A `Dictionary<TargetingRule, ITargetSelector>` (or `Func<EffectContext, EntityId[]>`) would let each rule live in its own file.

**Recommendation: Adapt.** Add to §3 an `ITargetSelector` interface:
```
interface ITargetSelector {
    string SelectorId { get; }
    TargetSelection Select(EffectContext context); // returns 0..N targets, or "needs choice"
}
```
`TargetSelection` can be `Immediate(EntityId[])` or `RequiresChoice(StartChoiceAction)`. The choice path threads into the existing `PendingChoice` mechanism.

**Open questions:**
- Selectors that need RNG access — do they receive `IRandom` via `EffectContext`?
- Should multi-target selectors return an ordered or unordered list?

### ✅ DECISION (2026-06-02): ADOPTED — one pure selector primitive, dual of `ITriggerCondition`; random draw stays at stage ④

Targeting was genuinely unspecified in **two** different places, and the decision unifies them under a single primitive rather than the note's narrower "registry replacing a switch":

1. **Player-chosen targets** — the §4 ③ validator's *"is a legal target type for this card"* was a hand-wave with no backing model.
2. **Effect-computed targets** — "all enemy minions", "random enemy", "adjacent" — the effect must *produce* a set the player never supplied (Item 6 parked where the *random* ones resolve).

**`ITargetSelector` adopted in spec §3** (placed immediately after `ITriggerCondition`, as its dual — a condition filters *events*, a selector produces *entities*):

```csharp
interface ITargetSelector { EntityId[] Select(EffectContext context); }  // pure, ordered, 0..N
```

- **Pure function of `GameState`** — no mutation, no RNG, no enqueue. Mirrors the condition library's shape: parameterless singletons (`AllEnemyMinions`, `EnemyHero`, `Self`, …) + parameterized factories (`AdjacentTo(id)`, `MinionsWithTribe(Tribe)`, `Union(…)`, `Filter(selector, ITriggerCondition)` — reusing the condition library as an entity predicate).
- **Ordered** (note's open Q resolved → ordered): reuses the **canonical board order already locked for §4 ⑦** (current player L→R → opponent → neutral), because sequential application is order-sensitive and determinism needs a fixed order. No second ordering invented.
- **Three consumption modes** cover every locked targeting feature with no new mechanism: **(a) auto-hit** (AoE — one concrete action per entity), **(b) random-K** (effect enqueues a single *pool-carrying* action; the **stage-④ handler draws K** with that action's epoch-derived `IRandom`), **(c) player-choice** (effect issues the existing `StartChoiceAction` with `options =` candidate set → `PendingChoice`; selector *feeds* the choice, does not subsume it).
- **Player-target legality unified into the same primitive** — a card declares `(selector, cardinality)`; §4 ③ validity check becomes `chosen ∈ selector.Select(context)` (+ cardinality). The *same* `AllEnemyMinions` an AoE hits-all with is what a single-target spell uses to define legal targets → client highlighting, validation, and resolution can never drift. **No separate `TargetRequirement` taxonomy** (the leaner "more structure / fewer concepts" outcome — parallels §4 ③ making the validator the single source of truth for action legality).

**Open question #1 resolved (RNG / the Item-6 parked question) → Fork A: selectors are pure and do NOT receive `IRandom`.** Random target resolution is a stage-④ concern like every other roll: the effect computes the pool purely; the handler draws K at ④. This keeps Item 6's "RNG consumed only at ④" invariant **verbatim**, keeps each draw attributable to one action's epoch (clean for the `StabilizationAbortReport`), and keeps selectors trivially unit-testable (no RNG mock). The leaner Fork B (`IRandom` in `EffectContext`, selector rolls end-to-end at ⑤) was rejected — it would have relaxed the Item-6 invariant for marginal call-site brevity.

**Note's `TargetSelection = Immediate | RequiresChoice` union — rejected.** Player-choice stays entirely in the existing `PendingChoice`/`StartChoiceAction` mechanism; folding a `RequiresChoice` variant into the selector return type would blur "compute targets" with "interrupt for player input." A selector only ever returns `EntityId[]`.

**Spec amendments:** §3 new `### ITargetSelector` section; §3 extensibility-layer interface list gains `ITargetSelector`; §4 ③ target-validity bullet rewritten to selector membership; §3 `IRandom` note (the Item-6 deferral) rewritten to point here. The illustrative §2B action catalog is left as-is (concrete pool-carrying actions like `DealDamageToRandomAction` are plan/implementation detail).

**Plan impact:**
- **New Epic 08 ticket (`ITargetSelector` library)** — alongside the already-flagged T8.10 `ITriggerCondition` library; same shape (singletons + factories + `Filter` reusing conditions), pure, ordered by §4 ⑦ board order. Scenario tests: `AllEnemyMinions` ordering, `Filter(AllFriendlyMinions, IsDamaged)`, `AdjacentTo`, `Union`, empty-set.
- **Effect/action tickets (Epic ~07/08)** — random-K effects use the **pool-carrying-action + stage-④ draw** pattern (one action per random group, not one per pre-rolled target). Add a `DealDamageToRandomAction`-style handler test that asserts the draw is reproducible from `(seed, epoch)`.
- **Validation ticket (Epic ~05/06, §4 ③)** — `InvalidTarget` check implemented as selector membership; the *same* selector serves the future `GetLegalActions` bot enumerator (Item 5) and client preview. The standalone validator predicate (Item 5) now depends on the selector library.
- **`PlayCardAction`/`UseHeroPowerAction` card definitions** carry a `(selector, cardinality)` target requirement; reconcile with whatever effect-definition ticket owns the card `definition` JSON schema.

---

## Item 13 — Command-log replay vs. event-log replay

**What the old project does:**
Neither is implemented (both `GameSnapshot.cs` and `EventLog.cs` are stubs). The investigation flagged this as "design fresh." Worth choosing the strategy in the spec.

**Why it might matter for our spec:**
The plan's Epic 16 backlog mentions "Event-log replay — reconstruct GameState from the event stream." That's only sound if every state change has a corresponding event (currently true). The alternative — replaying the command/action log — is also sound *if* the engine is deterministic (which requires Item 6 above to be done).

**Recommendation: Keep on radar; decide before implementing Epic 16 backlog item.** Pre-decision: **command-log replay is simpler and faster** because the event log can be regenerated from the command log + seed; the reverse is not true (events don't encode random rolls). Probably we want both: command-log for canonical replay, event-log as the wire format to clients.

**Open questions:**
- Single sequence per game, or per-player streams? Single is simpler.

### ✅ DECISION (2026-06-02): ADOPTED — confirms Item 6's model + tightens the spec's Replay paragraph with three gaps

The core was **already locked by Item 6** (spec §3 `IRandom` Replay paragraph + determinism contract). Item 13 confirms it and closes three gaps a review of that paragraph against the locked feature set surfaced — two of them genuine **correctness** fixes, not nits. **Spec §3 Replay paragraph rewritten** (the disposition is *tighten spec now*, not plan-only) to state the engine as a deterministic function of `(rngSeed, ruleset version, decklists, ordered input log)`, with four clarifications:

1. **An *input* = anything submitted via `Submit`, including timeout / forfeit / disconnect-injected actions** (auto-`EndTurn`, auto-skip, auto-resolve a `PendingChoice`). These are *external nondeterminism* (wall-clock-dependent, not state-derivable) so they must be logged like any other action — Item 6's "player actions only" wording was subtly wrong the first time a turn times out. **The timer that injects them is a Game Server concern**; GameLogic only requires they enter via the same `Submit` seam and are logged. *(Correctness fix.)*
2. **Ruleset-version pinning** — a command-log replay re-runs engine logic + card `definition` JSON, so it is valid only against the **same engine + card-definition version** (the classic command-log-replay-fragility-across-patches problem; cf. StarCraft/AoE). The replay package carries a version stamp. *(Correctness fix — was entirely unstated.)*
3. **Command log vs. event log — two artifacts, two jobs** (the note's actual question, now with the *why*): **command/input log = canonical** (compact, re-simulatable, source of truth, but version-fragile); **event log = client wire format + spectator/late-join stream + version-robust archive** (self-contained deltas, no game logic needed to render, but *not* re-simulatable). Asymmetry: **event log is derivable from command log + seed, never the reverse** (events record outcomes, not the inputs/rolls). So we keep both — exactly the note's pre-decision.
4. **Single, total-order input stream** (note's open question resolved) — one sequence per game, not per-player; per-player streams would reintroduce interleaving ambiguity.

No rejection, nothing for Unaddressed Features. Disposition test outcome: **SPEC (tighten) + Plan-impact**.

**Plan impact:**
- **Epic 16 backlog** — the existing "Event-log replay" item stays as the **client-facing / spectator** path; **add a command/input-log canonical-replay item** alongside it (compact replay package = `rngSeed` + ruleset version + decklists + input log). Note the asymmetry (event log derivable from command log, not vice versa) so they aren't built as redundant.
- **Input-log capture** — wherever `Submit` lives (Epic 01 engine ticket), note that the canonical input log records **every** `Submit`-ed action including timeout/forfeit/disconnect-injected ones; do not log system actions, rolls, or epochs (they regenerate). This is the same log the abort report's `triggeringAction` is drawn from.
- **Ruleset-version stamp** — the replay package and (later) the Game Server's match record carry an engine + card-definition version; flag for the Game Server spec (timer/timeout injection + version stamping are its concerns).
- **Determinism audit ticket** (Epic 16 "Determinism/RNG audit") — extend its checklist to assert *no wall-clock / no ambient nondeterminism in handlers* (already the §3 `IRandom` contract) so command-log replay stays bit-exact.

**This completes the borrow-list pass (Items 1–13 all resolved). Next: the end-of-pass plan reconciliation below, then implementation at Epic 01 / T1.1.**

---

## Fireplace-comparison amendments (post-pass, 2026-06-02+)

Post-pass refinements driven by comparing our spec against **jleclanche/fireplace** on shared subsystems (see the `reference-fireplace-comparison` memory). These are **spec amendments beyond the 13 items** — the spec is not frozen. Each is a distinct shared-subsystem comparison point, not tied to a borrow-list item.

> The first such amendment — the **keyword-model collapse (2026-06-02)** — is recorded under **Item 8 → "Follow-on amendment (2026-06-02): active/declarative split COLLAPSED"** above, since it grew out of the Item 8 keyword discussion. The amendments below are standalone.

### Comparison point B — death-resolution cadence: per-action → cascade-settle (Hearthstone-faithful)

**The design question (the framing the decision was made under):**

> **Is "the moment of death" something the game's mechanics get to play with — or is death an instantaneous bookkeeping fact?**

Our §4 pipeline runs **⑥ aura recalc + ⑦ death resolution per action**, with the invariant *"next action not started until death check runs to stability."* Fireplace (and Hearthstone) instead settle deaths **once, when the whole triggered cascade drains** (`_action_stack` empty → refresh auras + `process_deaths`). The behavioural difference is purely *ordering*: in our model a minion that hits 0 HP is **removed immediately (⑦)**, before same-event triggered reactions are dequeued (⑨); in HS it **lingers in a "mortally wounded" / pending-death state** — still on the board, still counted, still emitting auras, still able to trigger and be referenced — until the sequence settles, *after* those reactions resolve.

Worked example that makes it concrete — I cast "deal 1 to minion M" (a 1/1, lethal); opponent's minion N triggers *"when a friendly minion takes damage, deal damage to the enemy hero equal to the number of minions I control"*:
- **HS / Fireplace:** damage → N's trigger resolves while M is still on board → counts **2** (M + N) → I take **2**. Then the death step removes M.
- **Our spec (per-action ⑦):** damage → M to 0 → ⑦ removes M → ⑨ N's reaction runs against a board of just N → counts **1** → I take **1**.

Same board, same cards → **2 vs 1 damage**, because in our model the dying minion has already vanished before the same-event reaction reads the board.

**✅ DECISION (2026-06-03): ADOPT Option 2 — cascade-settle (Hearthstone-faithful) death cadence.**

Decided as a **game-design** call, not an engineering one (it's a pre-implementation spec edit — no code to refactor; cost is spec coherence only). The deciding factor is the *design space* each cadence opens:

- **Option 2 introduces a "mortally wounded" window** — a 0-HP-but-not-yet-removed minion — which is a designable beat. It unlocks the entire **"dying matters"** mechanic family: **overkill / excess-damage**, **retaliation / dying-swinging** (a doomed minion gets a last act from the board), **simultaneous-death synergies** (minions that trade both resolve as present; "died at the same time" deathrattles), **count-the-doomed** (board-counting effects see the body about to die), and **last aura pulse** (a dying buffer still buffs the same-event reaction).
- **Option 2 is nearly a superset of Option 1.** Any effect can opt back into instant-death semantics by filtering `currentHealth > 0`; the reverse is impossible — instant death cannot recover the dying-window beat. The price is a recurring per-card *"does this see the doomed minion?"* decision (cognitive load + bug surface), which Option 1 settles once, globally.
- **Option 1 (instant death) is a coherent alternative we rejected** for this game: always-true board between actions, adaptive "machine-gun" sequential effects that never waste a hit on the already-dead, maximum legibility, cleaner spectating/analysis. Good for a tighter game; but it forecloses the dying-matters family entirely.

The decision was **"without a doubt" Option 2** because two of this game's *signature* mechanics interact natively with the dying-window (see references below).

**Mechanism (how Option 2 is realized):**
- We do **not** adopt Fireplace's literal action-*stack*. Our **FIFO trigger-reaction queue already matches Hearthstone's trigger-queue semantics** (triggers enqueue and resolve in order). The only structural change is *where the death step fires*.
- **Move ⑦ death + ⑧ win from per-action to "when the action queue drains"** (cascade settled), then loop the existing death wave (Phase 1/2/3, `MaxDeathWaves=16` cap unchanged — Item 1). Stages ④⑤⑥ still run per action while draining.
- **New entity state: "pending death"** — a minion at `currentHealth ≤ 0` that remains on the board until the settle point. This single concept is the source of every downstream ripple ("what does X see when a minion is mid-death?").
- **New rule:** resolve deaths **before halting for a `PendingChoice`/`PendingIntervention`** — you never pick a target (or open an intervention window) while lethally-damaged minions linger, except where the intervention is *itself* keyed to the dying window (see reference R1).

**📌 Design-avenue references (recorded for future card design — these are *why* Option 2 won):**

- **R1 — Intervention × lethal.** The locked single-response **Intervention** window can fire *inside* the pending-death window: *"in response to your minion taking lethal damage, you may…"* — a **save-from-lethal** interaction. Under instant death there is no such moment; the minion is already in the graveyard before anyone could respond. Option 2 makes "rescue from death" a playable beat. (Requires the pre-halt death rule above to carve out an exception for dying-window-keyed interventions.)
- **R2 — Inversion × lethal.** The **Inversion** mechanic (Attack↔Health swap) can be played *in response to lethal*: a minion at 0 health flips its former Attack into Health and **survives**. A genuinely *ours-not-Hearthstone's* mechanic that exists **only** if a dying minion lingers long enough to be inverted. (Exact stat/damage-carry math is an inversion-semantics detail to pin when the card is designed; the *avenue* is what Option 2 unlocks.)

**Disposition: SPEC (§3 selectors applied; §4 cadence rewrite unblocked) + Plan-impact.**

- **✅ SUB-DECISION RESOLVED (2026-06-03) — pending-death targetability, via an `All… / Alive… / Dead…` selector trichotomy (APPLIED to spec §3 `ITargetSelector`).** Rather than make "alive" a single global property, the targeting library exposes the distinction as explicit selector families that **partition the on-board minions**, `All… = Alive… ∪ Dead…` (disjoint):
  - `AllEnemyMinions` / `AllFriendlyMinions` / `AllNeutralMinions` / `AllMinions` — **include** mortally-wounded (HS-faithful AoE / board-count).
  - `AliveEnemyMinions` / `AliveFriendlyMinions` / `AliveNeutralMinions` / `AliveMinions` — `currentHealth > 0` only (explicit opt-out from the dying window).
  - `DeadEnemyMinions` / `DeadFriendlyMinions` / `DeadNeutralMinions` / `DeadMinions` — the mortally-wounded only (`currentHealth ≤ 0`, on board, pending removal) — the natural home for an **R1 Intervention × lethal** save/finisher.

  This resolves "what does a selector see during the pending-death window" *per card* (the designer picks the family) and makes the §4 ③ validator's target-legality well-defined for each (membership in the declared selector's output). Also added a plain `AllNeutralMinions` for symmetry, and recorded that **graveyard ("died this turn" / resurrection) selection is a *separate* primitive**, not an `ITargetSelector` (you summon from the graveyard, you don't board-target it). Dovetails cleanly with the pending **Item 12 Fork-A tightening** (random-K carries the *selector*, evaluated against the current board at ④) — a card picks e.g. `AliveEnemyMinions` and the ④ draw naturally reflects the live board. **§4 cadence rewrite is now unblocked.**

**Spec amendment — APPLIED (2026-06-03):**
- **§3** `ITargetSelector` singleton catalog + the `All…/Alive…/Dead…` partition table and graveyard-exclusion note; the **random-K consumption row** retightened per Item 12 Fork-A (carry the *selector reference*, evaluate against the current board at ④, not a frozen pool).
- **§4 intro** rewritten as a **loop** (not "nine stages in strict sequence"): ①–⑥ per action draining the queue; ⑦ death + ⑧ win are **settle stages** firing only when the queue empties; ⑨ is the loop driver. Mortally-wounded framing + the reactions-before-removals consequence stated.
- **§4 ⑥/⑦/⑧/⑨** rewritten accordingly; **Deterministic Ordering** table "action isolation" row → "Resolution cadence" (settle-point batching).
- **§4 `PendingChoice`** gains the pre-halt death rule; **`PendingIntervention`** gains it **with the R1 exception** (don't pre-resolve a death keyed to the dying window) + a forward note to the Secrets/armed-reactive discussion for window-opening + batching.
- **§2B** gains **`MinionMortallyWoundedEvent`** (fires the instant a minion enters pending-death from any cause — damage / aura-loss / destroy-mark — distinct from `MinionDiedEvent`).
- **§1 `MinionOnBoard`** `currentHealth` note gains the pending-death framing; **Unaddressed Features** dead-source-keyword entry now distinguishes *removed* (keywords don't fire) from *mortally-wounded* (still on board, keywords DO fire).

Both queued spec edits (the §4 cadence rewrite **and** Item 12 Fork-A) are now landed; the spec is internally consistent (the earlier "amendment B, §4 ⑦" forward reference from §3 is now satisfied). See the 2026-06-03 R1 follow-up subsection below for the realization detail + deferred design choices routed to point D.

**Plan impact:**
- **Epic 04 / pipeline ticket (`GameEngine` action loop)** — death/win checks move to the queue-drain boundary; the loop is "drain ④⑤⑥ → at empty, run ⑦ wave + ⑧ → loop." Re-verify the epoch model (Item 3) is untouched (epochs are per-action at ④, independent of when ⑦ runs) — confirm only.
- **Epic 04 / `DeathResolutionService` (T4.5)** — wave loop itself unchanged; only its *invocation point* moves. Re-check `StabilizationAbortReport` snapshot semantics (Item 1 / T4.8) — "start of the triggering action" when deaths now span a settled cascade.
- **Epic 01 / `MinionOnBoard`** — add the pending-death state (a 0-HP minion is valid on-board until settle); scenario builders must allow constructing it.
- **Epic 08 / `ITargetSelector` + validator** — implement the `All…/Alive…/Dead…` minion-selector families (12 singletons; `Alive…` = `Filter(All…, hp>0)`, `Dead…` = `Filter(All…, hp≤0)`, partitioning the on-board minions); the same families govern selector output, §4 ③ validity, and client highlighting. Scenario tests: `All` includes a mortally-wounded minion, `Alive` excludes it, `Dead` returns exactly it, `All = Alive ∪ Dead` disjoint.
- **Epic 09 (source attribution, Item 9)** — the "death-snapshot if already dead" rule must cover the new pending-death source window (a source at 0 HP but still on board).
- **Test scenarios** — the 2-vs-1 worked example above as a regression fixture; overkill/retaliation/simultaneous-death/count-the-doomed timing tests; R1 save-from-lethal-intervention and R2 invert-from-lethal as design-validation scenarios when those cards exist.

#### Follow-up (2026-06-03): how an R1 dying-window intervention fits the pipeline

**Question raised:** a held card *"if a friendly minion is mortally wounded during the opponent's turn, you may invert it"* needs to fire *during* the dying window — but at the moment the minion hits 0 HP the action queue may still be non-empty, so the death wave hasn't run. How does the intervention get scheduled in?

**Resolution — the deferred death wave *is* the window; correct ordering is by construction, not by scheduling.** The minion has **not died** — it's *mortally wounded* (on board) and stays that way until the queue drains. So:

> The death wave runs only when the queue is empty. The intervention is an action *in* the queue. Therefore it is dequeued and resolved **before** the queue can empty — i.e. before the death wave — every time.

Trace: opponent AoE drops my **M** to 0 at its ④ → ⑤ a mortal-wounding hook fires → R1 trigger enqueues `StartInterventionAction`. Queue is non-empty, so ⑦ does **not** run. Draining eventually dequeues the intervention → pipeline halts, my window opens with **M still on the board** → I invert M → its health flips `> 0` → it is no longer mortally wounded. Queue finally empties → ⑦ collects `currentHealth ≤ 0` → M is excluded → M lives. (Skip/timeout → M stays at 0 → collected normally.)

**This costs three things in the §4-amendment-B rewrite:**
1. **A mortal-wounding hook at ⑤ — `MinionMortallyWoundedEvent`** — fires the instant a minion enters pending-death **from any cause** (damage to ≤0, aura loss dropping maxHealth ≤0, destroy-mark), **distinct from `MinionDiedEvent`** (death-wave Phase-1 removal). R1 keys off *wounding*, not *died* — that is the entire difference between "save it" and "too late." (A damage-only version could instead reuse `DamageTakenEvent` + an `IsMortallyWounded` condition; the dedicated event is the clean source-agnostic hook and also serves any future "whenever a minion would die" card.)
2. **An R1 exception to the pre-halt death rule.** The rule "resolve deaths before halting for a choice/intervention" (so a *normal* choice sees a clean board) must carve out dying-window-keyed halts: **pre-resolve deaths before a halt *unless the halt is itself keyed to the pending-death window*** — you must not pre-kill the death you're offering to prevent.
3. **Reuse `PendingIntervention` as-is** — `respondingPlayerId` = non-active player, single response (play the card / skip), timeout injects a skip (consistent with Item 13's timeout-as-injected-input). No new phase.

**Design choices surfaced, deliberately deferred (not decided this session):**
- **What opens the window** — a general engine rule (always open on friendly mortal-wounding during opponent's turn) vs. only when the responder holds an **armed/secret-style card**. *Same machinery as Secrets (Fireplace menu point D)* — a held reactive card with a live off-board trigger; R1 is the *player-choice* flavor, a Secret the *auto-resolve* flavor. Likely **one "armed reactive trigger" concept covers both** → fold this into the point-D discussion.
- **Batching** — one AoE wounding several friendlies = one window (save which / save all) or N sequential windows? FIFO yields N unless batched.
- **Scope of "mortally wounded"** — does the hook (and R1's offer) include *marked-for-destruction* (destroy bypasses health, so inversion can't restore positive HP → save fails), or only health-based lethality?
- **R2 stat-math dependency (separate from the pipeline)** — the save only works if inversion leaves `currentHealth > 0` (does inverting clear damage / recompute current HP from the new maxHealth?). The pipeline handles "if >0 at drain, it lives" regardless; whether inversion *achieves* that is the R2 inversion-semantics rule to pin when R2 is designed.

**Disposition:** confirms amendment B's R1 reference is pipeline-realizable; `MinionMortallyWoundedEvent` + the pre-halt R1 exception are now **APPLIED** to §2B / §4 (2026-06-03); the window-opening + batching questions were routed to Fireplace menu point D — **now resolved below** (window-opening: locked; batching: still deferred game-feel).

### Comparison point D — Secrets / armed reactive triggers → one unified reactive-trigger + interception model

**The design question:** what *opens* an intervention window, and are Secrets, the locked Intervention system, and the R1 dying-window three mechanisms or one? Worked through 2026-06-04 across the action/event split, the MTG-stack comparison, the secret catalog, and two simplifications the user drove.

**✅ DECISION (2026-06-04): ONE mechanism over `ITrigger`. APPLIED to spec §3 (new "Reactive Triggers, Interventions & Interception" subsection) + §4 (new stage ③′, PendingIntervention rewrite, ⑧ note, ③ attacker-alive check `AttackerNotAlive`) + §2 (events/action).**

The model, in four locked pieces:

1. **Zone-scoped reactive hosting (aura-like, the user's framing).** An `ITrigger` is live by virtue of its host's zone, registered/unregistered by `ICardHandler` on zone entry/exit — the *exact* lifecycle a board minion's triggers already use, run on **other zones**: **hand** (→ a player-choice **intervention** window) and the deferred **secret zone** (→ **auto**-resolve). This *answers "what opens the window"*: a card in hand whose live reactive trigger matched — never a general engine rule; precise + inspectable. It also **harmonizes** the locked "play one card / skip" literally (the window only offers a card *designed* reactive) instead of revising it. Obeys every invariant unchanged: registration inside ④ → `birthEpoch` → epoch filter stops self-reaction; bus still carries only `ITrigger`.

2. **Two hook phases = the whole taxonomy** (collapsed from my initial three — the user's first simplification). Every action has a **pre** phase (`ActionDeclaredEvent`, new stage ③′, *before* ④) → **interception** (prevent/alter); and **post** events → plain **reaction**. The mortally-wounded events (`MinionMortallyWoundedEvent` + new **`HeroMortallyWoundedEvent`**) are *post*-reactions whose before-finalization timing is a free gift of the ⑦/⑧ settle deferral — **"consequence-deferral" is not a third kind**, just a post-reaction exploiting the cadence.

3. **Interception = ordinary effects + re-resolution; NO disposition vocabulary** (the user's second, decisive simplification). The response runs normal effects (grant Divine Shield / Poisonous / armor, summon a Taunt, destroy/return the attacker, …); the held action then **re-validates (②③)** and most "alterations" emerge for free (shield absorbs, poison retaliates, killed/returned attacker or new Taunt → fizzle/redirect by the ordinary rules). The *entire* irreducible surface that state-mutation can't express is **two ops on the held action: `cancel`** (Counterspell) **and `retarget`-in-place** (Misdirection/Spellbender). Completeness rests on one invariant: **anything interceptable is a discrete action** (intrinsic keyword/armor modifiers are *pulled* at resolution, never declared).

4. **Uniform declaration, not a `*Declared` taxonomy** (the user's Q2). Stage ③′ publishes one generic `ActionDeclaredEvent { action }`; triggers filter on the carried action's type/params via the condition library. Fires once per action (resume re-validates, never re-declares → no loops). Depth-1 cap: a response can't itself be intervened (bounds the MTG stack to a single suspend/resume; auto-secrets still chain FIFO).

**Validation — does it capture HS?** Walked the secret catalog: declaration-hold covers Counterspell / traps / redirects; post-reactions cover Mirror Entity / Effigy / Redemption / Eye-for-an-Eye / Avenge; R1 covers the dying-window. **The one class consequence-deferral can't do is damage *prevention* (Ice Block)** — saving *after* lethal registers its side-effects (lifesteal/reflect) before undoing the death. Prevention is an **interception on `DealDamageAction`** = **menu point A (Predamage), now SUBSUMED** into the declaration model (armor / Divine Shield / double-damage / caps / prevent all live at the damage declaration). Only residue of point A = **deterministic modifier precedence** at one damage declaration.

**Batching — RESOLVED (2026-06-04), APPLIED to §3/§4.** A post-reaction player window does **not** halt per sub-event; matches **accumulate** through the cascade into a per-card record `{cardId, responder, matched:[…]}`, and **one** window opens per (reactive card, settle) **before ⑦**, with the **matched set as the window's selector** (∩ the card's `(selector, cardinality)` family; `All/Alive/Dead` decides whether mortally-wounded matches are offered). Worked case: AoE damages 7 friendlies + responder holds a single-use "heal one minion +2 on `DamageTakenEvent`" → **one** window offering the 7 (dying ones included), pick one to heal/save. `SubmitInterventionAction` gained `targetIds[]?` (validated by §4 ③ membership). Settle order locked = **batched windows → ⑦ death → ⑧ win**. Declaration-hold needs no batching (one `ActionDeclaredEvent` per declared action → one window for the whole AoE). Only player-choice *windows* batch; auto reactions (secrets/board triggers) fire inline per-event; a deterministic no-choice reaction ("heal *it* for 2") belongs in the auto/secret flavor and opens no window. **⚠ The cross-cascade-accumulate / open-at-settle timing in this paragraph is SUPERSEDED by the 2026-06-06 follow-up below (per-action windows at ⑥′). The *outcomes* above (AoE → one window; selector = matched set; auto inline) are unchanged — only the *when* moved.**

**Point A (Predamage) — CLOSED (2026-06-04), APPLIED.** Fully subsumed into the declaration/interception model, no separate epic: (1) **all damage routes through `DealDamageAction`**, combat included (`AttackAction` enqueues it, not inline) → the predamage declaration is universal (Ice Block stops combat damage too); (2) **auto-hit AoE = one selector-carrying `DealDamageAction`** (revised the `ITargetSelector` "one action per entity" rule to the one-action-carries-selector pattern already used by Random-K) → one ③′ declaration → **one batched pre-damage window** (targets = selector), the immediate twin of post-reaction settle-batching; (3) **④ damage-modifier precedence pinned**: base(+baked spell-dmg) → ×multipliers → −reductions(floor 0) → cap → immune(→0) → Divine Shield(break if >0) → apply/overkill/mortally-wounded events. Per-target protection in an AoE = grant immune/shield (state, ④ pull), not a new op. Applied to §2A (`DealDamageAction` carries entity|selector), §3 (`ITargetSelector` auto-hit + "Reactive…" point-A close-out), §4 (Batching reconciled).

**Still deferred (game-feel, did not block the lock):** the **Secret auto-resolve flavor** + secret-zone data model + one-per-name/max/own-turn rules; **visibility** (hand-live = hidden "gotcha" vs HS telegraph); **cost timing** (free vs pay-on-response); **marked-for-destruction scope** of dying-window saves; **R2 inversion stat-math**.

**Plan impact:**
- **NEW EPIC (or major epic section): Reactive Triggers & Interventions** — the ③′ declaration stage, `ActionDeclaredEvent` publish + suspend/resume, `cancel`/`retarget` on the held action, re-validate-on-resume/fizzle, depth-1 cap. Tests: Counterspell (cancel), Misdirection/Spellbender (retarget), Explosive/Freezing Trap (state + fizzle on resume), Divine-Shield/Poisonous grant via interception.
- **Epic 01 (data model)** — `PendingIntervention.heldAction` nullable; `GamePhase` already has `PendingIntervention`. Scenario builders must construct a suspended-action state.
- **Epic ~07/keywords + ICardHandler** — extend the trigger-registration lifecycle to the **hand** zone (register on `DrawCard`/`GiveCard`, drop on play/discard/return); `DefaultCardHandler` reads a `reactive` block from the card `definition` (trigger phase + condition + effect).
- **Epic 04 (pipeline)** — insert stage ③′ between ③ and ④; the engine suspend/resume + depth-1 guard; publish `ActionDeclaredEvent` per dispatched action.
- **Epic 02 (events/actions)** — add `ActionDeclaredEvent`, `HeroMortallyWoundedEvent`; `StartInterventionAction.heldAction` nullable; `HeroMortallyWoundedEvent` published by the damage handler when a hero crosses ≤0.
- **Menu point A (Predamage) — CLOSED**; fully subsumed (combat→`DealDamageAction`, AoE = one selector-carrying damage action, ④ precedence pinned). No separate point-A epic; the damage-handler ticket carries the precedence order + combat routing.
- **Test scenarios** — Ice Block (as a damage-declaration interception), simultaneous mutual hero lethality → draw at ⑧, save-the-hero post-reaction.

### Comparison point D follow-up — intervention-window timing made bulletproof (2026-06-06)

**The question:** *when* do the two hook kinds actually fire? Worked through 2026-06-06 (session 4) by tracing a worked example.

**✅ DECISION (2026-06-06): post-reaction windows are PER-ACTION (new stage ⑥′), not cross-cascade-at-settle. APPLIED to spec §3 (pillar 2 + Deterministic-Ordering table) + §4 (new ⑥′ stage, ⑨ rewrite, new "Response resolution" subsection, PendingChoice pre-halt rule shrunk to type-A, PendingIntervention post-reaction block + Batching rewritten) + §2 (`SubmitInterventionAction` row).** Three locked findings:

1. **Pre-hooks (③′) fire immediately, per-action, mid-cascade** — forced, since interception must precede the action's ④. No batching (one action = one `ActionDeclaredEvent`).
2. **Post-hooks fire per-action at a new stage ⑥′** (after each action's ⑤/⑥), batching *that one action's* matching events into one window. This satisfies the AoE motivation (AoE = one action → one window) *without* the over-generalization of accumulating across the whole cascade. **"Deal 1 twice" = two actions → two windows** (single-use card spent on either; skip holds it). The batching unit is **one action's events — never per-event, never per-cascade**.
3. **A window's response resolves IMMEDIATELY** (both kinds), before the engine returns to drain the pre-existing queue. **Load-bearing, not an optimization:** the worked example showed that FIFO-appending the response lets an already-queued `PendingChoice`/declaration-hold pre-settle deaths (⑦) and remove the very minion a save just targeted, before the heal lands. Confirmed: during a response the depth-1 cap suppresses only *player* windows; **board/auto triggers fire inline**.

**Why this dissolves the original contradiction:** the "pre-halt death rule + R1 exception" framing was confusing because it stated R1 as an exception to a rule that should never have covered it. New clean split: the **pre-halt death settle applies to type-A halts only** (`PendingChoice` + declaration-holds — about the player's own *forward* action, where lingering pending-death minions are noise); **post-reaction windows (⑥′) never pre-settle** (they're *about* the just-happened events, so the mortally-wounded subject must stay on board). "R1" is no longer a carve-out — it's just "post-reactions are a different category." The full-cascade model also had a concrete **save-preemption bug** (an unrelated Discover in the same cascade kills the dying minion before the settle window opens); per-action fixes it for free.

**Prior art:** HS secrets fire immediately on the triggering event (per-action/per-event), not batched to end-of-cascade; MTG batches triggers per-event on the stack with state-based actions between. The superseded full-cascade model was the outlier.

**Plan impact (refines point D's list above; no new epics):**
- **Epic 04 (pipeline)** — add stage ⑥′ (post-reaction window) after ⑥, parallel to ③′; the per-action window check + the immediate-response-resolution rule (shared with ③′ resume); ⑨ no longer opens windows at the settle. Tests: AoE-save (one window, dying ones offered), deal-1-twice (two windows, single-use spent on either), the save-preemption regression (save + unrelated Discover in one cascade → save still lands).
- **Reactive Triggers & Interventions epic** — the post-reaction batching is per-action; `SubmitInterventionAction` candidate set = one action's matched subjects.
- No data-model change beyond point D (the per-card `{cardId, responder, matched}` record is now scoped to one action, not the cascade).

### Comparison point D follow-up (cont.) — combat atomicity (2026-06-06)

**The question:** combat routes through `DealDamageAction` and every action declares at ③′ — so is combat **two independent damage actions** (attacker strike + defender retaliation, each its own ③′/⑥′, sequential) or **one atomic combat unit** (one declaration, both hits, one ⑥′)? Three options weighed: **A** two independent actions (uniform), **B** a bespoke atomic `CombatDamageAction`, **C** two actions grouped so both ④s apply before a shared ⑥′ (HS-faithful "apply-both-then-react").

**✅ DECISION (2026-06-06): Option A — two independent `DealDamageAction`s, sequential, per-recipient ③′ + ⑥′. APPLIED to spec §3 (point-A close-out combat bullet) + §2A (`DealDamageAction` row).** Rationale:
- **Only A is consistent with the just-locked ⑥′ rule** ("one window per action; two instances → two windows"). B and C both carve a combat-specific *exception* into a rule we just locked; B also breaks "all damage = `DealDamageAction`" and re-hides per-recipient structure inside the combined action anyway.
- **Per-recipient interception is the *more* faithful model for prevention** — Ice Block / Divine Shield protect one entity, not "a combat."
- **Death simultaneity is preserved regardless** (⑦ settles at queue-drain → both combatants mortally-wounded-on-board → one shared wave). The fork never affected deathrattle/reborn timing.

**Accepted costs (narrow):** combat is not atomic w.r.t. windows — a held card can interleave between the two hits (bounce-the-defender dodges retaliation), and a same-combat reaction sees **per-hit** board state (the defender's reaction fires before retaliation lands), a deliberate divergence from HS apply-both-then-react.

**Payoff that clinched it (mirror of the cost):** A's sequencing means a **mortally-wounded defender's retaliation is computed while it is live-and-mortally-wounded** → **dying-swing retaliation** is supported *by construction* (Poisonous/Lifesteal on a dying defender; "retaliation doubled while mortally wounded"). Base attack snapshotted at `AttackAction` ④; ×modifiers pulled at each damage ④ (precedence step 2). Under B/C this mechanic would **not** fire (the defender isn't mortally wounded yet when its retaliation is computed). So the "divergence" is the feature. Aligns with the spec's existing "dying-swing retaliation" example under Unaddressed-Features (keyword-active while mortally wounded).

**Plan impact:** Epic 04 / combat-damage ticket — `AttackAction` enqueues **two** `DealDamageAction`s (strike + retaliation), base-attack snapshot at `AttackAction` ④, modifiers pulled per damage ④; both share the settle death wave. Tests: simple trade (two declarations, one death wave), dying-swing retaliation (×2 while mortally wounded), bounce-the-defender between hits → retaliation fizzles on re-validation.

---

## Hole-hunting pass (post-Fireplace, 2026-06-07+)

A free-form Q/A sweep for spec gaps, distinct from the 13-item borrow list and the Fireplace points. The user probes; each confirmed gap is closed with a recorded decision + Plan impact.

### Hole #3 — Neutral zone: control / command / graveyard (2026-06-07)

**The gap:** the locked feature scope said players can attack, *command*, and *mind-control* neutral minions, but (a) the §4 ③ validator had no attacker-ownership check at all (a player could swing with an opponent-owned minion — latent bug), (b) no control-change action existed (so even plain Hearthstone enemy mind-control was unmodeled), (c) neutral deaths had no graveyard home, and (d) turn-reset machinery is owner-scoped so a commanded neutral could never reset its attack state. Surfaced as one contradiction, unravelled into a cluster.

**✅ DECISION (2026-06-07): APPLIED to spec §1/§2/§3/§4.** Vocabulary locked: **control** = permanently move a minion into the controller's zone (asleep first turn unless rush/charge), no longer neutral, dies to that player's graveyard; **command** = a one-shot card-granted single attack *this turn*, no zone change, stays neutral. Requirements from the user:

- **Neutrals are inert by default** (Req 2): not commandable/controllable without a card. So the default `AttackAction` legality is simply `attacker.ownerId == submitter` (own minions only) + new code `AttackerNotControlled` — which *also* closes the latent "swing with the opponent's minion" bug. Neutrals are attackable targets, never default attackers.
- **Per-lane Taunt (Req 1 — REVERSES the old "Taunt ignored for neutrals" scope):** the Taunt constraint is scoped to the *target's lane*; a neutral Taunt forces attacks into the neutral lane only, an opponent Taunt the opponent lane only; cross-lane never applies. Shared by `AttackAction` + `CommandAttackAction`.
- **Two actions, not three cards' worth:** `TakeControlAction { minionId, newOwnerId, boardPosition?, sourceId? }` (BoardFull-reject → re-home → mark summoning-sick [superseded "asleep-unless-rush/charge" — see follow-on] → **re-bucket** triggers under the new owner (no `OnSummon` re-run, `birthEpoch` preserved — 2.a, applied) → aura recalc → `MinionControlChangedEvent`); `CommandAttackAction { attackerId(neutral), targetId, sourceId }` (relaxed attacker rule; full activation honoring Windfury = 2 strikes at the one target, **each re-validated** so a first-retaliation mortal wound fizzles the second; ignores `attacksUsedThisTurn`; `canAttack` pinned true for neutrals; desugars to the combat `DealDamageAction` pair(s)). The neutral-only/enemy-only/either distinction is **selectors on the cards**, not new actions or new selectors (`Union(AllNeutralMinions, AllEnemyMinions)` etc.).
- **Command reaches anything a controlled minion can** (opponent characters incl. hero, or another neutral), under per-lane Taunt; **commanded minions still take retaliation**, and the **fizzle is the attacker's concern only — a mortally-wounded *defender* still retaliates** (its retaliation is a separate `DealDamageAction` with the defender as source, not gated by ③ attack-eligibility; consistent with the dying-swing payoff of the combat-atomicity decision).
- **Neutral graveyard (Req 1):** new shared `GameState.neutralGraveyard` (not per-player). New immutable **origin** flag `MinionOnBoard.bornNeutral` (set *only* by `SpawnNeutralMinionAction`; survives control). §4 ⑦ Phase-1 routing reads two inputs: `ownerId != null` → that player's graveyard; `bornNeutral && ownerId == null` → neutral graveyard; `!bornNeutral && ownerId == null` → **undefined/asserts** (deferred, recorded in Unaddressed Features — reachable only by a future "release to neutral" / "summon into neutral" path, which would need an owner-of-record field not added now).
- **`GraveyardEntry` refactor (the user's challenge):** since the card form is fabricated at death anyway *and* `MinionOnBoard` never retains its originating Card, a stored `originalCard` is derived data that can only drift (and eager-minting burns a `Card.id` per corpse). **Removed `originalCard` from the base**; each subtype keeps its entity snapshot (`snapshot`/`weaponState`; `GraveyardSpell` gains `definitionKey`+`isInverted`), and the card form is **fabricated lazily at point of use** (draw-from-graveyard/resurrect/recast/re-equip), minting the `Card.id` then as a stage-④ action. Net *less* spec; bouncing a controlled token already worked via the existing Fabrication rule.

**Deferred (recorded):** temporary/Shadow-Madness control (permanent only this pass); the `!bornNeutral`-dies-in-neutral routing branch.

**Plan impact:**
- **Epic 01 (data model):** `GameState.neutralGraveyard`; `MinionOnBoard.bornNeutral` + neutral `canAttack`-pinned-true semantics; `GraveyardEntry` hierarchy refactor (drop base `originalCard`, `GraveyardSpell.definitionKey/isInverted`); `ActionRejectionCode.AttackerNotControlled`.
- **Epic 04 (combat/validator):** per-lane Taunt; explicit attacker-control check; `CommandAttackAction` handler (Windfury-aware activation, independent re-validation, desugar to damage pair); defender-retaliation-not-fizzled tests.
- **Epic 0x (control):** `TakeControlAction` handler (BoardFull reject, re-home, asleep-unless-rush/charge, trigger re-registration to new owner list, aura recalc) + `MinionControlChangedEvent`. Tests: control a neutral / control an enemy / board-full reject / asleep-unless-charge / dies-to-controller-graveyard / trigger fires for new owner.
- **Epic 16 (graveyard/fabrication):** lazy card fabrication at point of use; Phase-1 graveyard routing (3 branches incl. the asserting one); neutral-graveyard population tests; fabricate-on-draw/resurrect/recast tests.
- **Lineage effects** now read `snapshot`/subtype identity, never a stored `originalCard`.

### Hole #3 follow-on — summoning-sickness & Charge/Rush eligibility model (2026-06-08)

**The gap (raised by the user's "does control reuse the summon pipeline?" question):** the spec stored a derived `MinionOnBoard.canAttack: bool` as the summon-sickness verdict, read at §4 ③. A single bool **can't encode Rush's restriction** (Rush = attack minions only on the entry turn; Charge = anything), so a Rush minion set `canAttack = true` could illegally go face. Same gap on summon and control. The user asked to *fully spec it*, not log it as a hole.

**✅ DECISION (2026-06-08): replace the derived bool with a raw fact + a single *pulled*, target-aware eligibility check. APPLIED to spec §1/§3/§4.**

- **First, the framing answer to "does control = summon?":** **No** — distinct handlers. Summon *creates* (fresh id, stats from definition, full health, registers triggers, fires Battlecry); control *moves an existing body* (preserves health/enchantments/`isInverted`/`isFrozen`/`bornNeutral`/`grantedKeywords`, **fires no Battlecry/`OnSummon`**, keeps its existing trigger subscriptions + `birthEpoch`). They share only a thin **board-entry routine** (insert at position, `attacksUsedThisTurn = 0`, mark `summoningSick`, aura recalc).
- **The model:** `canAttack: bool` → **`summoningSick: bool`** (raw: entered a board this turn, not yet woken at a controller turn-start; set on entry by summon AND control; cleared by the turn-start sweep, step 6). Eligibility at §4 ③ **pulls** effective keywords for a sick minion — **Charge** → any target; **Rush** → minion-only (enemy or neutral, per-lane Taunt still applies), hero rejected `RushCannotTargetHero` (new code); **neither** → `AttackerCannotAttack`. Non-sick → any legal target.
- **Why pulled, not stored:** it's the keyword-collapse philosophy (pull at the decision point). Aura-granted Charge/Rush works mid-turn and silenced haste is correctly lost, with **no recompute** — and the shared board-entry routine carries **zero** per-handler charge/rush logic (it only sets `summoningSick`). So 2.b stopped being a "rule duplicated across handlers" worry and became a single eligibility function.
- **Neutrals** are never `summoningSick`; `CommandAttackAction` bypasses eligibility entirely (card-granted activation), and retaliation is ungated.

**Sibling fix — the attack BUDGET is derived too (2026-06-08).** The user's "could a minion have `attacksAllowedThisTurn = 2` without Windfury?" exposed a §1↔§3 contradiction: §1 stored `attacksAllowedThisTurn` ("Windfury sets to 2 on summon"); §3 derived it ("`keywords has windfury ? 2 : 1`, read at eligibility"). The stored reading has a **concrete silence-desync bug**: summon Windfury (field = 2) → attack once (used = 1) → silence (keyword gone, field still 2) → `1 < 2` lets it swing again with no Windfury. **Resolved the same way as `summoningSick`:** **dropped the `attacksAllowedThisTurn` field**; the per-turn budget is **pulled** at §4 ③ (`budget = effectiveKeywords has windfury ? 2 : 1`, extensible to Mega-Windfury → 4), eligibility = `attacksUsedThisTurn < budget`. Now impossible to have budget 2 without an effective Windfury-family keyword; silence drops it instantly. A deliberate non-Windfury multi-attack = a budget-granting keyword; a temporary "+1 attack this turn" = the future **Modifier System** (Item 4 deferral), not a stored counter. `attacksUsedThisTurn` stays as the only stored attack-state.

**One sub-choice flagged:** the rush-hits-hero rejection is a new structured code `RushCannotTargetHero` (target-side, sibling of `MustTargetTaunt`) rather than overloading `AttackerCannotAttack` — chosen because codes are the test/localization contract (Item 7) and "Rush can't go face" is a distinct, assertable reason from "fully summoning sick."

**Refinement 2.a — APPLIED (2026-06-08):** the `TakeControlAction` trigger-handling wording is now "**re-bucket** by owner (the bus orders by host owner vs. active player, so however it tracks ownership it must reflect `newOwnerId`); **not** an `OnSummon` re-run — triggers neither re-created nor re-fired; `birthEpoch` preserved." Closes the over-specified "list-move" phrasing.

**Plan impact:**
- **Epic 01 (data model):** `MinionOnBoard.canAttack` → `summoningSick` (semantics inverted: now the sickness fact, not the attack verdict); **`attacksAllowedThisTurn` field REMOVED** (budget pulled from keywords; `attacksUsedThisTurn` remains); `ActionRejectionCode.RushCannotTargetHero`.
- **Epic 04 (combat/validator):** §4 ③ eligibility is now the pulled Charge/Rush/hero check; tests — Charge hits face on entry turn, Rush hits a minion but is rejected on the hero (`RushCannotTargetHero`), vanilla minion can't attack on entry turn, aura-granted Charge enables attack mid-turn, silence removes haste, woken-next-turn attacks normally.
- **Epic 0x (control) / summon:** both call the shared board-entry routine (sets `summoningSick`); control fires no Battlecry; eligibility is not recomputed at entry. Test: control a Charge minion → can attack immediately; control a vanilla minion → asleep this turn, wakes next turn.
- **Turn lifecycle (step 6):** clears `summoningSick` (wake) + resets `attacksUsedThisTurn` for the new active player's minions.

---

## Topics deliberately omitted

These came up in the investigation but don't merit changes:

- **Uniform ID model (single counter for cards and minions).** Old project does this; our spec deliberately splits them. The split was justified by the Card-vs-Minion type distinction and survives every scenario we walked through (tokens, copies, transforms, resurrection, return-to-hand with policies). No change.
- **Animation playback layer.** Belongs to the client (Unity/Godot), not `CCG.GameLogic`. Out of scope.
- **Multiplayer scaffolding.** Belongs to the Game Server (C# / ASP.NET Core), not `CCG.GameLogic`. Out of scope.
- **AI / Bot scaffolding.** Whole subsystem — defer to a future epic after the core lands. Item 5 (`GetLegalActions`) is the only piece the library needs to expose for bots to be possible.

---

## How to use this file

Pick one item at a time. For each: confirm whether it goes into the spec, into the plan, or gets rejected. When a decision is made, this file gets updated with the decision and the spec is amended in the same session.

**Spec-first workflow + Plan-impact convention (set 2026-05-31).** During the borrow-list pass we amend the **spec only**, not the epic/ticket files — those are reconciled in one pass at the end (see below). To keep that reconciliation mechanical rather than archaeological, every `✅ DECISION` block ends with a **Plan impact:** list naming the affected epics/tickets and the change each needs, and flagging where a **NEW ticket** (or epic) is likely required. Carry this convention forward for every remaining item.

## Plan reconciliation — end-of-pass task

After all 13 items are walked through, before implementation begins at Epic 01 / T1.1:

1. Walk every `Plan impact:` list in this file and apply the edits to the corresponding epic/ticket files under `docs/superpowers/plans/2026-05-27-ccg-game-logic/`.
2. **Create new tickets / epics where flagged.** The amendments have already surfaced needs the original plan didn't cover — known so far:
   - `StabilizationAbortReport` telemetry (Item 1 → new Epic 04 **T4.8**).
   - `ITriggerCondition` library as its own unit (Item 2 → new Epic 08 **T8.10**).
   - **NEW EPIC: Bot Support** (Item 5 → end of plan) — houses `GetLegalActions(playerId)` (brute-force-then-filter, library-resident, phase-aware; first ticket) **plus other bot-building logic** (move generation/scoring, state evaluation, bot driver loop — flesh out when writing the epic). Depends only on the Item 5 ②③-predicate seam (Epic 01 validator ticket).
   - `IRandom` foundation (Item 6 → Epic 01) — counter-based per-action reseed `mix(rngSeed, currentActionEpoch)`, `GameState.rngSeed`, init shuffle/coin/deal via Fisher-Yates; the Epic 16 "Determinism/RNG audit" backlog item references it.
   - Structured error codes (Item 7 → Epic 01/validator) — `ActionRejectionCode` enum + `ActionRejection(Code, Detail?)`, `Submit` returns `SubmitResult = Accepted | Rejected`.
   - Tribe/keyword 4-field model (Item 8 + follow-on → data-model + auras tickets) — `Tribe [Flags]` enum, `intrinsic/granted/aura` fields for tribes *and* keywords, `AuraEffect.GrantedTribes`/`GrantedKeywords`, Silence asymmetry.
   - **Keyword-model collapse (Item 8 second follow-on, 2026-06-02 → Epic 01 `IKeyword` + Epic 07)** — `IKeyword { KeywordId }` + role hooks (`IOnDealtDamage`), NO `OnApplied`/`OnRemoved`/bus subscriptions; Epic 07 re-scoped (Lifesteal/Poisonous = hooks, Windfury = declarative read, Enrage = ⑥ recompute, Freeze = turn sweep); aura grants accept any keyword (drop the declarative-only assert); bus carries only `ITrigger`.
   - `ITargetSelector` library (Item 12 → **new Epic 08 ticket**, beside T8.10 `ITriggerCondition`) — pure, ordered by §4 ⑦ board order, singletons + factories + `Filter`(reuses conditions); random-K effects use the pool-carrying-action + stage-④ draw pattern; the §4 ③ validator's target check becomes selector-membership (so the validator ticket depends on this library); `PlayCard`/`HeroPower` card definitions carry a `(selector, cardinality)` target requirement.
   - Replay artifacts (Item 13 → Epic 16) — keep "Event-log replay" (client/spectator path) and **add a command/input-log canonical-replay item** (package = `rngSeed` + ruleset version + decklists + input log; input log captures every `Submit`-ed action incl. timeout/forfeit/disconnect-injected ones); ruleset-version stamp flagged for the Game Server spec.
   Do not force everything into existing tickets — add tickets/epics freely so each finalized amendment has a home, and update the README index + ticket outline + writing-progress tracker to match.
3. Re-verify the README's "Resume point" line and epic index reflect the reconciled plan.
