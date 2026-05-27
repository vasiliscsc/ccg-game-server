# Epic 13 — Intervention System

**Goal:** The single-response window: when one player declares a qualifying action, the other player gets one chance to play a card (or skip) before the held action resolves. Implements the `PendingIntervention` phase, the held action, the timer, and the response/skip/timeout paths.

**Depends on:** Epic 12 (pause/resume machinery is conceptually similar — reuse patterns), Epic 05 (attacks are the canonical trigger), Epic 08 (effects).

---

## T13.1 — PendingIntervention + open window

**Goal:** Hold a declared action and open a response window for the opponent.

**Files (create):** `State/PendingIntervention.cs` (`string RespondingPlayerId`, `GameAction HeldAction`, `int TimeoutSeconds`), `Actions/StartInterventionAction.cs` (`{ required string RespondingPlayerId; required GameAction HeldAction; required int TimeoutSeconds }`), `Handlers/StartInterventionHandler.cs`, `Events/ChoiceEvents.cs` add `InterventionWindowOpenedEvent { string RespondingPlayerId; int TimeoutSeconds }`. **Modify:** `GameState` (`PendingIntervention? PendingIntervention`), `PhaseGuard` (in `PendingIntervention`, only `SubmitInterventionAction` from the responding player). **Test:** `Scenarios/InterventionScenarios.cs`.

**Mechanism:** the engine (or a trigger) emits `StartInterventionAction` *instead of* immediately processing the qualifying action; handler sets `Phase = PendingIntervention`, stores `HeldAction`, emits `InterventionWindowOpenedEvent`, halts the queue. **Exact trigger conditions** for when a window opens are a higher-level/Game-Server concern (spec leaves TBD) — for `CCG.GameLogic`, expose the mechanism and let a configurable rule decide. For tests, trigger it explicitly.

**Scenario tests:**
- *Window halts resolution:* declare an attack that opens intervention → `Phase == PendingIntervention`, `InterventionWindowOpened(opponent)`, held attack not yet resolved.
- *Gating:* only responding player's `SubmitInterventionAction` accepted; declarer's actions rejected meanwhile.

**Notes:** `HeldAction` is a full `GameAction` stored in state — simple in-process reference. Persistence/serialization is the Game Server's problem; note the seam.

---

## T13.2 — Respond with a card

**Goal:** The responder plays one card, which resolves before the held action.

**Depends on:** T13.1.

**Files (create):** `Actions/SubmitInterventionAction.cs` (`{ required string PlayerId; string? CardId }` — null = skip), `Handlers/SubmitInterventionHandler.cs`, `Events/ChoiceEvents.cs` add `InterventionPlayedEvent { string RespondingPlayerId; string CardId }`, `InterventionWindowClosedEvent { string RespondingPlayerId; bool WasSkipped }`. **Test:** `InterventionScenarios.cs`.

**Handler (card path):** validate the responder owns/can afford the card; set `Phase = InProgress`; emit `InterventionPlayedEvent`; enqueue the responder's `PlayCardAction` **first**, then re-enqueue the `HeldAction`; emit `InterventionWindowClosedEvent(skipped:false)`; resume draining. Net effect: response card fully resolves, then the held action resolves against the updated state.

**Scenario tests:**
- *Counter before resolution:* p1 attacks p2's 2/2 with a 3/3; p2 intervenes by playing a buff making it 2/5 → buff resolves first → attack then trades into a 5-health minion (survives). Order: `InterventionPlayed → (buff effects) → AttackResolved/DamageTaken → InterventionWindowClosed`? Lock the exact sequence and document.
- *Held action sees new state:* the held action resolves against post-response state (e.g. target gained Divine Shield → attack pops shield).

**Notes:** decide whether `InterventionWindowClosed` fires before or after the held action resolves. Recommend: close the window (phase back to InProgress) *before* re-enqueuing, so closed fires logically at window end, then card + held action resolve. Pin with tests.

---

## T13.3 — Skip / timeout

**Goal:** No response (explicit skip or timeout) → held action resolves directly.

**Depends on:** T13.1.

**Files (modify):** `SubmitInterventionHandler` (null `CardId` = skip), and a timeout entry point (the engine exposes a `ResolveInterventionTimeout()` or accepts a system `SubmitInterventionAction` with skip — the actual timer lives in the Game Server; `CCG.GameLogic` just needs the "skip" resolution). **Test:** `InterventionScenarios.cs`.

**Scenario tests:**
- *Explicit skip:* responder submits skip → `InterventionWindowClosed(skipped:true)`, held action resolves unchanged.
- *Timeout = skip:* invoke the timeout/skip path → same as explicit skip.

**Notes:** keep timing out of the pure logic library — model timeout as an injected "skip" resolution. Document that the Game Server owns the wall-clock timer and calls the skip path on expiry.

---

## Epic 13 done-when

- A held action can be paused behind an intervention window; the responder may play one card (resolving first) or skip/timeout (held action resolves directly).
- Held action always resolves against the post-response state; event ordering for played/skipped paths is locked by tests.
- The library exposes the mechanism; trigger conditions and the wall-clock timer are left to the Game Server (documented seams).
