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

### Hole #1 — `AttackResolvedEvent` vs. two-action combat (2026-06-08)

**The gap:** `AttackResolvedEvent { attackerId, targetId, damageToTarget, damageToAttacker }` bundled *both* combat hit amounts into one event — only producible by atomic apply-both-then-announce combat, which the two-action model replaced. Under two-action combat `AttackAction` ④ enqueues two independent `DealDamageAction`s and finishes; neither hit has landed at ④, each can be modified/intercepted/interleaved, so the two totals are never jointly known as one delta. The event was referenced **nowhere** (purely vestigial) and isn't even a delta (the real deltas are the two `DamageTakenEvent`s).

**✅ DECISION (2026-06-08): Option (b) — remove `AttackResolvedEvent`, add `AttackPerformedEvent`, and write the `AttackAction` handler. APPLIED to spec §2A/§2B/§3.**

- **`AttackDeclaredEvent`** (③′) = the *intended*/pre-interception declaration (renderer telegraph + redirect hook; may fizzle).
- **`AttackPerformedEvent { attackerId, targetId }`** (④) = the *committed*/post-interception swing — chosen over bare removal because `AttackAction` ④ **does** mutate state (consumes the activation, breaks Stealth) and §2's invariant says every state change is announced; right now ④ had no event. Carries **no damage numbers** (those are the two `DamageTakenEvent`s). Also the correctly-shaped anchor for a future "after this attacks" trigger (not in v1's catalog).
- **`AttackAction` ④ handler now specified** (it previously lived only as §3 prose, which is *why* the event got stranded): snapshot base attack → `attacksUsedThisTurn++` → break Stealth (`StealthBrokenEvent`) → emit `AttackPerformedEvent` → enqueue strike + (conditional) retaliation `DealDamageAction`s.
- **Folded in (user request): Stealth + unfreeze.** Stealth now **breaks on attack** at ④ (`StealthBrokenEvent`, new — parallels `DivineShieldBrokenEvent`; §3 `IKeyword` Stealth row updated) — this **closes the Stealth half of hole #4**. Freeze: a frozen attacker is rejected at ③ (`AttackerFrozen`); the `attacksUsedThisTurn` consumed at ④ is the input the end-of-turn unfreeze sweep reads — **the precise `frozenOnTurn` thaw rule is the remaining half of hole #4** (still open).

**Plan impact:**
- **Epic 02/04 (events + combat):** drop `AttackResolvedEvent`; add `AttackPerformedEvent` (④) + `StealthBrokenEvent`; `AttackAction` handler emits them and enqueues the two `DealDamageAction`s. Tests: attack emits Declared(③′)→Performed(④)→two DamageTaken; redirect → Declared(intended) but Performed/DamageTaken(actual); stealthed attacker → `StealthBrokenEvent` on its first swing; renderer reconstructs combat with no AttackResolved.
- **Hole #4 now half-closed:** Stealth-on-attack done; only Freeze `frozenOnTurn` thaw tracking remains.

### Hole #4 (freeze half) — Freeze thaw rule + timing (2026-06-08)

**Two bugs found:** (1) the lifecycle unfroze at **step 4, *after* the active-player swap** — i.e. at the *start* of the controller's turn — which would thaw a minion *before* it was meant to miss its attack, negating Freeze entirely; (2) my hole-#1 `AttackAction` note had the thaw condition **inverted** (claimed frozen-after-attacking thaws, frozen-while-able stays — the reverse of the real rule). Plus there was no `frozenOnTurn` field to evaluate any rule against.

**✅ DECISION (2026-06-08): HS-faithful "Freeze costs exactly one attack" rule. APPLIED to spec §1/§2/§3/§4.**

- **Field:** `MinionOnBoard.frozenOnTurn: int` (stamped `= turn.number` by `FreezeTargetAction`, valid while `isFrozen`).
- **Timing moved to END of the ending player's turn** (Turn Lifecycle **step 3**, after end-of-turn triggers, **before** the swap) — for that player's frozen characters.
- **Rule:** thaw **unless** frozen *this* turn while exhausted — keep iff `frozenOnTurn == turn.number && attacksUsedThisTurn >= budget` (reuses the `AttackerExhausted` test). On thaw: `isFrozen = false`, clear `frozenOnTurn`, emit new **`MinionThawedEvent`**.
- **Why it's correct (Water-Elemental check):** a minion that swings into a retaliation-freezer is frozen *after* exhausting → stays frozen through its **next** turn (misses next attack); a minion frozen on the opponent's turn, or this turn before swinging, thaws at the end of the turn it couldn't act (misses that one attack). Either way: exactly one attack lost.
- **Hero freeze DEFERRED** (recorded in Unaddressed Features): `FreezeTargetAction` accepts a hero, but `isFrozen`/`frozenOnTurn` live on `MinionOnBoard` only; hero-freeze needs `PlayerState.heroIsFrozen/heroFrozenOnTurn` + hero freeze/thaw events + ③ wiring, which belongs with the (lightly-specced) hero-combat path. The thaw rule itself is complete and applies unchanged.

**Hole #4 is now fully closed** (Stealth half in hole #1; Freeze half here), modulo the recorded hero-freeze deferral.

**Plan impact:**
- **Epic 01 (data model):** `MinionOnBoard.frozenOnTurn`.
- **Epic 02 (events):** `MinionThawedEvent`.
- **Epic 04 / turn lifecycle:** move the unfreeze sweep to end-of-ending-player's-turn (pre-swap) with the exhausted-when-frozen-this-turn rule; `FreezeTargetAction` stamps `frozenOnTurn`. Tests: freeze on opponent's turn → can't attack next turn, thaws at its end; Water-Elemental (frozen after swinging) → frozen through next turn; freeze a friendly un-swung minion → misses this turn only, thaws same end-of-turn; Windfury frozen after one of two swings (not exhausted) → thaws end of turn.

---

### Hole #6 — Intervention: multi-card resolution → MTG-style re-declare/re-offer loop (2026-06-09, session 6)

**The question (user-driven Q/A revisit of point D).** What happens when a player holds **multiple** intervention cards whose triggers match the **same** event? Worked example: two pre-damage cards ("before a friendly minion is damaged, grant it Divine Shield" / "…bounce the attacker") when an enemy minion attacks a friendly minion. The spec's ③′ procedure was written purely in the **singular** ("opens *the single-response* window") — and the depth-1 cap covers *nesting*, not *siblings* — so the multi-match case at ③′ was **underspecified**. (⑥′ had a rule — "one window per matched card, hand order" — but ③′ had no parallel, an asymmetry from the point-D worked examples all being post-reaction AoE.)

**Options weighed (full trade-off in session-state).** (1) N sequential windows, fixed hand-order; (2) one window, player picks one card total (one-per-event cap); (3) **MTG-style** — set-valued window, player picks any one, response resolves, the action **re-declares** and re-opens over the remaining cards, "use as many as you have & can afford," **no nesting** (responses can't be intervened). Prior art: HS fires *all* matching secrets (auto, no choice); MTG lets you cast as many instants as you can afford (player-ordered). Neither caps at one → option 2 rejected as unprecedented. Option 1 forces hand-order. **User chose option 3.**

**✅ DECISION (2026-06-09): MTG-style loop, both phases. APPLIED to spec §1/§2/§3/§4.**

- **Set-valued window.** `PendingIntervention.candidateCardIds: string[]` = the responder's matching **+ affordable** reactive hand cards. The player plays **one** (`SubmitInterventionAction { chosenCardId?, targetIds[]? }`) or **skips**. Two granularities: *across cards* the offer is the set; *within a card* its targets are its matched-subject set ∩ `(selector, cardinality)` (AoE batching survives — one card over N subjects).
- **PRE = re-declare loop (③′).** After a **played** response, the held action re-validates (②③); if still legal it is **re-published to ③′** (reverses the old "declared once / not re-published" rule, `spec` §4 ③′) → re-opens over the *remaining* cards; loop ends on **skip**, no-affordable-match, or **fizzle** (cancel/retarget-off-subject/attacker-dead — the fizzle gives the "withdrawal" of a now-moot sibling card *for free*, via re-validation). 
- **POST = re-offer loop (⑥′).** No held action to re-declare; instead re-offer the *remaining* matched set against the same (frozen) events with current board (a card whose condition lapsed drops out).
- **Skip terminates; only a play re-opens.** Because the window is set-valued, a skip changes nothing → re-presenting the same set is pointless. (A skip does **not** consume the card — it stays armed for *future* events, so the "skip the first of two separate damage instances to hold for the second" case still works.) This **replaces** the old ⑥′ "one window per matched card in hand order"; replay determinism now comes from the logged `SubmitInterventionAction` choices, not engine-imposed order.
- **Iteration vs. nesting (the key disentanglement).** Depth-1 stays, but it caps **nesting only** (a response can't itself be intervened). **Iteration** (one responder chaining several of their own cards against one action/event) is **uncapped by count** — it's a different axis, self-bounding because each played card leaves the hand.
- **Termination = mana, no hard cap (user's call).** Pay-on-response: each play costs `effectiveCost`, spent from `mana`. Residual non-termination only if a 0-cost intervention generates another 0-cost intervention; Item-1 stabilization cap is the backstop if such a card ever exists. No engine iteration cap for now.
- **Mana model — user's simplification (replaced my `reservedMana` proposal).** Instead of tracking reserved mana, **move the mana refresh from turn-START to turn-END** (Turn Lifecycle): at the end of your turn you ramp + restore to full; that pool sits through the opponent's turn; interventions spend straight from `mana` (ordinary `effectiveCost ≤ mana`); your next turn begins with whatever's left. **No `reservedMana` field, no special ceiling** — and the ceiling becomes the *full ramped* next-turn pool (the truer reading of "use next turn's mana"). Outcome-identical to HS on your own turns; only bookkeeping location moved. Caveats baked into the spec: game setup seeds turn-1 mana (no preceding turn-end); a future overload-style "less mana next turn" effect applies at this turn-end refresh. *(Open sub-decision the end-of-turn model resolved: ceiling = ramped next-turn pool, not the unramped `maxMana` I'd first proposed.)*

**Worked-example answer (the user's "already-queued actions" question).** When a strike's ⑥′ window opens, A's responses **resolve immediately, ahead of the pre-existing queue** (the already-queued retaliation + the attacker's "draw on damage") — per the load-bearing immediate-resolution rule (`spec` §4 "Response resolution"). So A reflects/heals before the retaliation lands (combat-as-two-actions, per-hit reaction, `spec:698`), and the attacker's earned draw resolves after A's reactions. The ⑥′ window is a checkpoint *inside* the cascade at the triggering action's position, not behind the queue.

**Plan impact:**
- **Epic 01 (data model):** `PendingIntervention.candidateCardIds: string[]`; **remove** any `reservedMana` notion (never added); no new mana field.
- **Epic 02 (actions):** `SubmitInterventionAction { playerId, chosenCardId?, targetIds[]? }` (skip = null card; exempt from ③ turn-ownership; pays from `mana`); `StartInterventionAction` gains `candidateCardIds[]`.
- **Epic 04 / turn lifecycle:** **move the mana ramp+restore to end-of-ending-player's-turn** (it now sits beside the unfreeze sweep, both pre-swap); delete the start-of-turn mana step; game-init seeds each player's turn-1 `mana`/`maxMana`. Tests: off-turn intervention spends from `mana` and reduces next turn's pool; full-ramped ceiling; turn-1 seeding.
- **Epic 0X (intervention/reactive engine — wherever ③′/⑥′ live):** set-valued window; PRE re-declare loop (held action re-published to ③′ after a play; fizzle ends loop); POST re-offer loop; skip terminates / only-a-play-re-opens; depth-1 caps nesting only; iteration bounded by mana. Tests: two pre-damage cards on one attack (play both in chosen order; play one then skip; first card cancels → second's window withdrawn by fizzle); two post-reaction cards on one strike resolve before the queued retaliation; skip leaves cards armed for a later event; can't afford the second card → loop ends.

**Follow-up (2026-06-09, same session) — secret-vs-intervention worked example → two refinements.** Probe: A holds an active **secret** "before the hero takes damage, add 8 armor"; B holds an intervention "when a friendly minion takes damage, deal that much to the enemy hero"; A attacks B's 1/1 with a 1/1. Resolution: at the strike's ⑥′, B's reflect resolves immediately (ahead of the queued retaliation); the reflect's ③′ fires **A's secret inline** — because the **depth-1 cap suppresses player *windows* only, not auto/secret reactions** — so 8 armor absorbs the 1 reflected damage; then the queued retaliation kills A's 1/1; both 1/1s die in one shared wave. Two refinements applied:
- **(a) Depth-1 wording tightened (4 spots).** "A response cannot itself be intervened" **over-claimed** — this example shows an auto secret legitimately *intercepting* a response's sub-action. Reworded everywhere to "no further **player window** opens on a response; auto/secret hosts still fire — and may intercept — inline" (§3 pillar-pre, §4 ③′ step 3, §4 Response-resolution, Ordering-table Interception row). No behavioural change — the operative rule was always player-windows-only; the prose just contradicted it.
- **(b) Hero armor logged as a gap → hero-combat pass.** `armor` is **not** in `PlayerState` (§1) — it existed only as an interception *example* in §3. Added Unaddressed-Features **"Hero armor"** (needs `PlayerState.armor` + `GainArmorAction` + `ArmorGainedEvent` + a pinned slot in the §4 ④ damage precedence: absorbs after reductions/caps, before health, Divine Shield separate). Deferred to the **hero-combat pass** (bundles with hero-freeze); the consume-armor mechanics already resolve once the field exists. **Plan impact:** Epic 01 (`PlayerState.armor`), Epic 02 (`GainArmorAction`/`ArmorGainedEvent`), hero-combat pass (precedence slot) — all alongside the hero-freeze items.

---

### Hole #2 — Fatigue: empty-deck draw → escalating self-damage (2026-06-09, session 7)

**The hole.** `DrawCardAction { playerId, sourceId? }` was the only draw path and the spec **never said what an empty-deck draw does** — so a game couldn't be lost by decking out. Worse, `HeroMortallyWoundedEvent` already listed `fatigue` among its `cause` values: a forward reference to a mechanism that didn't exist. No counter on `PlayerState`, no fatigue action, no fatigue event.

**Prior art (Hearthstone-exact).** Drawing from an empty deck deals escalating damage to your own hero — 1, 2, 3, … — incrementing per failed draw, a per-player counter that never resets. Fatigue is sourceless and (in HS) reduced by armor like any hero damage.

**Forced by locked decisions.** Point-A close-out locked "**all damage routes through `DealDamageAction`**" (`spec:700`). Fatigue *is* hero damage, so it must enqueue a `DealDamageAction` — a self-contained fatigue path would contradict that and miss the ④ modifier precedence (future hero-armor slot), interception/⑥′ windows, and the hero→`HeroMortallyWoundedEvent`→⑧ settle path. So routing was not a fork; only the **counter model** and the **`cause` discriminator** were open.

**The real fork — the `cause` discriminator (user chose: drop it).** `cause` sat on **both** `MinionMortallyWoundedEvent` and `HeroMortallyWoundedEvent` but **nothing populated it** — `DealDamageAction`/`DamageTakenEvent` carry no cause. It is **not mechanically load-bearing** (no card branches on it; save-from-lethal fires regardless). Three options weighed: (1) **drop `cause`** from both events + add a dedicated `FatigueDamageEvent` (leanest — closes the forward-ref by deleting the dangling field); (2) full `DamageCause` taxonomy on `DealDamageAction` + map destroy/aura-loss (coherent but real hot-path surface for an informational field); (3) fatigue-minimal middle (cause only on `DealDamageAction`). **User chose (1)** — YAGNI; the per-cause taxonomy isn't earned by any consumer.

**✅ DECISION (2026-06-09): ADOPTED — counter on `PlayerState`, route through `DealDamageAction`, drop `cause`, add `FatigueDamageEvent`. APPLIED to spec §1/§2A/§2B (+ §6 ⑥ aura-loss reword).**

- **§1:** `PlayerState.fatigueCounter: int` (init 0, persists all game, never resets; increment-then-deal — `++` then deal the new value).
- **§2A:** expanded the `DrawCardAction` ④ handler to three explicit branches — non-empty+room → `CardDrawnEvent`; non-empty+full hand → burn/`HandOverflowEvent`; **empty → `fatigueCounter++`, `FatigueDamageEvent`, enqueue `DealDamageAction { target = hero, amount = fatigueCounter, sourceId = null }`**. Sourceless even when an effect forced the draw; single-card action ⇒ a "draw N" effect ticks once per empty draw; empty-deck and full-hand mutually exclusive per draw.
- **§2B:** added `FatigueDamageEvent { playerId, amount }` (beside `HandOverflowEvent`; fires before the damage lands — telegraph then hit, the `AttackPerformedEvent`→`DamageTakenEvent` shape; it's how the client knows a hero hit was fatigue). **Removed `cause`** from `HeroMortallyWoundedEvent` → `{ playerId, sourceId }` and `MinionMortallyWoundedEvent` → `{ minionId, sourceId }`, with an explanatory note on each. **Reworded §6 ⑥** ("cause = aura loss" → "an aura-loss death") since that was the one place a `cause` value was being *written*.
- **Flow-through (no new code):** deck-out loss + mutual-deckout draw fall out of the existing hero-damage settle path; the future `PlayerState.armor` absorbs fatigue automatically once it exists (hero-combat pass).

**Same-pass cleanup — stale `ActionDeclaredEvent` text (session-6 miss).** §2B `ActionDeclaredEvent` still said "Fires **once** per action … resumed … **not** re-declared (prevents redirect/secret loops)" — contradicting the session-6 **re-declare loop** (`spec:842`, §4 ③′). Session-6's grep didn't cover "fires once"/"not re-declared." Reworded to "published afresh on each (re-)declaration … bounded by the depth-1 nesting cap + finite mana." No behavioural change; doc consistency only.

**Plan impact:**
- **Epic 01 (data model):** add `PlayerState.fatigueCounter: int` (init 0, never resets).
- **Epic 02 (actions/events):** expand the `DrawCardAction` handler (three branches incl. fatigue); add `FatigueDamageEvent { playerId, amount }`; **drop the `cause` field** from `HeroMortallyWoundedEvent` and `MinionMortallyWoundedEvent` (and from any ticket that constructs them — e.g. the aura-recalc death and combat-damage death tickets). No new action type.
- **Tests:** draw from empty deck deals 1 then 2 then 3 (counter persists, never resets); a draw that empties the deck draws the last card *then* the next draw fatigues; "draw N" past deck end ticks once per empty draw; lethal fatigue → `HeroMortallyWoundedEvent` → loss; simultaneous mutual deck-out → draw; `FatigueDamageEvent` precedes the `DamageTakenEvent`.

---

## Queued topics — to investigate next (logged 2026-06-09, session 6)

Raised but not yet worked. Distinct from "deliberately omitted" below (those are rejected); these are open items awaiting a session.

1. ✅ **RESOLVED (2026-06-09, session 7) — Trigger ordering = deathrattle ordering: unified into ONE canonical board order. APPLIED to spec §3 (`IEventBus`, `ITargetSelector`) + §4 (⑤ Publish, Turn Lifecycle) + Deterministic-Ordering table.** The hole was real: `IEventBus` kept **two** subscriber lists (current player, opponent) and **never mentioned neutral**, while death sort had three groups ending in neutral — and the `ITargetSelector` row already *assumed* a unified "death/trigger" order that the trigger side didn't actually state. Fix: (a) **one canonical board order** — active player `board[0..n]` → opponent `board[0..n]` → **neutral by index** — used identically by trigger dispatch (`IEventBus`, snapshot at *publish*), death/deathrattle/reborn sort (`DeathResolutionService`, snapshot at *collect*), and selectors; board index primary, `summonOrder` the fallback; heroes enter only via selectors. (b) **Neutral's slot pinned last** on the trigger side (`IEventBus` gains a third dispatch group). (c) **De-duplicated** the two Ordering-table rows into one "Canonical board order" row. (d) **Design call:** neutral minions fire **board-wide event** triggers (last) but **not turn-scoped** Start/End-of-Turn triggers (no turn of their own) — falls out of condition semantics (controller-scoped conditions can't bind a controllerless host); Turn Lifecycle steps 2/10 now state the exclusion intentionally. **Plan impact:** Epic 03/engine (`IEventBus` dispatch = three ordered groups, neutral last); Epic 04/turn-lifecycle (turn-trigger steps exclude neutral, stated); selector-library ticket cross-refs the one canonical-order rule; no data-model change. **Tests:** neutral minion's on-death/on-damage trigger fires after both players' in board order; neutral minion with an End-of-Turn trigger never fires it; a "your"-conditioned trigger on a neutral host doesn't match.
2. ✅ **RESOLVED (2026-06-09, session 7) — Neutral-lane auras → generalized into a LANE-BASED friendly/enemy model (central, not aura-specific). APPLIED to spec §1, §2A/§2B, §3 (`ITriggerCondition`, `ITargetSelector`, `IAura`, `IEventBus`), §4 ⑥.** Started as "do neutral minions emit auras"; the user's probe ("a neutral minion's friendly minions are its lane-mates") showed the real question was the **central** `friendly`/`enemy` definition (used by auras, triggers, *and* selectors alike), which was controller-based and left the neutral case empty. **North-star (user):** *the neutral lane should behave as close to a regular lane as possible so mechanics work out of the box.*
   - **Lane-based friendly/enemy, formalized (user's predicates).** For host `h`, evaluated entity `e`: `friendly(h,e) ⟺ h.ownerId == e.ownerId` (null==null true → a neutral host's friendlies are its lane-mates); `enemy(h,e) ⟺ h.ownerId != e.ownerId && e.ownerId != null`. The `e.ownerId != null` clause makes a **neutral minion never anyone's *enemy*** — for a player host neutral stays the separate third category (selector trichotomy), while a neutral host *does* see both boards as enemy (asymmetry falls out of the clause). For player hosts this is HS-identical (your board == your friendlies). Mutually exclusive; exhaustive only from a neutral host.
   - **Auras (the original question) fall out:** a player's friendly aura excludes neutral; a neutral minion's friendly aura buffs its lane-mates; reaching the lane from a player aura needs an explicit `AllNeutralMinions`/`AllMinions` selector; `AdjacentTo` is per-lane. **No hardcoded neutral wall**; recalc (§4 ⑥) already rewrites `aura*` on every minion incl. neutral.
   - **Triggers inherit it too** (user confirmed the blast radius): a neutral minion's `FriendlyOnly` trigger fires off lane-mates ("when a friendly minion is summoned, +1/+1" works in the lane). **Corrected a session-7/topic-#1 overstatement** — I'd lumped `FriendlyOnly` with the genuinely-empty turn/hero-context conditions; only Start/End-of-Turn (no turn), Inspire (no hero power), Combo (no hand/play-count) stay empty for neutral — the friendly *relation* resolves.
   - **Summon-event consolidation (user's call + north-star):** a neutral spawn now emits the **regular `MinionSummonedEvent` with `ownerId == null`**, entering via the same board routine as a normal summon — so summon-watching triggers (incl. the lane-based friendly example) catch it out of the box. **Removed `NeutralMinionSpawnedEvent`** (its existence was the bug — generic triggers would miss neutral); `NeutralZoneRepopulatedEvent` demoted to a renderer-only batch marker (per-minion summons go through `MinionSummonedEvent`).
   - **Plan impact:** Epic 02 (events) — **drop `NeutralMinionSpawnedEvent`**; `MinionSummonedEvent` is the sole summon event (neutral = null owner); `SpawnNeutralMinionAction` emits it + shares the summon board-entry routine; `NeutralZoneRepopulatedEvent` = renderer-only. Epic 08 (condition + selector libraries) — `FriendlyOnly`/`EnemyOnly` + `AllFriendly/Enemy/NeutralMinions` implement the two predicates verbatim (one shared helper). Epic 0X (aura engine) — recalc covers the neutral lane; no neutral wall. No data-model field change. **Tests:** neutral minion's friendly aura/trigger hits lane-mates but not players; player friendly aura/trigger skips neutral; a player "all enemy minions" effect does not hit neutral, an `AllNeutralMinions` one does; neutral Poisonous treats a player minion as enemy; neutral spawn fires `MinionSummonedEvent(null owner)` and a watching neutral's "friendly summoned" trigger fires (but not on its own summon — epoch filter).
3. ✅ **RESOLVED (2026-06-10, session 8) — Debug text format → a structured per-action DEBUG TRACE (`ITraceSink`, JSON Lines). APPLIED to spec §3 (new "Debug Trace (`ITraceSink`)" subsection after `IRandom`) + §4 (intro cross-ref, `StabilizationAbortReport` note).** The user re-scoped the topic at the first question: NOT a scenario-authoring/parse format — a **read-only diagnostic the engine emits per processed action** (state + queued actions + events) for stepping through a failed test manually. Q&A pinned: **diff per action + full snapshots at cascade boundaries** (not full-state-per-action); **stages traced only when they act** (quiet stages invisible). Format fork (sigil ledger vs narrative vs YAML) **dissolved by the user's reframe**: emit *parsable structured records* and let any human-friendly rendering — incl. an eventual **visualizer** — be a derived view; we spec ONE schema, not three formats. With raw readability demoted (user: visualizer makes it less important), encoding landed on **JSON Lines via System.Text.Json** over YAML/XML/binary — zero dependency, the stack `JsonElement` already commits to, append-as-it-happens (a mid-cascade crash leaves a valid trace), greppable/diffable line-per-record; YAML's only edge (raw readability) no longer decisive, plus YamlDotNet dependency + implicit-typing footguns.
   - **Model:** `ITraceSink { void Record(TraceRecord) }`, engine ctor takes `ITraceSink?` (null = zero overhead). Emission from *inside* stages (rejections produce zero events — invisible to any event-stream post-processor). **Diffs by structural compare** of before/after state (clone cost only when attached; nothing can mutate unseen). `JsonLinesTraceSink(Stream)` = default impl.
   - **Record kinds:** `state` (match start + each settle-to-idle; full snapshot incl. ordered decks; `definition` bodies ELIDED — `definitionKey` only, Fabrication-rule philosophy), `action` (per ④, epoch + §2A params + only-if-nonempty events/diff/enqueued, enqueued entries tagged `src` enqueuer; aura recalc = just diff entries), `reject` (action + error code), `window` / `choice` (opening only; the `Submit*Action` resolution is its own `action` record — append-only real-time order, re-declare/re-offer loop traces as alternating records), `settle` (per ⑦ wave; ⑧ only when it acts).
   - **Conventions:** bare entity ids (join via `state` records); dotted diff paths with `[from,to]`; zone moves = id key + zone-path values; payloads defer to §2 verbatim; **no `queueAfter`** (derivable from `enqueued` + FIFO — ruled redundant).
   - **Non-goals:** third purely-diagnostic stream — no replay/persistence/wire role (vs command log = canonical, event log = wire); **visualizer + pretty-printers out of library scope** (same ruling as animation/bots); no authoring DSL (tests keep programmatic Build).
   - **Prior art:** HS `Power.log` (BLOCK_START/TAG_CHANGE/BLOCK_END ≈ action blocks with field diffs — proof the shape works at scale, deck trackers parse it); old-project Spine/Exact traces (a structured trace enables record-matching assertion helpers later); `StabilizationAbortReport.cascadeTrace` = the event-only cousin (report + attached sink = strictly richer repro view).
   - **Plan impact:** Epic 03/engine — new ticket: `ITraceSink` + `TraceRecord` kinds + structural-diff compare + `JsonLinesTraceSink` (System.Text.Json, compact one-line records). Testing epic — test-helper hook "attach trace sink, dump on failure". No data-model change; no new events.
   - **Tests:** rejected action yields a `reject` record and no `action` record; a cascade's records reconstruct the queue (enqueued+FIFO) exactly; mid-cascade exception leaves a parseable JSONL file ending at the failing action; `state` snapshot elides `definition` bodies; diff catches an aura recalc (`auraAttackBonus`/`attack` entries) with no stage marker; window→submit→re-offer loop appears as alternating `window`/`action` records.
4. ✅ **RESOLVED (2026-06-10, session 8) — Intervention visibility (responder's-eye view) → ONE information ceiling + directed/public wire split. APPLIED to spec §1 (`PendingIntervention.candidates`, new `InterventionCandidate`), §2A (`StartInterventionAction`, `SubmitInterventionAction`), §2B (`eventId`; "Directed events" paragraph; Choice/Intervention event rows split), §3 (new "Visibility — the responder's side" block; deferred list re-worded), §4 (③′ step 2, ⑥′ step 1, PendingIntervention Interruption).** Driven by a **prior-art survey the user requested** (MTG stack / YGO chain links / LoR stack-with-targets / FaB attacks vs the no-window camp HS/Gwent/Snap): two findings — **(1) cause visibility is unanimous** (every game with a responder window shows the pending object in full; hidden identity exists only on pre-armed face-down cards, i.e. it protects the *responder's* card, never the actor's pending action); **(2) the real split is window existence** — *structural* windows (MTG paper, LoR: offered regardless, leak-free, slow) vs *contingent* (Arena auto-pass, Master Duel prompts: only when you hold a response — fast, but the pause is a tell). User locked: **(Q1) ③′ cause = the held action VERBATIM** ("this spec should do the same as that is what the players expect, note the unanimity"); **(Q2) contingent + thin public event** (Arena/Master-Duel camp; engine unchanged, tell accepted; masking = server/UX option).
   - **The information ceiling (principle):** a window prompt never widens the responder's visibility — exactly *their own candidates + the cause*, where the cause is public-by-nature (③′ declared params are pre-resolution facts; casting = public reveal) or already on their wire (⑥′ matched events, published at ⑤). Corollary: the prompt **references** ⑥′ events (`causeEventIds[]`) rather than re-embedding payloads — a trigger matching the opponent's `CardDrawnEvent` can never expose the drawn card. Secret-typed `PlayCardAction` declaration identity = carve-out routed to the deferred secret bundle.
   - **Wire shape — directed/public split, formalized** (mulligan precedent made explicit in a §2B "Directed events" paragraph): responder-directed **`InterventionPromptEvent`** (cause + `candidates[]` + timeout) vs public thin **`InterventionWindowOpenedEvent`** (responder, timeout); `InterventionWindowClosedEvent.wasSkipped` stays public (observationally inferable). **Adjacent fix, same hole:** `ChoiceStartedEvent` was broadcasting `options[]` — would leak **Discover options**; split into public thin (`optionCount`) + directed **`ChoicePromptEvent`** (options to the chooser only).
   - **Data model:** `PendingIntervention.candidateCardIds: string[]` → **`candidates: InterventionCandidate[] {cardId, targetIds[]}`** — engine-computed legal targets per candidate (⑥′ = matched subjects ∩ `(selector, cardinality)` family, recomputed per re-offer pass; ③′ = the card's own targeting; cancel/retarget-type target the held action). Single source for the §4 ③ membership validation AND the directed prompt — client stays a pure renderer. New **`eventId: long`** on all events (append-order, global — directed filtering never renumbers a stream) as the stable ref for `causeEventIds[]`.
   - **Deferred remainder (unchanged):** armed-state telegraph (standing hand-live/secret threat pre-shown HS-style vs hidden MTG-style — now explicitly independent of the pinned responder-side), secret flavour/zone/identity, cost timing, marked-for-destruction scope.
   - **Plan impact:** Epic 01 (data model) — `candidates`/`InterventionCandidate`; `eventId` on the event base. Epic 02 (events) — `InterventionPromptEvent` + `ChoicePromptEvent` new; `ChoiceStartedEvent` thinned to `optionCount`; `InterventionWindowOpenedEvent` confirmed thin; directedness flag on event types. Epic 03/engine — candidate-target computation at window open/re-offer; `StartInterventionAction` handler emits both events. Game Server epic — per-player wire fan-out honors directedness (directed events absent from opponent + spectator streams).
   - **Tests:** opponent's wire stream contains the thin opened-event but never `InterventionPromptEvent`/`ChoicePromptEvent`; Discover: chooser's wire has `options[]`, opponent's has only `optionCount`; ⑥′ prompt's `causeEventIds` all reference events already delivered to the responder; a draw-triggered window reveals no card identity beyond the responder's wire; submit-validation rejects `targetIds ⊄` the chosen candidate's `targetIds`; re-offer pass recomputes `candidates` (lapsed-subject card drops out).

---

## Hero-combat pass (session 8, started 2026-06-10→11)

The bundled deferrals: hero attack/weapon/durability wiring, hero armor, hero freeze. Opened with the §4-③/attack inventory; **mid-pass the user pivoted the hero kit** (below). Decided so far: **hero defenders never retaliate** (HS-faithful — weapons spend durability only on the hero's own swings; rejected the symmetric one-rule model as alien to HS-likes and weapon-hostile).

### Part 1 — ARTIFACTS replace the hero-power system (DECIDED + APPLIED 2026-06-11)

**User-initiated spec change** ("in light of new information and discussions with stakeholders and potential players"): instead of HS-style unique hero powers, a new **Artifact** system. APPLIED to spec §1 (new `ArtifactOnBoard` + `PlayerState.artifacts` cap 3 + `GraveyardArtifact`; `heroPower`/`heroPowerUsedThisTurn`/`HeroPower` type REMOVED), §2A (new `ActivateArtifactAction`/`DiscardArtifactAction` player actions + `EquipArtifactAction`/`DestroyArtifactAction`/`ModifyArtifactChargesAction` system actions; `UseHeroPowerAction` REMOVED; CardType `HeroPower` → `Artifact`), §2B (6 artifact events; `HeroPowerUsedEvent` + `InspireTriggeredEvent` REMOVED), §2C (`ArtifactSlotsFull`, `ArtifactNotActive`), §3 (zone-hosting table row; `IEventBus` owner-group ordering; trigger catalog + neutral-empty list: Inspire → **`OnArtifactActivated`**; Source Attribution), §4 (⑤ publish order, Turn Lifecycle step 7, Deterministic-Ordering row, trace `a` prefix).

- **Model:** artifacts = first-class entities in a per-player row (cap **3** incl. starter), **passive** (host triggers, may host an `IAura`), **active** (activate → effects; optional cost ≥ 0 + target via selector), or **both**. No attack/health — never combat/death-wave entities, not Silence-able, not aura-receivers. Acquired by playing the new **Artifact card type** or effect-enqueued `EquipArtifactAction`; owner may **discard** in-turn (free, unlimited). All heroes share the same **starter artifact** (equipped at match setup): active-only "pay mana, draw a card", unlimited per turn, **escalating cost = base + usesThisTurn**.
- **User-locked forks:** (1) **full hero-power replacement** (fields/action/card-type/events gone; Inspire re-pointed); (2) **escalation is starter-specific and per-turn-resetting** (a definition cost formula, not a system rule — the engine seam is a *pulled* effective-cost formula over `(artifact, state)`, default = base, matching the keyword-pull philosophy); (3) **full row rejects at validation** (`ArtifactSlotsFull`; effect-enqueued equips fizzle at ④ — the full-board-summon pattern); (4) **ONE durability counter + definition-declared consuming moments** (activation / trigger-fire / both; null = infinite; 0 → destroy) over per-side counters.
- **Clarified on request:** (a) **trigger ordering** — artifacts append to their owner's dispatch group **after that owner's minions** (slot order within), NOT a fourth global group — preserves the active-player-first promise, cheap for the owner-bucketed bus; death sort untouched (artifacts never die in waves). (b) **Discard vs destroy events are disjoint** (never both): the state delta is identical, the split exists for trigger vocabulary ("when destroyed" must not fire on voluntary discards; discard-synergy needs its own match) + renderer — the session-7 idiom (distinct event types over a `cause` enum). "Leaves play for any reason" subscribes to both.
- **Flagged rulings (accepted):** generic `charges: int` accumulator (mutated only via `ModifyArtifactChargesAction`); activation does **not** increment `cardsPlayedThisTurn` (Combo unaffected); activation declared at ③′ (interceptable); `usesThisTurn` reset in lifecycle step 7; starter base cost is content-tunable (suggest 1); artifacts participate in turn-scoped triggers (owner has turns — unlike neutral); dropped `InspireTriggeredEvent` rather than renaming it (`ArtifactActivatedEvent` is the hook; no marker needed).
- **Plan impact:** Epic 01 (data model) — `ArtifactOnBoard`, `PlayerState.artifacts`, `GraveyardArtifact`, removals; Epic 02 (actions/events) — the 5 actions + 6 events, removals, rejection codes; Epic 03/engine — equip/activate handlers, pulled-cost seam, durability consumption, owner-group trigger ordering; **any hero-power tickets in the plan re-scope to artifacts** (sweep at reconciliation); content epic — starter-artifact definition; Epic 08 (conditions) — `OnArtifactActivated` replaces Inspire.
- **Tests:** starter costs base, base+1, base+2 within one turn and resets next turn; 4th equip rejects for a player play / fizzles for a battlecry; dual-example artifact charges off neutral deaths and destroys after 3 activations (`ArtifactDurabilityLostEvent 0` → `ArtifactDestroyedEvent`); discard emits only `ArtifactDiscardedEvent` (a "when destroyed" trigger does NOT fire); active player's artifact trigger fires after their minions but before opponent's; activation neither breaks Combo counting nor consumes a card play; neutral host's `OnArtifactActivated` never matches.

### Part 2 — the HERO-WEAPON CONCEPT is DROPPED; armor lands (DECIDED + APPLIED 2026-06-11) — PASS CLOSED

**User-initiated, at the re-posed `heroAttack` question:** rather than pick temporary-vs-persistent semantics, the user dropped the whole concept — **heroes can't attack, can't be frozen, and have no equippable weapons; the artifact system is the hero kit.** This dissolved items 1, 2 and 4 of the pass (hero-as-attacker wiring, durability consumption, hero freeze) outright; item 3 (**hero armor**) was the one genuine residual fork — armor is *defense*, independent of hero offense — and the user chose **keep, land now** (its deferral reason, "rides the lightly-specced hero-combat path", evaporated when that path was deleted).

- **Demolition (spec):** §1 — `PlayerState.heroAttack` + `weapon?` REMOVED; `WeaponOnHero` + `GraveyardWeapon` types REMOVED; `CardType` → `Minion | Spell | Artifact` (**no weapon cards exist**); `baseAttack?/baseHealth?` → Minion-only. §2A — `EquipWeaponAction`/`DestroyWeaponAction` REMOVED; ID-convention example re-pointed to artifacts. §2B — `WeaponEquippedEvent`/`WeaponDestroyedEvent`/`WeaponDurabilityLostEvent` REMOVED; `ArtifactDurabilityLostEvent` row de-paralleled. §3 — Source Attribution + `CardTypeIs` drop weapons; the freeze status paragraph is now **minion-only**. §4 — Unaddressed-Features "Freezing a hero" + "Hero armor" entries replaced by one RESOLVED record.
- **Freeze ruling (flagged, no real fork):** heroes cannot be frozen — freeze denies attacks and heroes have none; `FreezeTargetAction.targetId` is a **minionId**; "frozen character" → "frozen minion" swept (§3 status paragraph, §2A `AttackAction` row, Turn-Lifecycle step 3). No hero frozen fields ever existed — nothing to remove, only the deferral entry.
- **Armor landing:** `PlayerState.armor: int` (init 0, **no cap**); `GainArmorAction { playerId, amount, sourceId? }` → `ArmorGainedEvent { playerId, amount }` (the sole gain path); ④ precedence step **(6) generalized to "the absorb slot, by target type"** — minion: Divine Shield (all-or-nothing break); hero: armor (`absorbed = min(armor, amount)`, partial lets the remainder through) — immune (5) still short-circuits before any armor loss. **`armorAbsorbed` decision: a FIELD on `DamageTakenEvent`** (`targetId, amount, armorAbsorbed, overkill, sourceId`; health loss = `amount − armorAbsorbed`) — the `overkill` idiom (one occurrence, split delta), NOT the disjoint-event idiom (that's for different occurrences, per session 7). No armor-loss event. Fatigue now auto-absorbs (§2A DrawCardAction note updated); the §3 interception "gain armor" pillar + the session-6 armor-secret worked example are now fully backed by the model.
- **Part-1 lock now structural:** "hero defenders never retaliate" needs no rule — retaliation is gated on defender attack > 0 (§2A `AttackAction` step 5) and a hero *has no attack*. Heroes remain ordinary attack **targets** (`RushCannotTargetHero`, per-lane Taunt, command targeting all unchanged).
- **Plan impact:** Epic 01 (data model) — remove `heroAttack`/`weapon`/`WeaponOnHero`/`GraveyardWeapon`/Weapon CardType, add `armor`; **delete or never-create any weapon tickets** (sweep the plan for weapon/hero-attack scope at reconciliation — hero-power tickets were already re-scoped to artifacts in Part 1); Epic 02 (actions/events) — remove the 2 weapon actions + 3 weapon events, add `GainArmorAction`/`ArmorGainedEvent`, `DamageTakenEvent` gains `armorAbsorbed`; damage-handler epic — the ④ absorb-slot branch (target-typed) + armor consumption; validator — `FreezeTargetAction` minion-only target.
- **Tests:** partial absorb (8 dmg vs 5 armor → `armorAbsorbed 5`, health −3, armor 0); full absorb (3 dmg vs 5 armor → `armorAbsorbed 3`, health unchanged, **`DamageTakenEvent` still fires**, no `HeroMortallyWoundedEvent`); immune → no armor loss; fatigue absorbed by armor (deck-out postponed); minion `DamageTakenEvent` always `armorAbsorbed 0`; armor stacks across multiple `GainArmorAction`s (no cap); the session-6 secret ("before the hero takes damage, add 8 armor") absorbs a reflected hit end-to-end.

**HERO-COMBAT PASS CLOSED (2026-06-11).** Everything the pass bundled is resolved: no-retaliation (Part 1, now structural), artifacts (Part 1), heroAttack/weapons/hero-freeze (dropped, Part 2), hero armor (landed, Part 2). The only remaining pre-implementation task is the **end-of-pass plan reconciliation**.

## Sigils — the unified reactive-card model (session 9, DECIDED + APPLIED 2026-06-11)

**Closes the entire deferred secret bundle** (point D game-feel leftovers: auto flavour, zone, constraints, telegraph, cost timing, identity — surfaced by the session-9 deferral scan). **User-initiated reframe:** instead of secrets and hand-interventions as two card kinds, ONE family — every reactive card supports both deployments, and the host zone (already the behavioral discriminator in the §3 hosting model) IS the mode selector. The engine "already secretly agreed": zone-scoped hosting needed zero new machinery for this.

**Vocabulary (user-defined):** the reactive card = **sigil**; playing it from hand in a window = **invoking** (an **invocation**); pre-arming it face-down = **inscribing**; the armed entity = an **inscription**; the zone = `PlayerState.inscriptions`. Engine identifiers keep the `Intervention*` names for the window machinery.

**The three axes (all zone-derived):** choice (window has it / inscription auto-fires on first match), depth (player windows depth-1-capped / inscriptions fire inline even on response sub-actions), economy (invoke pays later from the carried off-turn pool + hand slot / inscribe pays now from the on-turn pool + face-down tell). The session-6 turn-end mana model auto-prices the modes — same card cost, two pools — dissolving the dual-mode differential-pricing problem.

**User-locked forks:**
1. **Universal inscribability — choice degradation → random** (REPLACED my choice-free-guard recommendation): even choice-ful sigils can be inscribed; an inscription resolves every player decision at random via `IRandom` — target requirements consume as **random-K at ④** (the *existing* `ITargetSelector` consumption mode; the mode flips, the card doesn't), modal choices uniform. "Better to keep the mechanic consistent even if players are unlikely to ever use it for some cards."
2. **Constraints: one-per-definitionKey, cap `MaxInscriptions = 3`** (tight cap keeps hold-vs-inscribe tense under unification; matches the artifact row). `InscriptionSlotsFull` / `SigilAlreadyInscribed` at ③.
3. **Actor-attribution firing rule** (replaced BOTH own-turn options after the user's probe "opponent invokes on my turn — technically still my turn?"): an inscription never fires on events caused by its own owner's action — attribution = submitter (player actions) / source's controller (effect actions, §3 Source Attribution); neutral causes are unattributed → spring both sides. Turn-identical to HS on normal turns; correctly lets my inscription punish an opponent's invocation during my turn. Confirmed: windows never change `activePlayerId` (a window is a suspension inside the actor's turn — the ③ turn-ownership exemption wording depends on it).

**Flagged rulings (accepted):** pay-on-inscribe (ordinary play cost; trigger fires free — arm-free/pay-on-trigger breaks auto-resolve); one-shot (fire → reveal → graveyard; persistent standing triggers = passive artifacts' job); empty-pool guard (required selector pool empty at fire time → stays inscribed, unconsumed); inscribed + held copies independent (auto fires in dispatch order, held copy still offered at ⑥′); dispatch slot = inside owner's group after their artifacts, inscription order (auto-chains terminate structurally: each fire consumes, rows cap at 3 — no depth rule); identity hidden / existence public (owner-directed `SigilInscribedEvent` + public thin `InscriptionAddedEvent` *instead of* `CardPlayedEvent`; full reveal at fire = `SigilRevealedEvent`; mana delta makes full hiding impossible anyway); redacted ③′ declaration for inscribe plays ("inscribes a sigil" + mana — the survey's one sanctioned hidden-identity case); inscribe counts toward `cardsPlayedThisTurn`.

**For/against trail:** FOR — leaner than restricting (zone-decides is already law), one pool double duty, signature-mechanic differentiation (nearest prior art: YGO quick-play spells), dissolves the typing fork. AGAINST (accepted as content problems, not engine ones) — per-card mode dominance likely (choice stays valuable only at margins), double balance filter, dual-text comprehension tax (mitigated by the normal/inverted dual-section precedent), deduction-game dilution (anonymity set = whole sigil pool).

**Spec edits APPLIED:** §1 (`PlayerState.inscriptions` + Card reactive-trigger comment rewritten); §2A (`PlayCardAction` inscribe branch; `SubmitInterventionAction` invocation tag); §2B (new Sigils event table ×3; `CardPlayedEvent` suppression note); §2C (2 codes); §3 (hosting-table rows reworded; new **"Sigils"** subsection; ③′ carve-out resolved; "Still deferred" shrunk to marked-for-destruction scope; `IEventBus` owner-group order + auto/secret→inscription sweep ×6); §4 (step-2 reference; canonical-order row). Re-grepped: remaining "secret" mentions are 3 intentional decision notes.

**Plan impact:** Epic 01 — `inscriptions` field, `MaxInscriptions`, 2 rejection codes; Epic 02 — inscribe branch on the play handler, 3 sigil events, `CardPlayedEvent` suppression; engine epics — inscription registration/drop, dispatch slot, actor-attribution skip, random-degradation path (reuses random-K — no new selector work), empty-pool guard, one-shot consume; content epic — sigil definitions need NO mode-specific sections (the response + declared choices are mode-agnostic; degradation is engine-side); Game-Server/UX — face-down rendering, masking options.
**Tests:** opponent invocation on my turn springs my inscription (actor rule) / my own action never does / neutral-sourced springs both; duplicate + 4th inscribe reject; same-seed random degradation is deterministic and matches the held-mode selector pool; empty pool → stays inscribed; inscribed+held copies → auto fires then window offers the held one; chain of mutual inscriptions terminates ≤ row caps; inscribe play emits thin public + directed full, no `CardPlayedEvent`; ③′ declaration of an inscribe play is redacted.

## R2 inversion stat-math — PINNED (session 9 part 5, 2026-06-11) → ⛔ SUPERSEDED: inversion parked to V2 (2026-06-14)

> **The entire inversion mechanic was parked as a V2 feature on 2026-06-14** (session 11). The decision below remains the canonical stat-math *design*, but it builds nothing in v1 — it now lives in `notes/2026-06-14-inversion-v2.md` (the v2 seed), along with the still-open enchantment-flip question and the keyword offense↔defense pairing work. **#10c is moot** (the `sourceId: "inversion"` memento it concerned no longer exists in v1). The death-cadence dying-window is unaffected (R1 heal-to-save still justifies it). Read the v2 note, not this section, when inversion is revived.


The last item from amendment B's deferred list (deferred 2026-06-03 "to pin when the card is designed"; resurfaced by the session-9 deferral scan). While answering "what was the R2 stat-math?", the re-check found the deferral **undersold the gap**: `InvertTargetAction` was a bare table row — the spec had all inversion *plumbing* but never defined the stat arithmetic of an invert at all, even for undamaged minions; the unwritten ④ handler of a core v1 mechanic, blocking Epic implementation day-one. User picked from three candidates (clear-damage / damage-carries / direct value swap):

**✅ DECISION: Option 3 — `currentHealth` and `attack` trade places, on AURA-EXCLUSIVE values; auras re-applied after** (the re-apply clause costs nothing: the handler operates on aura-stripped values and the existing per-action ⑥ recalc rebuilds bonuses on the new orientation).

**Derived consequences (flagged + accepted):**
- **Enchantment deltas flip** (`attackDelta ↔ healthDelta`) and the **base pair swaps systemically** (the definition's inverted section defines effects/triggers only, never its own stat line) — both forced by self-consistency: they make post-invert `currentHealth = maxHealth` exactly. **An inverted minion always lands at full flipped health; damage persists as reduced attack, not as damage.**
- **The damage carrier is a system `StatModifier { attackDelta: −min(D, auraless maxHealth), sourceId: "inversion" }`** — attack stays derived, never negative. Re-inverting an unhealed minion trades the penalty back as a health delta (the wound returns; an R2-saved minion re-doomed if re-inverted before healing). **Silence forgives the traded damage** (it clears all enchantments incl. the memento) — accepted as flavor-consistent.
- **R2 × lethal works as the cadence decision envisioned:** mortally wounded (`H ≤ 0`) → invert in the dying window → full flipped health, 0 attack, survives ⑦ iff auraless attack `A > 0`. Worked examples: 4/5 @ 7 dmg → 4/4 full, attack 0, lives; 4/5 @ 2 dmg → 4/4 full, attack 3.
- `UnInvertTargetAction` = the same symmetric operation; mismatched `isInverted` state → no-op, no event. Card-in-hand inversion = flip minus the trade (no `currentHealth`).

**Plan impact:** the inversion/effects epic's `InvertTargetAction` ticket now has full ④ semantics to implement (trade + delta-flip + memento + no-op rule); data-model ticket: none (no new fields — the memento is an ordinary `StatModifier`). **Tests:** the two worked examples; invert→re-invert returns original `currentHealth`/attack (wound returns); silence-after-invert restores full flipped attack; aura +1/+1 present during invert: stripped before trade, re-applied after (⑥); mortally-wounded save iff `A > 0`; `UnInvert` on normal minion no-ops; card-side flip swaps `modifiers` deltas.

## Marked-for-destruction scope — RESOLVED, option A (session 9 part 6, 2026-06-12)

The last deliberately-deferred design question in the spec (point-D leftover). Re-grounding found a **latent trap**: `MinionMortallyWoundedEvent` — the dying-window hook — fired on destroy-marks too, so a mark would open the save window, offer heal/invert cards, and let the player waste one on an unsavable minion.

**✅ DECISION: Option A — the fiat doom is absolute and silent post-④.** User's understanding, confirmed as the rule: *MortallyWounded occurs only when `currentHealth` reaches ≤ 0 (damage, or aura/enchant loss collapsing `maxHealth` — both health-domain and both savable). Once a minion is marked for death there is no saving it — it dies at ⑦ no matter what; the only counterplay is preventing the mark (③′ interception of the `DestroyMinionAction`: cancel/retarget). Once applied, it's over.*

- **Event narrowed:** `MinionMortallyWoundedEvent` = health-domain only; a destroy-mark emits **no event** (silent until `MinionDiedEvent` at settle) — no futile window can open. Retro-strengthens the session-7 `cause` drop: with marks excluded, all remaining causes are health crossings, indistinguishable-by-need.
- **Two-channel counterplay grammar** (matches unanimous prior art — MTG counter-the-spell/"can't be regenerated"/indestructible, HS Counterspell, YGO pre-resolution chains, LoR damage<kill<obliterate tiers; the one counterexample, MTG regeneration, was deprecated after decades of escape-hatch text): **fiat dooms preventable at the cause (③′); health dooms savable at the consequence (⑥′/dying window until settle).**
- **Terminology pinned:** "pending death" = the shared lingering-on-board state (both kinds); "mortally wounded" = the health-domain savable kind only (§4 amendment-B narrative reworded).
- **Riders:** inverting a marked minion executes the part-5 trade but the mark survives (dies anyway — clause added to the `InvertTargetAction` row); both dooms can coexist (mark wins); `Alive…`/`Dead…` selectors stay **health-only, deliberately** — a marked healthy minion sits in `Alive…` (correct: `Dead…` is the save-targeting family; for AoE/counts/retaliation it behaves as the living presence it is); a future "cleanse the mark" = additive unmark action, rejected for v1 (Item-10 disposition).
- **Spec edits:** §1 `currentHealth` comment (savability split), §2A `DestroyMinionAction` (full ④ handler — was a bare row), §2B `MinionMortallyWoundedEvent` (narrowed), §3 selector-trichotomy note + InvertTargetAction clause + "Still deferred" block → "Nothing remains deferred", §4 amendment-B terminology.
- **Plan impact:** Epic 02 — `DestroyMinionAction` handler semantics (mark, no event); engine/death epic — ⑦ collection unchanged, window-opening reads the narrowed event; tests: mark at full health → no `MinionMortallyWoundedEvent`, no ⑥′ window, dies at settle; heal/invert on marked minion → resolves, mark survives, dies; ③′ cancel of a `DestroyMinionAction` → minion lives; marked minion retaliates but cannot initiate; marked+≤0 minion → one death, one graveyard entry; Poisonous proc is interceptable at ③′.

**With this, the spec has ZERO open design questions.** Every item is decided, recorded, or explicitly re-scoped to v2 (crafting). Pre-implementation work remaining: plan reconciliation only.

## Spec-review fix pass (session 10–13, 2026-06-12→16 — BATCH 1 + #11–#15 + Items 1–2 + Interturn ✅ APPLIED; #16/#17 ✅ APPLIED session 13; walk of #19–#33 pending)

Walking `notes/2026-06-12-spec-review-findings.md` finding-by-finding (user chose to review every item, fork or not; Q&A batches of ≤4). Status: **#1–#10b + #18 + #34 ✅ APPLIED (session 11); #10c MOOT (inversion parked to v2 — section below); #11 + #12 (both halves) + #13 + #14 + #15 + #22 ✅ APPLIED (session 12) together with two NEW user mechanics (Items 1 & 2) and the Interturn step; #35 NEW finding logged + reviewed → current behavior KEPT (revisit later, no spec edit); **#16 + #17 REVIEWED → PARKED** pending a foundational question (should the active player ever intervene on his own turn? — Model A attribution-based vs Model B turn-based); #19–#21/#23–#33 not yet walked.** Batch 1 grep-verified clean (no stale `isDamaged`/`SourceMinionId`/`MinionDamagedEvent`/`DeathrattleAction`/"hero-power damage"/"damaged enemy"/③ "is alive"/⑦ "④–⑥"); #11 batch grep-verified clean (no stale `repopulateOnTurnStart`/"renderer convenience"/turn-start neutral repopulation).

**▶ BATCH 1 APPLIED (session 11) — spec edits landed:** #1 §4③ target-validity ("aliveness is *not* checked here" + selector note); #2 `CardDrawnEvent`/`CardAddedToHandEvent` → **directed→owner** + new public-thin `CardDrawnPublicEvent`/`CardAddedToHandPublicEvent`, both directed events added to the §2B directed-list (+ `SigilInscribedEvent` folded in for completeness), `DrawCardAction`/`GiveCardAction` emit both, `HandOverflowEvent`/`CardDiscardedEvent` annotated public; #3 Poisonous "any minion it damages, heroes excepted"; #4 Lifesteal null-owner no-op; #5 `AttackDeclaredEvent` **bus-only**; #6 `isDamaged` field DELETED, Enrage pulls `currentHealth < maxHealth`; #7 §4⑤ publish-order gains the inscriptions slot; #9 `IAura.SourceMinionId`→`SourceEntityId` + zone-entry registration prose; #10a SpellDamageBonus "artifact-activation" (was "hero-power"); #10b §3 `ITriggerCondition` host = "minion, artifact, or sigil (in hand or inscribed)"; #18 §4⑦ Phase-2 = full-pipeline enqueue-and-drain; **#34 the two-action split** — `PlayCardAction` ④ rewritten to author the **normal-play commit** branch (pay/leave-hand/count/`CardPlayedEvent`+`SpellCastEvent`/spell→graveyard/enqueue `ResolveCardAction`) alongside the existing inscribe branch; **new `ResolveCardAction` §2A row** (its ③′ = counterspell window; ④ runs effects/summon+Battlecry; fizzle→`CardPlayFizzledEvent`, no refund); new `CardPlayFizzledEvent` §2B; `InterventionWindowOpenedEvent` gains **cause** (window-cause rider); §4 PendingIntervention "counter hooks `ResolveCardAction`, prevent-cast hooks `PlayCardAction`"; #8 trace example replaced (3 blocks: corrected combat w/ `DamageTakenEvent`+overkill+`DeathrattleTriggeredEvent`, fireball-countered, intercepted-deathrattle) + new **`fizzle` record kind** in the trace vocab.

**Decided (mechanical, as recommended):**
- **#1** drop "is alive" from §4 ③ target-validity — aliveness policy lives entirely in the declared selector (`All…/Alive…/Dead…`); attacker-aliveness check unchanged.
- **#2** draw/give events split directed-full + public-thin (`CardDrawnEvent`/`CardAddedToHandEvent`: owner sees the card, opponent sees "drew a card"); `HandOverflowEvent` burned card + `CardDiscardedEvent` stay public (HS reveals both).
- **#3** Poisonous = destroys **any minion it damages** (heroes excepted) — relational scope removed (lane model made "enemy" wrong for neutrals).
- **#4** Lifesteal from an unowned (neutral) source = explicit **no-op**.
- **#6** `isDamaged` **DELETED** — pull `currentHealth < maxHealth` at the read points (Enrage ⑥ recompute, damaged-target conditions); continues the stored-verdict purge (`canAttack`, `attacksAllowedThisTurn`).
- **#7** §4 ⑤ publish-order text gains the inscriptions slot (minions → artifacts → inscriptions), matching §3 `IEventBus` + the Ordering table.
- **#9** `IAura.SourceMinionId` → **`SourceEntityId`** + zone-entry registration wording (artifact hosts implement the literal contract).
- **#10a** `SpellDamageBonus`: "cannot leak into combat or **artifact-activation** damage".
- **#10b** `ITriggerCondition.sourceId` host list = "the hosting **minion, artifact, or sigil — in hand or inscribed**" (user correction: I'd missed inscriptions).

**#5 — `AttackDeclaredEvent` → BUS-ONLY (discussed, revised from my carve-out recommendation).** Treated exactly like `ActionDeclaredEvent`: the typed ③′ "when this attacks" hook, excluded from persisted log + wire. The "Bus ⊋ log / log = state-delta stream" invariant survives unchanged; no third visibility class. Renderer coverage verified: no-window → `AttackPerformedEvent` (④, committed) anchors animation with no perceivable gap; window → directed `InterventionPromptEvent` (held action verbatim) + the public window event (which now carries the cause, below). Spectator attack-arrow during a window rides the window-cause.

**#34 — NEW finding (user-found, the session's headliner): cancel was a FREE NO-OP — cost timing finally pinned via a two-action split.** User probe ("counterspelled fireball must still go to graveyard") exposed: payment + zone move lived in `PlayCardAction` ④; an interception cancel drops the action *before* ④ ⇒ no mana paid, card back in hand, **zero deltas** — Counterspell strictly worthless (actor re-submits the identical play), and the whole drama invisible to spectators + event-log replay. This was the long-deferred session-3 "cost timing" question with teeth. **✅ DECISION — option B, the two-action split (keeps mutations ④-only; user asked exactly where commitment mutates and rejected a pre-③′ mutation point):**
- `PlayCardAction` ④ = **commitment**: pay mana, card leaves hand, `cardsPlayedThisTurn++`, emit `CardPlayedEvent` + `ManaChangedEvent`, **enqueue `ResolveCardAction { cardId, definitionKey, playerId, targetId? }`** — the idiom `AttackAction` (enqueues damage) and `ActivateArtifactAction` (pays at ④, enqueues effects) already use; card plays were the odd one out.
- `ResolveCardAction` = ordinary queue citizen: **its ③′ is the counterspell window** (held action = the resolution); ④ runs effects (spell effects; minion summon + Battlecry — Battlecry moves one level deeper, still synchronous-in-pipeline); cancel/illegal-on-resume → **fizzle**: `CardPlayFizzledEvent {playerId, cardId}`, effects never run, **never refunds**.
- **Dissolves two patches made earlier the same session:** the resume's cost-gate-skip (held resolution has no cost) and the trace "action record written at commitment" rule (PlayCardAction traces as a normal completed ④).
- **Two counter-semantics now expressible by construction:** cancel at `PlayCardAction` ③′ = "prevent the cast" (nothing sunk, card stays in hand); cancel at `ResolveCardAction` ③′ = true Counterspell (costs sunk, card spent). Trigger condition picks. The pending `ResolveCardAction` *is* the MTG "spell on the stack" — no new zone; the queue is the zone.
- **Graveyard entries:** spell → written at the play's ④ (in graveyard during the window, both outcomes — user's pick); minion card → entry **on fizzle only** (resolved → the body's eventual `GraveyardMinion`; avoids double entries for one body).
- Riders: an **invocation** emits a plain `CardPlayedEvent` (inscribe redaction is inscribe-only); a fizzled **attack** stays eventless (no cost committed; inferable from `WindowClosed` + no `AttackPerformedEvent`); uniform-commitment for artifacts is moot — they already pay at ④ and enqueue effects.

**Window-cause (rider to #5/#34):** public `InterventionWindowOpenedEvent` gains the **cause** (held action verbatim at ③′ / `causeEventIds[]` at ⑥′; inscribe-redaction carve-out applies). Session-8 already ruled ③′ causes public-by-nature; the thin form hides the *responder's* candidates, and the cause is the *actor's*. Closes the spectator/replay gap for canceled no-delta actions.

**#18 — pulled forward (the deathrattle example forced it): ✅ enqueue-and-drain.** ⑦-enqueued deathrattle/reborn actions are ordinary queue citizens with the FULL pipeline (①–⑥ incl. ③′ + ⑥′) — what ⑨'s text already said; ⑦ Phase-2's "④–⑥ run for each" wording is the drift to fix. New mortal wounds defer to the next wave (already pinned), making deathrattle saves coherent; ⑧ still fires only at full settle.

**#8 — trace worked example: replace + extend.** Old example used nonexistent vocabulary (`MinionDamagedEvent`→`DamageTakenEvent`+real fields incl. `overkill`; `attacker/defender/src/tgt`→§2A names; **no `DeathrattleAction`** — a deathrattle traces as its effect's ordinary actions tagged `src`, + `DeathrattleTriggeredEvent` in the settle record; `isDamaged` diff entry dies with #6). Spec gets **both session-10 verified scenarios** (full four-stream audits done in-session: bus = log + declarations only; log = pure deltas; wires = log filtered by directedness; event-log + command-log replays both complete): (a) Fireball countered — commitment deltas → window(cause) → invocation → fizzle; (b) intercepted deathrattle — settle enqueues `DealDamageAction(src=dead minion)` → ③′ window → cancel. Adjust (a) to the B-shape (E1/E2 at `PlayCardAction` ④; held = `ResolveCardAction`).

**Trace schema additions:** new **`fizzle` record kind** `{action, reason: Canceled|Illegal(code), events}` — distinct from `reject`, which stays zero-event. (The commitment-write rule was adopted then dissolved by #34-B.)

**#10c — ⛔ MOOT (resolved by deferral, 2026-06-14, session 11).** The question (`StatModifier.sourceId` entity-ref vs the `"inversion"` sentinel; options: document-sentinel / `kind` field / typed union) existed *only* because of the inversion damage-memento. **Inversion was parked to V2 the same way**, so the memento — and the sentinel — no longer exist in v1. The question is preserved for v2 in `notes/2026-06-14-inversion-v2.md` §A (with a note that the re-invert branch is load-bearing → mild lean toward the `kind` field when revived). **Batch 1 is unblocked: apply #1–#10b + #18 + #34 (no #10c edit needed).**

### #11 match setup + Items 1 & 2 + the Interturn step ✅ APPLIED (session 12, 2026-06-14)

Walked #11 first (the keystone H hole — its constants gate #12/#19/#21/#22/#28). Most sub-parts had obvious HS defaults (user confirmed the batch); the live decisions were going-second compensation and two NEW mechanics the user added.

- **#11 match setup (APPLIED).** New **`GameConstants`** block §1 — single source for `StartingHeroHealth 30`, `StartingHandFirstPlayer 3` / `SecondPlayer 4`, `HandCap 10`, `MaxMana 10`, `StartingMana 1`, `DeckSize 30`, `MaxCopiesPerCard 2`, `NeutralFirstSpawnTurn 3`; also re-homes the previously-inline `MaxArtifacts`/`MaxInscriptions`/`MaxDeathWaves` (partial **#28** close). User flagged future growth: this block is the seam for config overrides + deck-conditional rules (e.g. "deck has card X ⇒ that player's mana cap +1"). New **Match Setup** §4 subsection (before Turn Lifecycle): heroes at 30/30, deck assert+Fisher-Yates shuffle, seeded coin flip, opening hands, **The Coin** for P2, opening-hand **sigil registration at `birthEpoch = 0`** (dealt outside the pipeline), **HS-faithful mulligan** (replace any subset; replacements drawn *then* replaced cards shuffled back ⇒ no immediate redraw), turn-1 mana seeded directly, neutral lane empty until turn 3. **`WaitingForPlayers`** added to the ② phase-guard table (→ `Mulligan` on the 2nd connect).
- **#12 (cap half, APPLIED).** Hero `health` init = max = `StartingHeroHealth`; `HealAction` caps a hero there (the minion "healed up to maxHealth" rule). **Entity-id-convention half deferred to the #12 walk.**
- **#22 (APPLIED).** `ModifyManaAction`: floor 0, **no ceiling** (a gain MAY exceed `maxMana` — The Coin); never touches `maxMana`.
- **Going-second = The Coin** (user pick, option A). A 0-cost card → `ModifyManaAction(+1)` on current mana; stands in for HS temp crystals (absent from this model). Settled #22 by construction.
- **Item 1 (NEW) — deck-build guaranteed opening card.** A decklist may pin **one** `guaranteedOpeningCardKey`; after shuffle, if it's absent from the opening-hand slice the engine **swaps the top-most copy in** — a pure post-shuffle position swap consuming **no `IRandom`** (user-confirmed; keeps the deal unbiased + replay exact). No mulligan protection; optional; both players.
- **Item 2 (NEW) — neutral lane first appears end of P1's 2nd turn, then refills to max.** Initial seed + recurring refill unified into ONE rule fired at the Interturn step.
- **Interturn step (NEW lifecycle step 5) — "the neutral lane's own turn" (user's reframe; replaced my seat-order proposal, which he rejected for giving P1 an arbitrary advantage).** Inserted between the ending player's cleanup (4) and the swap; renumbers old 5–11 → 6–12 (cross-refs at §1 `summoningSick`/`usesThisTurn` updated 6→7, 7→8). For its duration `turn.activePlayerId = none` (null) ⇒ neither player's trigger groups fire, **only the neutral group** (new **IEventBus exception**) — the exact dual of neutral minions being excluded from player turns. Refill-to-`maxCapacity` from `spawnPool` via `SpawnNeutralMinionAction{definitionKey=null}` (new **pool-draw mode** — draws at its own ④ `IRandom`) × (max−count). `NeutralZoneConfig.repopulateOnTurnStart` bool **replaced** by `firstSpawnTurn: int` (=3). `TurnState.activePlayerId` → nullable. Intended consequence: a player's on-summon (or other) reaction does **not** fire on a handover spawn (a neutral *card-spawned on a player's turn* still triggers normally — only the handover is walled).
- **Interturn = a RESERVED engine seam.** The only owned-by-neither, fair-dispatch, once-per-handover point in the loop; v1's sole inhabitant is neutral repopulation. Reserved design space (recorded so it's not lost): **(A)** neutral-lane life-cycle — upkeep/aging (grow uncontested / wither ignored), autonomous neutral mini-turn; **(B)** environmental/"field" — world/event deck (per-handover global modifier), board-wide doom clock; **(C)** shared resources/objectives — aether accrual (v2 crafting), contested-lane control scoring; **(D)** round pacing — a true round concept (the handover after P2 closes a round); **(E)** fairness-critical symmetric per-round triggers (seat-bias-free by construction).
- **#35 LOGGED + reviewed → KEPT as-is (user's call, revisit later).** The "AttackDeclaredEvent is a renderer convenience" correction surfaced a Stealth-timing question. Re-reading the handler resolved it: the pipeline already has **two** interception points with the reveal committing between them — `AttackAction` ③′ (prevent → Stealth survives) → `AttackAction` ④ (commits Stealth-break + budget) → strike `DealDamageAction` ③′ (counter the swing → Stealth already broke). So a swing-counter already breaks Stealth (it already mirrors the #34 prevent/counter spell duality); only a *declaration-cancel* preserves Stealth (recommended: keep, parallel to prevent-the-cast). **No restructure, no spec edit** — current behavior accepted, revisit if a card ever needs declaration-cancel to break Stealth. Full record: findings-doc **#35**.

### #12 (entity-id half) + #13 ✅ APPLIED (session 12, 2026-06-14)

Walked together right after #11. **#13 (HS-exact, user accepted as-is):** `currentHealth` follows `maxHealth` changes — rise +Δ (gap preserved), fall = clamp `min(currentHealth, newMax)` (damage persists, no minimum; ≤0 ⇒ mortally wounded); unified `newCurrent = min(currentHealth + max(0, Δmax), newMax)`; in §1 `maxHealth`, applied at BuffMinionAction/SilenceMinionAction/⑥. **#12 (entity-id half):** the user's probes ("is `playerId` an ID or just the index 0/1?", "do neutral minions have ids?") tightened the whole id model — **type-prefixed, prefix-disjoint allocators** (`m*` minions, **incl. neutral on the same allocator** — neutrality = `ownerId==null`, not a separate id-space, stable across `TakeControl`; `c*` cards; `a*` artifacts); heroes/players = the seats **`p1`/`p2`** (1-indexed to match `GameState.player1`/`player2` — the table's draft `p0`/`p1` would have offset player1→p0), a hero **is** its player (no separate hero id), `playerId` = match-local seat, never an account id. Unified the trace's three-way hero refs (`p1`/`h1`/`p2.hero`) → `p1`/`p2`. §1 Identity + §2A ID-note + §3 trace conventions. **#12 now fully closed** (cap half was #11). **Plan impact:** Epic 01 data-model — prefix-disjoint id allocators (neutral on the `m` allocator), seat-based `playerId`, hero=player addressing, entity-target resolution by prefix; #13 health-clamp shared by buff/silence/aura.

### #14 (Silence) + #15 (Transform) ✅ APPLIED (session 12, 2026-06-14)

The two sibling bare-row handlers, both HS-faithful. **#14 Silence ④:** clears `enchantments`/`intrinsicKeywords`/`grantedKeywords`/`grantedTribes`/`isFrozen`+`frozenOnTurn`/`rebornAvailable` and **unsubscribes** triggers/Deathrattle/emitted auras; stats recompute with `currentHealth` clamping per #13; does NOT touch `intrinsicTribes`/`summoningSick`/persisted damage/destroy-marks; `auraKeywords` + aura bonuses recompute at ⑥ (Silence ≠ immunity); emits `MinionSilencedEvent`. Micro-calls (user-approved): included `isFrozen` + `rebornAvailable` (both "loses its text" consequences). **#15 Transform ④:** full reset to `newDefinitionKey` (replacement, not death → **no Deathrattle**) — new base stats/keywords/tribes, enchants+grants cleared, `currentHealth = maxHealth` (damage+buffs wiped), `summoningSick = true`, `rebornAvailable` from new def, `summonOrder` reset, old triggers/auras unregistered + new ones registered with a **fresh `birthEpoch`**; **preserves** `minionId` (in-place, references stay valid), board position, `ownerId`/`bornNeutral`; emits `MinionTransformedEvent`. Inversion sub-questions on both (memento / `isInverted`-across-transform) moot — parked. **Plan impact:** Epic 01/data-model — Silence + Transform handlers (bus unsubscribe + fresh-epoch re-register, #13 clamp reuse); tests: silence-strips-all-but-base, transform-no-deathrattle + summoning-sick.

### #16 + #17 ✅ RESOLVED — Model B′ (turn-based responder) + unconditional two-stage windows (session 13, 2026-06-16, commit `3dd95e5`)

After a multi-turn design discussion (prior-art survey of MTG APNAP / YGO Quick-Effects / LoR reactive stack; a "reap-as-action" idea to host R1 saves at a death-removal ③′, explored then not taken; a user-authored delivery spec, reviewed and reworked), #16/#17 were resolved by a **full rework of the intervention delivery model**, not a point fix.

**Responder model (closes #16, dissolves #17):**
- The responder is **always the non-active player.** Interventions — both **inscriptions** and **invocations** — are live only off-turn; **no one** intervenes while `activePlayerId == null` (Interturn). Permanents (board/artifact/deathrattle/aura) are NOT interventions — they fire at ⑤ regardless of turn.
- **HS-faithful**: Hearthstone Secrets trigger only on the opponent's turn — *including* Effigy-class "when a friendly minion dies" secrets (a global HS rule), so off-turn gating loses **no** HS content. (My "Effigy-class boundary" concern was wrong and dropped.)
- Single-valued responder ⇒ #16's ordering / both-respond question dissolves; **depth-1 automatic**; **source-based actor attribution unnecessary** ⇒ #17 dissolves (own-draw/own-fatigue can't spring the owner's own cards — they're off). The `playerId`-fallback actor-classification is **dropped, not built.**
- DELETED: the source-based "never on your own action" inscription rule + its "punish an opponent's invocation during my turn" corollary (spec line 815); active-player self-intervention (Model A — never HS).

**Window-delivery model (the rework):**
- **Unconditional (structural) + two-stage**, reversing the contingent-window model: Stage-1 **decision** (`InterventionDecisionWindow` 1s; `RespondInterventionAction` skip/skip-all/intervene) opens at every ③′/⑥′ even with no match (existence uninformative); pressing intervene opens Stage-2 **extension** (`InterventionExtensionWindow` 3s; `SubmitInterventionAction` pick/back-out).
- **Press cap** `MaxInterventionsPerTurn = 3` (`TurnState.interventionsUsed`, reset at turn advance, ++ on escalation) — counts presses uniformly at ③′/⑥′; on top of mana; inscriptions exempt. Budget exhaustion suppresses the decision window (leak-free — escalations are public).
- `GamePhase` split `PendingIntervention` → `InterventionDecision | InterventionExtension`; Phase Guard two rows; `InterventionWindowOpenedEvent { phase, stage }` (Extension = explicit escalation tell, replaces the contingent tell).
- **Pacing:** active player's turn clock paused during windows; `PlayerState.autoSkipAll` toggle (server-read at window-open, replay records transitions). **Pinned/deferred:** per-turn real-time base-window budget (the slow-play vector the press-cap doesn't cover) — not built in v1 (Unaddressed Features); revisit on telemetry.

**Plan impact:** Epic 04/08 intervention tickets gain the two-stage window (`RespondInterventionAction` + re-scoped `SubmitInterventionAction`), press-cap, `autoSkipAll`, `stage`-bearing event; `GamePhase` split + Phase Guard two rows; new `GameConstants` (decision/extension/cap). The #17 actor-classification (queued from session-12 parking) is **REMOVED** from the backlog. Trace `window` record gains `stage`; worked examples (2)/(3) show the two-stage flow.

#### Prior parking (session 12, 2026-06-15) — superseded

Walked the responder definition (#16). Provisionally decided **Model A (attribution-based):** responder = any player the action is *not* attributed to; active player never responds to his own actions but **does** respond to non-own (neutral/opponent-sourced) actions mid-cascade on his turn; a neutral (unattributed) action → both respond, **active player first** (user-decided ordering, = the canonical active→opponent→neutral order). The §3 responder paragraph + inscription cross-ref were **applied then reverted** (`git checkout`, **not committed**) because the user raised a FOUNDATIONAL question: *should the active player EVER intervene on his own turn?* His framing — everything cascading during his turn is, by chain-of-causation, his own action, so perhaps he just deals with the consequences → **Model B (turn-based):** active player never intervenes on his turn; only the off-turn player gets windows; the neutral both-case collapses to a single (off-turn) responder; **#17 mostly dissolves** (active player can't react to his own draw/fatigue). Model B is leaner; Model A keeps active-player mid-cascade self-saves. **Resume: DECIDE the foundational question first, then apply #16 (+#17) accordingly.** Full record: findings-doc #16/#17 + session-state Session 12. (Spec clean at `208b573`.)

## Inversion parked to V2 (session 11, 2026-06-14)

User's call: **park the entire inversion mechanic as a V2 feature.** Triggered by walking the option-3 stat-math against a pre-existing enchantment (1/6 → +3 atk → 4/6 → 4 dmg → 4/2 → invert → **2/4**), which surfaced the unresolved enchantment-flip question and revealed how deep/ramifying inversion is (a novel mechanic with no HS reference). **v1 builds nothing for it.** Full footprint removed from spec §1/§2A/§2B/§3 + Identity (the `normal`/`inverted` definition split → v1 definitions are **flat**); Epic 14 marked ⛔ deferred (retained as the v2 seed); refs scrubbed from Epics 03/16 + README. Design preserved in `notes/2026-06-14-inversion-v2.md`: option-3 stat-math (full), the **open** enchantment-flip question (flip-deltas vs persist-label — first v2 question), the keyword offense↔defense pairing (4 native pairs incl. the systemically-consistent Divine Shield↔Poisonous and Enrage↔Lifesteal; involution + invented-anti-keyword open choices). **Dissolves #10c.** Death-cadence unaffected (R1 justifies the dying-window alone). **Plan impact:** Epic 14 out of the v1 build order; v1 card-definition schema is flat (no `inverted` section); no `isInverted`/invert actions/invert events/On-Invert trigger anywhere in v1.

**Plan impact (accumulating; fold into reconciliation):** `ResolveCardAction` = new §2A action → card-play epic ticket re-scope (commit/resolve split; Battlecry placement) + validator + engine pipeline tickets; #2 directed-split → events/wire epic; #6 `isDamaged` removal → data-model + Enrage tickets; #18 → death-wave ticket ("full pipeline for wave-enqueued actions"); trace epic ticket gains `fizzle` kind + both worked examples; #5/window-cause → events epic (`AttackDeclaredEvent` bus-only; `InterventionWindowOpenedEvent.cause`); tests: cancel-eats-card+mana, identical-resubmit impossible, pre-cast vs post-cast cancel distinction, deathrattle interception, countered-minion-card graveyard entry. **#11/Items/Interturn:** new `GameConstants` → Epic 01 data-model ticket; **Match Setup** → a match-init ticket (deck assert + size/copy validation, seeded coin flip, opening deal, The Coin, HS mulligan procedure, opening-hand sigil registration at birthEpoch 0, `WaitingForPlayers`→`Mulligan`) — Epic 01/02; **guaranteed-card pin** → match-init + decklist-input shape (`guaranteedOpeningCardKey`, no-RNG post-shuffle swap); **Interturn step** → Turn Lifecycle ticket (+ `TurnState.activePlayerId` nullable, IEventBus `active=none` ⇒ neutral-only dispatch, `SpawnNeutralMinionAction` pool-draw mode, `NeutralZoneConfig` `repopulateOnTurnStart`→`firstSpawnTurn`); **neutral-zone epic** gains the refill-to-max rule + the reserved-seam note (A–E future inhabitants); tests: P2-first-crack on the seed, walled-off handover (player on-summon does NOT fire), guaranteed-card present after deal, Coin pushes mana>maxMana.

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
