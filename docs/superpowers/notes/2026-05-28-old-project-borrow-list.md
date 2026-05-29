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

---

## Item 3 — Snapshot triggers before processing (registration-during-cascade safety)

**What the old project does:**
`TriggerProcessor` (TriggerProcessor.cs:65–107) **snapshots the trigger list at the start of an event batch.** Triggers registered mid-cascade (e.g. a deathrattle that summons a minion with its own trigger) do **not** fire on events already in flight — they only catch *subsequent* events in the next batch.

**Why it might matter for our spec:**
Our spec §3 `IEventBus` says "publish ordering — current player's list first, then opponent's; sorted by current board index at publish time." It doesn't say *what list*: the live subscriber list at the moment of publish, or a snapshot taken at the start of the publish batch. These differ exactly when a trigger handler installs a new trigger while the bus is iterating. Worth being explicit in the spec.

**Recommendation: Adopt and document.** Add a sentence to §3 `IEventBus`: *"Subscribers for a given event are snapshotted at publish time. Listeners registered during dispatch of that event do not receive it; they receive subsequent publishes only."* Same applies during the death-wave Phase 2 — new triggers from reborn minions don't retroactively fire on deaths already published this wave.

**Open questions:**
- Does the spec want the snapshot at *individual event publish* or at *action-handler batch start*? Different answer for "does an event fired during the action see triggers from a minion summoned earlier in the same action."

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

---

## Item 9 — `PlayContext → EffectContext` conversion / nested contexts

**What the old project does:**
`PlayContext` is the higher-level context held by `PlayCardResolver` (caster, target, services). When an effect needs to fire, `PlayContext.ToEffectContext()` (PlayContext.cs:44–45) produces a lighter `EffectContext` for the op chain. Clean separation: high-level resolvers don't pollute low-level ops with all their fields.

**Why it might matter for our spec:**
Our spec defines `EffectContext` in §3 (`sourceCardId, sourcePlayerId, targetId?, read-only GameState`). It doesn't define how this is constructed inside an action handler, or how nested effects inherit context. A trigger that fires during another effect's cascade — what's its `EffectContext.sourcePlayerId`? The original caster, the trigger source's owner, or the active player? Not specified.

**Recommendation: Adopt as a clarification.** Add to §3 a paragraph: *"Each action handler constructs an `EffectContext` from its action's fields. Triggers fired during an action's event cascade construct their own `EffectContext` from the trigger source minion (not the original action's caster). Nested effects fired by triggers always identify themselves as the trigger source, not the originating action."* This determines who gets credited (e.g., for stats), who gets the spell-damage bonus, and so on.

**Open questions:**
- This decision affects "if Yeti's deathrattle damages a minion, who is the *source* of that damage — Yeti, or the spell that killed Yeti?" Real gameplay implications (poisonous, lifesteal attribution). Recommend: Yeti.

---

## Item 10 — `IEffect` op `Results` ledger as a debugging / tooling target

**What the old project does:**
Not present. Effects fire and forget.

**Why it might matter for our spec:**
If §3 `IEffect.Execute` writes nothing observable beyond events, a "why did this card not fire its second effect" debug session is a search through the event log. A small ledger — one entry per op invocation with `{opType, succeeded, summary}` — would massively shorten that loop. Same ledger feeds tooling (replay viewer, card-design preview).

**Recommendation: Keep on radar; defer.** Don't bake it into the v1 spec; the event log is sufficient for v1 cards. Revisit when we hit a card whose debugging would have benefited.

**Open questions:** none for now.

---

## Item 11 — Game engine builder pattern

**What the old project does:**
`GameBootstrapper` (GameBootstrapper.cs:32–70) wires up everything manually inside one MonoBehaviour. Order matters; services are constructed inside `GameEngine` constructor. Works fine for one-shot setup.

**Why it might matter for our spec:**
For tests, the scenario builder will need to construct an engine in many slightly different configurations (different starting mana, different RNG seeds, mocked aura services, custom card libraries). A small builder makes this readable.

**Recommendation: Keep on radar; let the test framework drive it.** Don't put a builder in the spec — that's an implementation detail. The scenario builder helpers from §11 of our plan (T1.7) will produce one organically.

**Open questions:** none.

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

---

## Item 13 — Command-log replay vs. event-log replay

**What the old project does:**
Neither is implemented (both `GameSnapshot.cs` and `EventLog.cs` are stubs). The investigation flagged this as "design fresh." Worth choosing the strategy in the spec.

**Why it might matter for our spec:**
The plan's Epic 16 backlog mentions "Event-log replay — reconstruct GameState from the event stream." That's only sound if every state change has a corresponding event (currently true). The alternative — replaying the command/action log — is also sound *if* the engine is deterministic (which requires Item 6 above to be done).

**Recommendation: Keep on radar; decide before implementing Epic 16 backlog item.** Pre-decision: **command-log replay is simpler and faster** because the event log can be regenerated from the command log + seed; the reverse is not true (events don't encode random rolls). Probably we want both: command-log for canonical replay, event-log as the wire format to clients.

**Open questions:**
- Single sequence per game, or per-player streams? Single is simpler.

---

## Topics deliberately omitted

These came up in the investigation but don't merit changes:

- **Uniform ID model (single counter for cards and minions).** Old project does this; our spec deliberately splits them. The split was justified by the Card-vs-Minion type distinction and survives every scenario we walked through (tokens, copies, transforms, resurrection, return-to-hand with policies). No change.
- **Animation playback layer.** Belongs to the client (Unity/Godot), not `CCG.GameLogic`. Out of scope.
- **Multiplayer scaffolding.** Belongs to the Game Server (C# / ASP.NET Core), not `CCG.GameLogic`. Out of scope.
- **AI / Bot scaffolding.** Whole subsystem — defer to a future epic after the core lands. Item 5 (`GetLegalActions`) is the only piece the library needs to expose for bots to be possible.

---

## How to use this file

Pick one item at a time. For each: confirm whether it goes into the spec, into the plan, or gets rejected. When a decision is made, this file gets updated with the decision and the spec/plan is amended in the same session.
