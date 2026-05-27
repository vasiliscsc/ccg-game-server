# Epic 01 — Foundation & Core Engine

**Goal:** Stand up the solution and the minimal-but-complete engine pipeline, proven end-to-end by a single working action: `EndTurn`. After this epic, the architecture (pipeline → handler → event bus → dequeue) exists and is testable; later epics only add handlers, keywords, triggers, and state fields.

**Outcome demo:** A two-player `GameState`, player 1 submits `EndTurnAction`, the engine validates it, runs the handler, publishes `TurnEndedEvent` then `TurnStartedEvent`, and the active player flips to player 2.

---

## T1.1 — Solution & project scaffold

**Goal:** Create the .NET solution, library project, and test project with packages, building green.

**Depends on:** none.

**Files (create):**
- `CCG.sln`
- `CCG.GameLogic/CCG.GameLogic.csproj` (`net10.0`, `Nullable=enable`, `ImplicitUsings=enable`)
- `CCG.GameLogic.Tests/CCG.GameLogic.Tests.csproj` (xUnit + FluentAssertions; project reference to the lib)

**Scope in:** project files, package refs, a trivial smoke test that asserts `true`. **Out:** any game types.

**Steps / verification:**
- `dotnet new sln -n CCG`
- `dotnet new classlib -n CCG.GameLogic -f net10.0`
- `dotnet new xunit -n CCG.GameLogic.Tests -f net10.0`
- `dotnet add CCG.GameLogic.Tests package FluentAssertions`
- `dotnet add CCG.GameLogic.Tests reference CCG.GameLogic`
- `dotnet sln add CCG.GameLogic CCG.GameLogic.Tests`
- Delete the template `Class1.cs` / `UnitTest1.cs`; add `Smoke_Builds` test.
- **Verify:** `dotnet test` → 1 passing.

**Notes:** Enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in the lib to keep the codebase clean. Commit.

---

## T1.2 — Minimal data model

**Goal:** The smallest state needed for turn-passing.

**Depends on:** T1.1.

**Files (create):**
- `CCG.GameLogic/State/GameState.cs`
- `CCG.GameLogic/State/PlayerState.cs`
- `CCG.GameLogic/State/TurnState.cs`
- `CCG.GameLogic/State/Enums.cs`

**Shape:**
- `GameState`: `SessionId`, `Player1`, `Player2`, `Turn`, `Phase`. Add a `GetPlayer(string id)` helper returning `PlayerState?` (used by the event bus later). Add `Opponent(string id)` helper too.
- `PlayerState`: `PlayerId`, `HeroClass`, `Health` (default 30). (Mana/board/hand added in later epics — keep minimal.)
- `TurnState`: `ActivePlayerId`, `Number` (default 1).
- `Enums`: `GamePhase { Mulligan, WaitingForPlayers, InProgress, PendingChoice, PendingIntervention, Ended }`.

**Scope out:** mana, board, hand, deck — added when first needed (Epics 02/04).

**Verify:** builds. No test yet (pure data). Commit.

**Notes:** State types are mutable classes (engine mutates them). Keep helpers (`GetPlayer`/`Opponent`) on `GameState` since many handlers need them.

---

## T1.3 — Action & event base types

**Goal:** Define the command/fact base records and the first concrete pair.

**Depends on:** T1.2.

**Files (create):**
- `CCG.GameLogic/Actions/GameAction.cs` — `abstract record` with `string? SourcePlayerId`, `DateTime RequestedAt = UtcNow`.
- `CCG.GameLogic/Actions/EndTurnAction.cs` — `record EndTurnAction : GameAction { required string PlayerId }`.
- `CCG.GameLogic/Events/GameEvent.cs` — `abstract record` with `DateTime OccurredAt = UtcNow`.
- `CCG.GameLogic/Events/LifecycleEvents.cs` — `TurnStartedEvent`, `TurnEndedEvent` (each `{ required string ActivePlayerId; required int TurnNumber }`).

**Verify:** builds. Commit.

**Notes:** `OccurredAt`/`RequestedAt` are timestamps that will break value-equality in tests — the assertion helper in T1.7 excludes them. Keep events grouped by domain in files (`LifecycleEvents.cs`, etc.) rather than one-file-per-event to limit file sprawl.

---

## T1.4 — Core interfaces

**Goal:** The three contracts the engine is built on.

**Depends on:** T1.3.

**Files (create):**
- `CCG.GameLogic/Interfaces/IActionHandler.cs` — `IActionHandler<in TAction> where TAction : GameAction { IEnumerable<GameEvent> Handle(TAction action, GameState state); }`
- `CCG.GameLogic/Interfaces/IEventBus.cs` — `Subscribe<TEvent>(string listenerId, string? ownerId, Action<TEvent, GameState> handler)`, `Unsubscribe(string listenerId)`, `Publish<TEvent>(TEvent evt, GameState state)`, `IReadOnlyList<GameEvent> PublishedEvents`, `ClearPublishedEvents()`.
- `CCG.GameLogic/Interfaces/IActionQueue.cs` — `Enqueue`, `EnqueueFront`, `bool TryDequeue(out GameAction? action)`, `bool IsEmpty`.

**Verify:** builds. Commit.

**Notes:** `ownerId` on `Subscribe` is what lets the bus order subscribers by owning player at publish time (spec §3 IEventBus). `null` ownerId = system subscription, ordered last.

---

## T1.5 — EventBus implementation

**Goal:** Inspectable bus that fires subscribers in deterministic order.

**Depends on:** T1.4.

**Files (create):** `CCG.GameLogic/Infrastructure/EventBus.cs`; **Test:** `CCG.GameLogic.Tests/Scenarios/EventBusScenarios.cs`.

**Behaviour:**
- Keeps an appended `PublishedEvents` log (every `Publish` records the event) — this is the "inspectable bus" the design calls for.
- Subscribers stored per event `Type`. On `Publish`, fire in order: **current player's subscribers first** (`ownerId == state.Turn.ActivePlayerId`), then opponent's, then system (`ownerId == null`); within a player, **by current board index** (looked up in `state` at publish time). For this epic there are no minions, so the ordering code exists but is exercised more in Epic 04+.
- `Unsubscribe(listenerId)` removes all subscriptions with that id.

**Scenario tests:**
- *Publish records to log:* publish two events → `PublishedEvents` contains both in order.
- *Subscriber fires:* subscribe a counter to `TurnStartedEvent`, publish one → handler ran once.
- *Unsubscribe stops delivery:* subscribe then unsubscribe → publish → handler not called.
- *Current-player ordering:* subscribe two handlers with different `ownerId`s recording call order; set active player; publish → active player's handler fired first. (Board-index tie-break is covered in Epic 04 once boards exist.)

**Notes:** Board-index lookup must be null-safe (no `board` field until Epic 04 — until then treat index as `int.MaxValue`). Implement the lookup behind a small private method so Epic 04 just fills in the board access.

---

## T1.6 — ActionQueue implementation

**Goal:** FIFO queue with front-insertion for death-resolution priority.

**Depends on:** T1.4.

**Files (create):** `CCG.GameLogic/Infrastructure/ActionQueue.cs`; **Test:** `CCG.GameLogic.Tests/Scenarios/ActionQueueScenarios.cs`.

**Behaviour:** back a `LinkedList<GameAction>`. `Enqueue` = AddLast, `EnqueueFront` = AddFirst, `TryDequeue` pulls First.

**Scenario tests:** FIFO order for `Enqueue`; `EnqueueFront` jumps the line; `TryDequeue` on empty returns false; `IsEmpty` reflects state.

**Notes:** trivial but tested so later death-resolution ordering rests on a verified primitive. Commit.

---

## T1.7 — ScenarioBuilder + assertion helpers

**Goal:** The shared test harness every future scenario test uses.

**Depends on:** T1.2, T1.3.

**Files (create):**
- `CCG.GameLogic.Tests/Helpers/ScenarioBuilder.cs`
- `CCG.GameLogic.Tests/Helpers/EventExtensions.cs`

**`ScenarioBuilder`:** fluent, `static NewGame()`, `WithPlayer1(id)`, `WithPlayer2(id)`, `WithActivePlayer(id)`, `WithPhase(phase)`, `BuildState()` → `GameState`. Designed to grow: later epics add `WithMana`, `WithCardInHand`, `WithMinion`, etc. Default players `"player1"`/`"player2"`, 30 health, phase `InProgress`, turn 1.

**`EventExtensions.ShouldMatchInOrder(this IReadOnlyList<GameEvent> actual, params GameEvent[] expected)`:** asserts count equal, then each element `BeEquivalentTo` expected **excluding `OccurredAt`** (and any future timestamp). Provides clear per-index failure messages.

**Verify:** a tiny self-test of `ShouldMatchInOrder` (matching + mismatching cases). Commit.

**Notes:** This ticket is the linchpin of the whole "scenario test" methodology the user wants — invest in making it ergonomic. Consider an overload that also accepts the engine and submits in one line later, but keep it simple now.

---

## T1.8 — GameEngine pipeline + EndTurn (the vertical slice)

**Goal:** Wire the full pipeline and prove it with `EndTurn`.

**Depends on:** T1.5, T1.6, T1.7.

**Files (create):**
- `CCG.GameLogic/Engine/PhaseGuard.cs` — `static Validate(GameAction, GameState)`; throws `InvalidActionException` if phase disallows the action. This epic: reject everything when `Phase == Ended`.
- `CCG.GameLogic/Engine/ActionValidator.cs` — `static Validate(GameAction, GameState)`; this epic: `EndTurnAction.PlayerId` must equal `Turn.ActivePlayerId`.
- `CCG.GameLogic/Engine/InvalidActionException.cs`
- `CCG.GameLogic/Engine/GameEngine.cs` — holds `GameState` + `EventBus` + `ActionQueue` + handler registry (`Dictionary<Type, Func<GameAction, IEnumerable<GameEvent>>>`); `Register<TAction>(IActionHandler<TAction>)`; `Submit(action)`.
- `CCG.GameLogic/Engine/GameEngineFactory.cs` — `Create(GameState)` builds engine and registers all handlers (this epic: just `EndTurnHandler`). Updated each epic.
- `CCG.GameLogic/Handlers/EndTurnHandler.cs`
- **Test:** `CCG.GameLogic.Tests/Scenarios/TurnFlowScenarios.cs`

**`Submit` flow (this epic's subset of the 9-stage pipeline):**
1. `PhaseGuard.Validate` (stage ②)
2. `ActionValidator.Validate` (stage ③)
3. `Enqueue(action)`, then loop: `TryDequeue` → dispatch to handler (stage ④) → collect events → `Publish` each (stage ⑤) → repeat until queue empty (stage ⑨).
4. Return all collected events.

Stages ⑥ aura / ⑦ death / ⑧ win are **stubbed/absent** here and slotted in by later epics (Epic 04 adds death + win, Epic 09 adds aura). Leave clearly marked extension points (private methods `RecalculateAuras`/`ResolveDeaths`/`CheckWinCondition` that are no-ops for now, OR omit and add in Epic 04 — pick one and note it).

**`EndTurnHandler.Handle`:** emit `TurnEndedEvent(active, number)`; flip `Turn.ActivePlayerId` to the other player; `Number++`; emit `TurnStartedEvent(newActive, newNumber)`. (Mana/draw on turn start come in Epic 02.)

**Scenario tests:**
- *EndTurn flips active player:* p1 active, turn 1 → submit `EndTurn(p1)` → events `[TurnEnded(p1,1), TurnStarted(p2,2)]`; state active = p2, number = 2.
- *Wrong player rejected:* p1 active → `EndTurn(p2)` throws `InvalidActionException` ("not the active player"); state unchanged.
- *Alternation:* `EndTurn(p1)` then `EndTurn(p2)` → active back to p1, number = 3.
- *Ended phase rejects:* phase `Ended` → any `EndTurn` throws.

**Notes:** Decide the extension-point convention now and document it at the top of `GameEngine.cs` so Epic 04 knows exactly where death/win slot in. Commit after green.

---

## Epic 01 done-when

- `dotnet test` green with all T1.5–T1.8 scenario tests.
- Pipeline stages ②③④⑤⑨ implemented; ⑥⑦⑧ marked as extension points.
- `GameEngineFactory.Create` is the single composition root used by tests.
