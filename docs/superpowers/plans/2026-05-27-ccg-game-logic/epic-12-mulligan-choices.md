# Epic 12 — Mulligan & Choices

**Goal:** The interactive-pause machinery. First the Mulligan phase (pre-game hand selection), then the general `PendingChoice` system (pipeline halts, waits for player input, resumes via a stored continuation) used by Discover and mid-effect targeting.

**Depends on:** Epic 02 (deck/hand/draw), Epic 08 (effects enqueue actions — continuations are queued effects), Epic 01 (phases, PhaseGuard).

---

## T12.1 — Mulligan phase + start

**Goal:** Game opens in Mulligan; each player is shown their opening hand options.

**Files (create):** `State/MulliganState.cs` (`bool Player1Completed`, `bool Player2Completed`), `Events/LifecycleEvents.cs` add `MulliganStartedEvent { string PlayerId; List<Card> Options }` (fired once per player — only their own options). **Modify:** `GameState` (`MulliganState? MulliganState`), game-start/init path (deal opening hands, set `Phase = Mulligan`, emit a `MulliganStartedEvent` per player), `ScenarioBuilder` (`InMulligan(...)` to seed). **Test:** `Scenarios/MulliganScenarios.cs`.

**Scenario tests:**
- *Two start events, own options only:* game init → exactly two `MulliganStartedEvent`s, each with that player's hand options and not the opponent's.
- *Phase is Mulligan:* normal actions (PlayCard/EndTurn) rejected by `PhaseGuard` during Mulligan.

**Notes:** opening hand sizes (first vs second player + Coin) are a game-mode/init concern — keep configurable; default to a fixed count for tests. The "Coin" card for the second player can be a later refinement; note it.

---

## T12.2 — SubmitMulligan + resolve

**Goal:** Players choose which cards to replace; once both done, the game begins.

**Depends on:** T12.1.

**Files (create):** `Actions/SubmitMulliganAction.cs` (`{ required string PlayerId; required List<string> CardIdsToKeep }`), `Handlers/SubmitMulliganHandler.cs`. **Modify:** `PhaseGuard` (only `SubmitMulligan` allowed in Mulligan), `Events` add `MulliganCompletedEvent { string PlayerId }`. **Test:** `MulliganScenarios.cs`.

**Handler:** replace non-kept cards (shuffle them into deck, draw replacements — replacements drawn before returns are shuffled, Hearthstone-style, to avoid redraw), mark player completed, emit `MulliganCompletedEvent`. When both completed: set `Phase = InProgress`, set starting mana, emit `TurnStartedEvent` for the first player + their first draw (coordinate with Epic 02 turn-start init).

**Scenario tests:**
- *Keep all:* keep entire hand → hand unchanged, completed.
- *Replace one:* keep 2 of 3 → the dropped card returns to deck and a replacement is drawn; hand size preserved.
- *Both done → InProgress:* after both submit → phase `InProgress`, first `TurnStarted` emitted, normal actions now allowed.
- *Wrong phase:* `SubmitMulligan` after game started → rejected.

**Notes:** lock the redraw rule (draw replacements first, then shuffle returns back) so a returned card can't be immediately redrawn. Document.

---

## T12.3 — PendingChoice infra + StartChoice

**Goal:** Generic mechanism to pause mid-resolution for player input.

**Depends on:** Epic 08 (queued effects as continuation).

**Files (create):** `State/PendingChoice.cs` (`string WaitingPlayerId`, `ChoiceType ChoiceType`, `List<ChoiceOption> Options`, `JsonElement Context`), `State/Enums.cs` add `ChoiceType { Discover, Target, Mulligan, Craft }`, `Actions/StartChoiceAction.cs`, `Events/ChoiceEvents.cs` add `ChoiceStartedEvent { string WaitingPlayerId; ChoiceType ChoiceType; List<ChoiceOption> Options }`. **Modify:** `GameState` (`PendingChoice? PendingChoice`), `GameEngine.Submit`/`PhaseGuard` (entering `PendingChoice` halts the queue; only `SubmitChoice` from the waiting player allowed). **Test:** `Scenarios/ChoiceScenarios.cs`.

**Mechanism:** a `StartChoiceAction` handler sets `Phase = PendingChoice`, stores `PendingChoice` (including a **continuation**: the remaining effect actions serialized into `Context`), emits `ChoiceStartedEvent`, and the engine stops draining the queue. The continuation representation: store the not-yet-processed queued actions (or a re-buildable descriptor) in `Context`.

**Scenario tests:**
- *Choice halts pipeline:* an effect that starts a choice → `Phase == PendingChoice`, `ChoiceStarted` emitted, no further effects resolved yet.
- *Only waiting player, only SubmitChoice:* other actions / other player rejected.

**Notes:** continuation storage is the tricky bit. Pragmatic approach: drain-and-capture remaining `IActionQueue` contents into the `PendingChoice` when a choice starts; restore them on resume. Decide & document the serialization (in-memory action list reference is fine for a single-process engine — `Context` JSON only needed if state must be persisted/transmitted; note this for the Game Server layer).

---

## T12.4 — SubmitChoice + resume

**Goal:** Apply the selection and continue the paused resolution.

**Depends on:** T12.3.

**Files (create):** `Actions/SubmitChoiceAction.cs` (`{ required string PlayerId; required string ChoiceId; required string SelectedOptionId }`), `Handlers/SubmitChoiceHandler.cs`, `Events/ChoiceEvents.cs` add `ChoiceResolvedEvent { string WaitingPlayerId; string SelectedOptionId }`. **Test:** `ChoiceScenarios.cs`.

**Handler:** validate the option; apply the choice's consequence (e.g. add the discovered card to hand, or fill the chosen target into the continuation); emit `ChoiceResolvedEvent`; set `Phase = InProgress`; re-enqueue the stored continuation; the engine resumes draining.

**Scenario tests:**
- *Resume after choice:* choice → submit selection → continuation effects now resolve; final state reflects both the choice and the post-choice effects.
- *Invalid option rejected:* selecting an option not in `Options` → throws; phase stays `PendingChoice`.

---

## T12.5 — Discover

**Goal:** Present 3 options, player keeps 1.

**Depends on:** T12.3, T12.4.

**Files (create):** `Effects/DiscoverEffect.cs`, `State/ChoiceOption.cs` (`string OptionId`, `Card? Card`, `JsonElement? Payload`). **Test:** `Scenarios/DiscoverScenarios.cs`.

**Mechanism:** `DiscoverEffect.Execute` enqueues a `StartChoiceAction` with 3 generated `ChoiceOption`s (cards). On `SubmitChoice`, the chosen card is added to hand.

**Scenario tests:**
- *Discover adds chosen card:* discover among [A,B,C], pick B → hand gains B, `ChoiceStarted`(3 options) then `ChoiceResolved` then `CardAddedToHand`.
- *Options count:* exactly 3 options presented.

---

## T12.6 — Mid-effect targeting

**Goal:** An effect that pauses to ask for a target, then continues using it.

**Depends on:** T12.3, T12.4.

**Files (create):** a `TargetChoiceEffect` (or reuse `StartChoiceAction` with `ChoiceType.Target` listing valid targets). **Test:** extend `ChoiceScenarios.cs`.

**Scenario tests:**
- *Targeted mid-resolution:* effect "deal 2, then if you choose a minion, freeze it" → pauses for target → submit → freeze applies to the chosen minion; both the pre-choice and post-choice parts resolve correctly in order.
- *No legal target → skip:* if no valid targets, the choice auto-resolves/skips (define behaviour; recommend skip with no `ChoiceStarted`).

---

## Epic 12 done-when

- Game opens in Mulligan with per-player option events; mulligan resolves (with locked redraw rule) and transitions to InProgress with first turn started.
- `PendingChoice` halts and resumes the pipeline via a continuation; Discover and mid-effect targeting both work end-to-end.
- All phase-gating around choices enforced by `PhaseGuard`.
