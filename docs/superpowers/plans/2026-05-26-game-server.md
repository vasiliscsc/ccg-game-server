# Game Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the authoritative CCG Game Server — a .NET 8 / ASP.NET Core service that hosts WebSocket game sessions, processes player actions through the `CCG.GameLogic` rules engine, and broadcasts scoped events to players and spectators.

**Architecture:** Two C# projects: `CCG.GameLogic` (pure rules library — immutable state, no framework deps, fully unit-testable) and `CCG.GameServer` (ASP.NET Core host — WebSocket connections, session lifecycle, turn timers, reconnection, result reporting). All game state is immutable (C# records with `with` expressions). Actions are serialised through a per-session `Channel<GameAction>`, ensuring single-threaded state mutation per game.

**Tech Stack:** .NET 8, ASP.NET Core 8, System.Text.Json, xUnit 2, FluentAssertions

**Constants decided here (spec-deferred):**
- Turn time limit: 75 seconds
- Starting health: 30 per player
- Max mana: 10 (starts at 1 on turn 1, +1 per turn)
- Max board size: 7 minions per side
- Max hand size: 10 cards
- Initial hand: 3 cards for player 1, 4 for player 2

---

## File Map

### CCG.GameLogic (class library — zero framework dependencies)

| File | Responsibility |
|---|---|
| `Models/GameState.cs` | Root game state record + helper methods (`GetPlayer`, `WithPlayer`, `GetOpponent`) |
| `Models/PlayerState.cs` | Per-player health, mana, hand, deck |
| `Models/MinionOnBoard.cs` | A minion currently on the battlefield |
| `Models/Card.cs` | Card definition — static attrs, `Definition` JSON, optional `HandlerKey` |
| `Actions/GameAction.cs` | Discriminated union: `PlayCardAction`, `AttackAction`, `EndTurnAction`, `ConcedeAction` |
| `Events/GameEvent.cs` | Discriminated union of all server event types + `StateSnapshotEvent` |
| `Handlers/ICardHandler.cs` | Trigger interface every card handler implements |
| `Handlers/DefaultCardHandler.cs` | Interprets standard effect descriptors from `Definition` JSON |
| `Handlers/CardHandlerRegistry.cs` | Maps handler keys → `ICardHandler`; falls back to `DefaultCardHandler` |
| `Engine/ValidationResult.cs` | `Ok` / `Fail(error)` result type |
| `Engine/ActionValidator.cs` | Validates actions against current game state (pure, stateless) |
| `Engine/GameEngine.cs` | `InitializeGame` + `Process(state, action)` → `(GameState, IReadOnlyList<GameEvent>)` |

### CCG.GameServer (ASP.NET Core web app)

| File | Responsibility |
|---|---|
| `Program.cs` | DI registration, middleware pipeline, WebSocket config |
| `Sessions/PlayerToken.cs` | HMAC-SHA256 token encode/decode: `{ sessionId, playerId, role, exp }` |
| `Sessions/GameSession.cs` | Owns `GameState`; drives action `Channel`, turn timer, reconnection, broadcasting |
| `Sessions/SessionManager.cs` | Creates, indexes, and disposes `GameSession` instances |
| `WebSockets/ConnectionManager.cs` | Tracks active WebSocket + metadata per session |
| `WebSockets/WebSocketHandler.cs` | Accepts WS connections, authenticates tokens, routes messages |
| `Api/InternalController.cs` | `POST /internal/session/create` |
| `Api/Models/CreateSessionRequest.cs` | Request DTO |
| `Api/Models/CreateSessionResponse.cs` | Response DTO: `sessionId`, `p1Token`, `p2Token` |
| `Api/PlatformApiClient.cs` | HTTP client — reports game result to Platform API |

### Test projects

| File | Responsibility |
|---|---|
| `CCG.GameLogic.Tests/TestHelpers/GameStateBuilder.cs` | Fluent fixture builder for test `GameState` |
| `CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs` | All validation rules |
| `CCG.GameLogic.Tests/Engine/GameEngineTests.cs` | State transitions and event generation |
| `CCG.GameLogic.Tests/Handlers/DefaultCardHandlerTests.cs` | Standard effect interpreter |
| `CCG.GameServer.Tests/Sessions/PlayerTokenTests.cs` | Token round-trip and expiry |
| `CCG.GameServer.Tests/Sessions/GameSessionTests.cs` | Session lifecycle, timer, reconnection |

---

## Task 1: Solution scaffold

**Files:**
- Create: `CCG.sln`, `CCG.GameLogic/`, `CCG.GameServer/`, `CCG.GameLogic.Tests/`, `CCG.GameServer.Tests/`

- [ ] **Step 1: Create solution and projects**

```bash
dotnet new sln -n CCG
dotnet new classlib -n CCG.GameLogic -f net8.0 -o CCG.GameLogic
dotnet new webapi -n CCG.GameServer -f net8.0 -o CCG.GameServer
dotnet new xunit -n CCG.GameLogic.Tests -f net8.0 -o CCG.GameLogic.Tests
dotnet new xunit -n CCG.GameServer.Tests -f net8.0 -o CCG.GameServer.Tests
dotnet sln add CCG.GameLogic CCG.GameServer CCG.GameLogic.Tests CCG.GameServer.Tests
```

- [ ] **Step 2: Wire project references and packages**

```bash
dotnet add CCG.GameServer reference CCG.GameLogic
dotnet add CCG.GameLogic.Tests reference CCG.GameLogic
dotnet add CCG.GameServer.Tests reference CCG.GameServer
dotnet add CCG.GameServer.Tests reference CCG.GameLogic
dotnet add CCG.GameLogic.Tests package FluentAssertions
dotnet add CCG.GameServer.Tests package FluentAssertions
```

- [ ] **Step 3: Delete generated boilerplate**

```bash
rm CCG.GameLogic/Class1.cs
rm CCG.GameServer/Controllers/WeatherForecastController.cs
rm CCG.GameServer/WeatherForecast.cs
rm CCG.GameLogic.Tests/UnitTest1.cs
rm CCG.GameServer.Tests/UnitTest1.cs
```

- [ ] **Step 4: Verify build**

```bash
dotnet build CCG.sln
```

Expected: `Build succeeded. 0 Error(s)`

- [ ] **Step 5: Create directory structure**

```bash
mkdir -p CCG.GameLogic/Models CCG.GameLogic/Actions CCG.GameLogic/Events
mkdir -p CCG.GameLogic/Handlers CCG.GameLogic/Engine
mkdir -p CCG.GameServer/Sessions CCG.GameServer/WebSockets CCG.GameServer/Api/Models
mkdir -p CCG.GameLogic.Tests/TestHelpers CCG.GameLogic.Tests/Engine CCG.GameLogic.Tests/Handlers
mkdir -p CCG.GameServer.Tests/Sessions
```

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "chore: scaffold CCG solution with GameLogic and GameServer projects"
```

---

## Task 2: Core game state models

**Files:**
- Create: `CCG.GameLogic/Models/GameState.cs`
- Create: `CCG.GameLogic/Models/PlayerState.cs`
- Create: `CCG.GameLogic/Models/MinionOnBoard.cs`

- [ ] **Step 1: Write `PlayerState.cs`**

```csharp
// CCG.GameLogic/Models/PlayerState.cs
namespace CCG.GameLogic.Models;

public record PlayerState(
    string PlayerId,
    int Health,
    int Mana,
    int MaxMana,
    IReadOnlyList<Card> Hand,
    IReadOnlyList<Card> Deck
)
{
    public int DeckSize => Deck.Count;
}
```

- [ ] **Step 2: Write `MinionOnBoard.cs`**

```csharp
// CCG.GameLogic/Models/MinionOnBoard.cs
namespace CCG.GameLogic.Models;

public record MinionOnBoard(
    string MinionId,
    string CardId,
    string OwnerId,
    int Attack,
    int Health,
    bool HasTaunt,
    bool HasDivineShield,
    bool CanAttack
);
```

- [ ] **Step 3: Write `GameState.cs`**

```csharp
// CCG.GameLogic/Models/GameState.cs
namespace CCG.GameLogic.Models;

public enum GamePhase { WaitingForPlayers, InProgress, Ended }

public record TurnState(string ActivePlayerId, int Number);
public record TimerState(int SecondsRemaining);

public record GameState(
    string SessionId,
    PlayerState Player1,
    PlayerState Player2,
    IReadOnlyList<MinionOnBoard> Board,
    TurnState Turn,
    TimerState Timer,
    GamePhase Phase,
    string? WinnerId
)
{
    public PlayerState GetPlayer(string playerId) =>
        Player1.PlayerId == playerId ? Player1 : Player2;

    public PlayerState GetOpponent(string playerId) =>
        Player1.PlayerId == playerId ? Player2 : Player1;

    public GameState WithPlayer(PlayerState updated) =>
        Player1.PlayerId == updated.PlayerId
            ? this with { Player1 = updated }
            : this with { Player2 = updated };
}
```

- [ ] **Step 4: Verify build**

```bash
dotnet build CCG.GameLogic
```

Expected: `Build succeeded. 0 Error(s)`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Models/
git commit -m "feat(gamelogic): add core game state models"
```

---

## Task 3: Card model

**Files:**
- Create: `CCG.GameLogic/Models/Card.cs`

- [ ] **Step 1: Write `Card.cs`**

```csharp
// CCG.GameLogic/Models/Card.cs
namespace CCG.GameLogic.Models;

using System.Text.Json;

public enum CardType { Minion, Spell }
public enum CardRarity { Common, Rare, Epic, Legendary }

public record Card(
    string Id,
    string Name,
    int ManaCost,
    CardType Type,
    CardRarity Rarity,
    JsonElement Definition,
    string? HandlerKey = null
)
{
    public bool RequiresTarget =>
        Definition.TryGetProperty("targeted", out var t) && t.GetBoolean();
}
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameLogic
```

Expected: `Build succeeded. 0 Error(s)`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameLogic/Models/Card.cs
git commit -m "feat(gamelogic): add Card model with handler key and targeting flag"
```

---

## Task 4: Action types

**Files:**
- Create: `CCG.GameLogic/Actions/GameAction.cs`

- [ ] **Step 1: Write `GameAction.cs`**

```csharp
// CCG.GameLogic/Actions/GameAction.cs
namespace CCG.GameLogic.Actions;

public abstract record GameAction(string PlayerId);
public record PlayCardAction(string PlayerId, string CardId, string? TargetId) : GameAction(PlayerId);
public record AttackAction(string PlayerId, string AttackerId, string TargetId) : GameAction(PlayerId);
public record EndTurnAction(string PlayerId) : GameAction(PlayerId);
public record ConcedeAction(string PlayerId) : GameAction(PlayerId);
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameLogic
```

- [ ] **Step 3: Commit**

```bash
git add CCG.GameLogic/Actions/
git commit -m "feat(gamelogic): add action discriminated union"
```

---

## Task 5: Event types

**Files:**
- Create: `CCG.GameLogic/Events/GameEvent.cs`

- [ ] **Step 1: Write `GameEvent.cs`**

```csharp
// CCG.GameLogic/Events/GameEvent.cs
namespace CCG.GameLogic.Events;

using CCG.GameLogic.Models;

public abstract record GameEvent;

// Sent to each player on connect and on reconnect — scoped per recipient
public record StateSnapshotEvent(
    int MyHealth,
    int MyMana,
    int MyMaxMana,
    IReadOnlyList<Card> MyHand,
    int MyDeckSize,
    int OpponentHealth,
    int OpponentMana,
    int OpponentMaxMana,
    int OpponentHandCount,
    int OpponentDeckSize,
    IReadOnlyList<MinionOnBoard> Board,
    TurnState Turn,
    TimerState Timer
) : GameEvent;

// Card drawn: owner receives DrawnCard; opponent receives DrawnCard = null
public record CardDrawnEvent(
    string PlayerId,
    Card? DrawnCard,
    int NewHandCount,
    int NewDeckSize
) : GameEvent;

public record CardPlayedEvent(string PlayerId, string CardId, string? TargetId) : GameEvent;

public record MinionSummonedEvent(
    string MinionId,
    string CardId,
    string OwnerId,
    int Attack,
    int Health,
    bool HasTaunt
) : GameEvent;

public record DamageDealtEvent(string SourceId, string TargetId, int Amount) : GameEvent;
public record MinionDiedEvent(string MinionId, string OwnerId) : GameEvent;
public record TurnStartedEvent(string PlayerId, int TimeLimit) : GameEvent;
public record TurnEndedEvent(string PlayerId) : GameEvent;
public record GameEndedEvent(string WinnerId, string Reason) : GameEvent;
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameLogic
```

- [ ] **Step 3: Commit**

```bash
git add CCG.GameLogic/Events/
git commit -m "feat(gamelogic): add event discriminated union"
```

---

## Task 6: Card handler system

**Files:**
- Create: `CCG.GameLogic/Handlers/ICardHandler.cs`
- Create: `CCG.GameLogic/Handlers/DefaultCardHandler.cs`
- Create: `CCG.GameLogic/Handlers/CardHandlerRegistry.cs`

- [ ] **Step 1: Write `ICardHandler.cs`**

```csharp
// CCG.GameLogic/Handlers/ICardHandler.cs
namespace CCG.GameLogic.Handlers;

using CCG.GameLogic.Events;
using CCG.GameLogic.Models;

public interface ICardHandler
{
    (GameState State, IReadOnlyList<GameEvent> Events) OnPlay(
        GameState state, Card card, string? targetId);

    (GameState State, IReadOnlyList<GameEvent> Events) OnDeath(
        GameState state, MinionOnBoard minion);
}
```

- [ ] **Step 2: Write `DefaultCardHandler.cs` (skeleton — effects implemented in Task 14)**

```csharp
// CCG.GameLogic/Handlers/DefaultCardHandler.cs
namespace CCG.GameLogic.Handlers;

using CCG.GameLogic.Events;
using CCG.GameLogic.Models;

public class DefaultCardHandler : ICardHandler
{
    public (GameState State, IReadOnlyList<GameEvent> Events) OnPlay(
        GameState state, Card card, string? targetId)
    {
        if (card.Type == CardType.Minion)
            return PlayMinion(state, card);

        return PlaySpell(state, card, targetId);
    }

    public (GameState State, IReadOnlyList<GameEvent> Events) OnDeath(
        GameState state, MinionOnBoard minion) =>
        (state, Array.Empty<GameEvent>());

    private static (GameState, IReadOnlyList<GameEvent>) PlayMinion(GameState state, Card card)
    {
        var attack = card.Definition.TryGetProperty("attack", out var a) ? a.GetInt32() : 0;
        var health = card.Definition.TryGetProperty("health", out var h) ? h.GetInt32() : 1;
        var hasTaunt = card.Definition.TryGetProperty("taunt", out var t) && t.GetBoolean();
        var hasDivineShield = card.Definition.TryGetProperty("divine_shield", out var ds) && ds.GetBoolean();
        var hasCharge = card.Definition.TryGetProperty("charge", out var c) && c.GetBoolean();
        var hasRush = card.Definition.TryGetProperty("rush", out var r) && r.GetBoolean();

        var minionId = Guid.NewGuid().ToString();
        var minion = new MinionOnBoard(
            MinionId: minionId,
            CardId: card.Id,
            OwnerId: state.Turn.ActivePlayerId,
            Attack: attack,
            Health: health,
            HasTaunt: hasTaunt,
            HasDivineShield: hasDivineShield,
            CanAttack: hasCharge || hasRush
        );

        var newBoard = state.Board.Append(minion).ToList();
        var newState = state with { Board = newBoard };
        var events = new GameEvent[]
        {
            new MinionSummonedEvent(minionId, card.Id, state.Turn.ActivePlayerId, attack, health, hasTaunt)
        };
        return (newState, events);
    }

    // Spell effect processing implemented in Task 14
    private static (GameState, IReadOnlyList<GameEvent>) PlaySpell(
        GameState state, Card card, string? targetId) =>
        (state, Array.Empty<GameEvent>());
}
```

- [ ] **Step 3: Write `CardHandlerRegistry.cs`**

```csharp
// CCG.GameLogic/Handlers/CardHandlerRegistry.cs
namespace CCG.GameLogic.Handlers;

public class CardHandlerRegistry
{
    private readonly Dictionary<string, ICardHandler> _handlers = new();
    private readonly ICardHandler _default = new DefaultCardHandler();

    public void Register(string key, ICardHandler handler) =>
        _handlers[key] = handler;

    public ICardHandler GetHandler(string? handlerKey) =>
        handlerKey != null && _handlers.TryGetValue(handlerKey, out var h) ? h : _default;
}
```

- [ ] **Step 4: Write failing test for registry**

```csharp
// CCG.GameLogic.Tests/Handlers/DefaultCardHandlerTests.cs
namespace CCG.GameLogic.Tests.Handlers;

using CCG.GameLogic.Handlers;
using FluentAssertions;

public class CardHandlerRegistryTests
{
    [Fact]
    public void GetHandler_NullKey_ReturnsDefaultCardHandler()
    {
        var registry = new CardHandlerRegistry();
        var handler = registry.GetHandler(null);
        handler.Should().BeOfType<DefaultCardHandler>();
    }

    [Fact]
    public void GetHandler_UnknownKey_ReturnsDefaultCardHandler()
    {
        var registry = new CardHandlerRegistry();
        var handler = registry.GetHandler("UnknownCard");
        handler.Should().BeOfType<DefaultCardHandler>();
    }

    [Fact]
    public void GetHandler_RegisteredKey_ReturnsRegisteredHandler()
    {
        var registry = new CardHandlerRegistry();
        var custom = new FakeCardHandler();
        registry.Register("MyCard", custom);

        var handler = registry.GetHandler("MyCard");

        handler.Should().BeSameAs(custom);
    }
}

file class FakeCardHandler : ICardHandler
{
    public (CCG.GameLogic.Models.GameState, IReadOnlyList<CCG.GameLogic.Events.GameEvent>) OnPlay(
        CCG.GameLogic.Models.GameState s, CCG.GameLogic.Models.Card c, string? t) => (s, Array.Empty<CCG.GameLogic.Events.GameEvent>());
    public (CCG.GameLogic.Models.GameState, IReadOnlyList<CCG.GameLogic.Events.GameEvent>) OnDeath(
        CCG.GameLogic.Models.GameState s, CCG.GameLogic.Models.MinionOnBoard m) => (s, Array.Empty<CCG.GameLogic.Events.GameEvent>());
}
```

- [ ] **Step 5: Run test — expect pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~CardHandlerRegistryTests"
```

Expected: `Passed: 3`

- [ ] **Step 6: Commit**

```bash
git add CCG.GameLogic/Handlers/ CCG.GameLogic.Tests/Handlers/
git commit -m "feat(gamelogic): add card handler system with registry and DefaultCardHandler skeleton"
```

---

## Task 7: GameStateBuilder + ActionValidator skeleton

**Files:**
- Create: `CCG.GameLogic.Tests/TestHelpers/GameStateBuilder.cs`
- Create: `CCG.GameLogic/Engine/ValidationResult.cs`
- Create: `CCG.GameLogic/Engine/ActionValidator.cs` (skeleton)

- [ ] **Step 1: Write `ValidationResult.cs`**

```csharp
// CCG.GameLogic/Engine/ValidationResult.cs
namespace CCG.GameLogic.Engine;

public record ValidationResult(bool IsValid, string? Error = null)
{
    public static ValidationResult Ok() => new(true);
    public static ValidationResult Fail(string error) => new(false, error);
}
```

- [ ] **Step 2: Write `ActionValidator.cs` skeleton**

```csharp
// CCG.GameLogic/Engine/ActionValidator.cs
namespace CCG.GameLogic.Engine;

using CCG.GameLogic.Actions;
using CCG.GameLogic.Models;

public class ActionValidator
{
    public ValidationResult Validate(GameState state, GameAction action) => action switch
    {
        PlayCardAction a => ValidatePlayCard(state, a),
        AttackAction a   => ValidateAttack(state, a),
        EndTurnAction a  => ValidateEndTurn(state, a),
        ConcedeAction    => ValidateConcede(state),
        _                => ValidationResult.Fail("Unknown action type")
    };

    private static ValidationResult ValidatePlayCard(GameState state, PlayCardAction action) =>
        ValidationResult.Ok(); // implemented in Task 8

    private static ValidationResult ValidateAttack(GameState state, AttackAction action) =>
        ValidationResult.Ok(); // implemented in Task 9

    private static ValidationResult ValidateEndTurn(GameState state, EndTurnAction action) =>
        ValidationResult.Ok(); // implemented in Task 9

    private static ValidationResult ValidateConcede(GameState state) =>
        ValidationResult.Ok(); // implemented in Task 9
}
```

- [ ] **Step 3: Write `GameStateBuilder.cs`**

```csharp
// CCG.GameLogic.Tests/TestHelpers/GameStateBuilder.cs
namespace CCG.GameLogic.Tests.TestHelpers;

using System.Text.Json;
using CCG.GameLogic.Models;

public static class GameStateBuilder
{
    public static GameState InProgress(
        string player1Id = "p1",
        string player2Id = "p2",
        string? activePlayerId = null,
        int turnNumber = 1,
        int p1Health = 30,
        int p2Health = 30,
        int p1Mana = 3,
        int p2Mana = 3,
        IReadOnlyList<Card>? p1Hand = null,
        IReadOnlyList<Card>? p2Hand = null,
        IReadOnlyList<MinionOnBoard>? board = null)
    {
        activePlayerId ??= player1Id;
        return new GameState(
            SessionId: "test-session",
            Player1: Player(player1Id, p1Health, p1Mana, p1Hand),
            Player2: Player(player2Id, p2Health, p2Mana, p2Hand),
            Board: board ?? Array.Empty<MinionOnBoard>(),
            Turn: new TurnState(activePlayerId, turnNumber),
            Timer: new TimerState(75),
            Phase: GamePhase.InProgress,
            WinnerId: null
        );
    }

    public static PlayerState Player(
        string id, int health = 30, int mana = 3,
        IReadOnlyList<Card>? hand = null) =>
        new PlayerState(
            PlayerId: id,
            Health: health,
            Mana: mana,
            MaxMana: mana,
            Hand: hand ?? Array.Empty<Card>(),
            Deck: Array.Empty<Card>()
        );

    public static Card MinionCard(
        string id = "c1", int manaCost = 3, int attack = 3, int health = 2,
        bool taunt = false, bool charge = false) =>
        new Card(
            Id: id,
            Name: "Test Minion",
            ManaCost: manaCost,
            Type: CardType.Minion,
            Rarity: CardRarity.Common,
            Definition: JsonDocument.Parse(
                $$$"""{"attack":{{{attack}}},"health":{{{health}}},"taunt":{{{taunt.ToString().ToLower()}}},"charge":{{{charge.ToString().ToLower()}}}}"""
            ).RootElement.Clone()
        );

    public static Card SpellCard(
        string id = "s1", int manaCost = 4, bool targeted = false) =>
        new Card(
            Id: id,
            Name: "Test Spell",
            ManaCost: manaCost,
            Type: CardType.Spell,
            Rarity: CardRarity.Common,
            Definition: JsonDocument.Parse(
                $$$"""{"targeted":{{{targeted.ToString().ToLower()}}},"effects":[{"type":"deal_damage","amount":6}]}"""
            ).RootElement.Clone()
        );

    public static MinionOnBoard Minion(
        string minionId = "m1", string ownerId = "p1",
        int attack = 3, int health = 2,
        bool hasTaunt = false, bool canAttack = true) =>
        new MinionOnBoard(minionId, "c1", ownerId, attack, health, hasTaunt, false, canAttack);
}
```

- [ ] **Step 4: Verify build**

```bash
dotnet build CCG.sln
```

Expected: `Build succeeded. 0 Error(s)`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/ValidationResult.cs CCG.GameLogic/Engine/ActionValidator.cs
git add CCG.GameLogic.Tests/TestHelpers/
git commit -m "feat(gamelogic): add ValidationResult, ActionValidator skeleton, and GameStateBuilder"
```

---

## Task 8: ActionValidator — PlayCard rules

**Files:**
- Modify: `CCG.GameLogic/Engine/ActionValidator.cs`
- Create: `CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs` (PlayCard section)

- [ ] **Step 1: Write failing tests for PlayCard validation**

```csharp
// CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs
namespace CCG.GameLogic.Tests.Engine;

using CCG.GameLogic.Actions;
using CCG.GameLogic.Engine;
using CCG.GameLogic.Models;
using CCG.GameLogic.Tests.TestHelpers;
using FluentAssertions;

public class ActionValidatorTests
{
    private readonly ActionValidator _sut = new();

    [Fact]
    public void PlayCard_WhenNotActivePlayer_Fails()
    {
        var card = GameStateBuilder.MinionCard("c1");
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            p2Hand: new[] { card });

        var result = _sut.Validate(state, new PlayCardAction("p2", "c1", null));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Not the active player");
    }

    [Fact]
    public void PlayCard_CardNotInHand_Fails()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        var result = _sut.Validate(state, new PlayCardAction("p1", "c1", null));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Card not in hand");
    }

    [Fact]
    public void PlayCard_InsufficientMana_Fails()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 5);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 2,
            p1Hand: new[] { card });

        var result = _sut.Validate(state, new PlayCardAction("p1", "c1", null));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Not enough mana");
    }

    [Fact]
    public void PlayCard_TargetedSpellWithoutTarget_Fails()
    {
        var spell = GameStateBuilder.SpellCard("s1", targeted: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            p1Hand: new[] { spell });

        var result = _sut.Validate(state, new PlayCardAction("p1", "s1", null));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Target required");
    }

    [Fact]
    public void PlayCard_BoardFull_Fails()
    {
        var card = GameStateBuilder.MinionCard("c99");
        var fullBoard = Enumerable.Range(1, 7)
            .Select(i => GameStateBuilder.Minion($"m{i}", "p1"))
            .ToList();
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            p1Hand: new[] { card }, board: fullBoard);

        var result = _sut.Validate(state, new PlayCardAction("p1", "c99", null));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Board is full");
    }

    [Fact]
    public void PlayCard_ValidMinion_Succeeds()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var result = _sut.Validate(state, new PlayCardAction("p1", "c1", null));

        result.IsValid.Should().BeTrue();
    }
}
```

- [ ] **Step 2: Run — expect failures**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~ActionValidatorTests"
```

Expected: `Failed: 5` (all PlayCard tests fail — skeleton returns Ok always)

- [ ] **Step 3: Implement PlayCard validation**

Replace `ValidatePlayCard` in `ActionValidator.cs`:

```csharp
private static ValidationResult ValidatePlayCard(GameState state, PlayCardAction action)
{
    if (state.Turn.ActivePlayerId != action.PlayerId)
        return ValidationResult.Fail("Not the active player");

    var player = state.GetPlayer(action.PlayerId);
    var card = player.Hand.FirstOrDefault(c => c.Id == action.CardId);
    if (card is null)
        return ValidationResult.Fail("Card not in hand");

    if (player.Mana < card.ManaCost)
        return ValidationResult.Fail("Not enough mana");

    if (card.RequiresTarget && action.TargetId is null)
        return ValidationResult.Fail("Target required");

    if (card.Type == CardType.Minion)
    {
        var ownBoardMinions = state.Board.Count(m => m.OwnerId == action.PlayerId);
        if (ownBoardMinions >= 7)
            return ValidationResult.Fail("Board is full");
    }

    return ValidationResult.Ok();
}
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~ActionValidatorTests"
```

Expected: `Passed: 6`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/ActionValidator.cs CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs
git commit -m "feat(gamelogic): implement PlayCard validation rules"
```

---

## Task 9: ActionValidator — Attack, EndTurn, Concede

**Files:**
- Modify: `CCG.GameLogic/Engine/ActionValidator.cs`
- Modify: `CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs`

- [ ] **Step 1: Add failing tests**

Append to `ActionValidatorTests.cs`:

```csharp
    // --- Attack ---

    [Fact]
    public void Attack_AttackerNotOwned_Fails()
    {
        var minion = GameStateBuilder.Minion("m1", ownerId: "p2");
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", board: new[] { minion });

        var result = _sut.Validate(state, new AttackAction("p1", "m1", "p2"));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Attacker not owned by active player");
    }

    [Fact]
    public void Attack_AttackerExhausted_Fails()
    {
        var minion = GameStateBuilder.Minion("m1", ownerId: "p1", canAttack: false);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", board: new[] { minion });

        var result = _sut.Validate(state, new AttackAction("p1", "m1", "p2"));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Attacker cannot attack this turn");
    }

    [Fact]
    public void Attack_MustTargetTaunt_Fails()
    {
        var attacker = GameStateBuilder.Minion("m1", ownerId: "p1", canAttack: true);
        var tauntMinion = GameStateBuilder.Minion("m2", ownerId: "p2", hasTaunt: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { attacker, tauntMinion });

        // Targeting the opponent hero while a taunt minion exists
        var result = _sut.Validate(state, new AttackAction("p1", "m1", "p2"));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Must attack a minion with taunt");
    }

    [Fact]
    public void Attack_ValidMinionTarget_Succeeds()
    {
        var attacker = GameStateBuilder.Minion("m1", ownerId: "p1", canAttack: true);
        var target = GameStateBuilder.Minion("m2", ownerId: "p2");
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { attacker, target });

        var result = _sut.Validate(state, new AttackAction("p1", "m1", "m2"));

        result.IsValid.Should().BeTrue();
    }

    [Fact]
    public void Attack_ValidHeroTarget_Succeeds()
    {
        var attacker = GameStateBuilder.Minion("m1", ownerId: "p1", canAttack: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", player2Id: "p2",
            board: new[] { attacker });

        var result = _sut.Validate(state, new AttackAction("p1", "m1", "p2"));

        result.IsValid.Should().BeTrue();
    }

    // --- EndTurn ---

    [Fact]
    public void EndTurn_WhenNotActivePlayer_Fails()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        var result = _sut.Validate(state, new EndTurnAction("p2"));

        result.IsValid.Should().BeFalse();
        result.Error.Should().Be("Not the active player");
    }

    [Fact]
    public void EndTurn_WhenActivePlayer_Succeeds()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        var result = _sut.Validate(state, new EndTurnAction("p1"));

        result.IsValid.Should().BeTrue();
    }

    // --- Concede ---

    [Fact]
    public void Concede_AlwaysValid_ForEitherPlayer()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        _sut.Validate(state, new ConcedeAction("p1")).IsValid.Should().BeTrue();
        _sut.Validate(state, new ConcedeAction("p2")).IsValid.Should().BeTrue();
    }
```

- [ ] **Step 2: Run — expect new tests to fail**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~ActionValidatorTests"
```

Expected: multiple failures in the new Attack/EndTurn tests.

- [ ] **Step 3: Implement remaining validation methods**

Replace the three skeleton methods in `ActionValidator.cs`:

```csharp
private static ValidationResult ValidateAttack(GameState state, AttackAction action)
{
    if (state.Turn.ActivePlayerId != action.PlayerId)
        return ValidationResult.Fail("Not the active player");

    var attacker = state.Board.FirstOrDefault(m =>
        m.MinionId == action.AttackerId && m.OwnerId == action.PlayerId);
    if (attacker is null)
        return ValidationResult.Fail("Attacker not owned by active player");

    if (!attacker.CanAttack)
        return ValidationResult.Fail("Attacker cannot attack this turn");

    var opponentTauntMinions = state.Board
        .Where(m => m.OwnerId != action.PlayerId && m.HasTaunt)
        .ToList();

    if (opponentTauntMinions.Count > 0)
    {
        // Target must be one of the taunt minions
        var targetIsTaunt = opponentTauntMinions.Any(m => m.MinionId == action.TargetId);
        if (!targetIsTaunt)
            return ValidationResult.Fail("Must attack a minion with taunt");
    }

    // Target must be opponent minion or opponent hero (identified by player id)
    var opponent = state.GetOpponent(action.PlayerId);
    var targetIsOpponentHero = action.TargetId == opponent.PlayerId;
    var targetIsOpponentMinion = state.Board.Any(m =>
        m.MinionId == action.TargetId && m.OwnerId == opponent.PlayerId);

    if (!targetIsOpponentHero && !targetIsOpponentMinion)
        return ValidationResult.Fail("Invalid attack target");

    return ValidationResult.Ok();
}

private static ValidationResult ValidateEndTurn(GameState state, EndTurnAction action)
{
    if (state.Turn.ActivePlayerId != action.PlayerId)
        return ValidationResult.Fail("Not the active player");
    return ValidationResult.Ok();
}

private static ValidationResult ValidateConcede(GameState state) =>
    state.Phase == GamePhase.InProgress
        ? ValidationResult.Ok()
        : ValidationResult.Fail("Game is not in progress");
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~ActionValidatorTests"
```

Expected: `Passed: 14`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/ActionValidator.cs CCG.GameLogic.Tests/Engine/ActionValidatorTests.cs
git commit -m "feat(gamelogic): implement Attack, EndTurn, Concede validation"
```

---

## Task 10: GameEngine — game initialization

**Files:**
- Create: `CCG.GameLogic/Engine/GameEngine.cs`
- Create: `CCG.GameLogic.Tests/Engine/GameEngineTests.cs`

- [ ] **Step 1: Write failing tests for `InitializeGame`**

```csharp
// CCG.GameLogic.Tests/Engine/GameEngineTests.cs
namespace CCG.GameLogic.Tests.Engine;

using CCG.GameLogic.Engine;
using CCG.GameLogic.Handlers;
using CCG.GameLogic.Models;
using CCG.GameLogic.Tests.TestHelpers;
using FluentAssertions;

public class GameEngineTests
{
    private readonly GameEngine _sut = new(new CardHandlerRegistry());

    // --- InitializeGame ---

    [Fact]
    public void InitializeGame_SetsPhaseToWaitingForPlayers()
    {
        var deck = Deck(30);
        var (state, _) = GameEngine.InitializeGame("s1", "p1", "p2", deck, deck);
        state.Phase.Should().Be(GamePhase.WaitingForPlayers);
    }

    [Fact]
    public void InitializeGame_BothPlayersStartAt30Health()
    {
        var deck = Deck(30);
        var (state, _) = GameEngine.InitializeGame("s1", "p1", "p2", deck, deck);
        state.Player1.Health.Should().Be(30);
        state.Player2.Health.Should().Be(30);
    }

    [Fact]
    public void InitializeGame_Player1Gets3CardsPlayer2Gets4()
    {
        var deck = Deck(30);
        var (state, _) = GameEngine.InitializeGame("s1", "p1", "p2", deck, deck);
        state.Player1.Hand.Count.Should().Be(3);
        state.Player2.Hand.Count.Should().Be(4);
    }

    [Fact]
    public void InitializeGame_DecksReducedByHandSize()
    {
        var deck = Deck(30);
        var (state, _) = GameEngine.InitializeGame("s1", "p1", "p2", deck, deck);
        state.Player1.DeckSize.Should().Be(27);
        state.Player2.DeckSize.Should().Be(26);
    }

    [Fact]
    public void InitializeGame_Player1IsActiveOnTurn1()
    {
        var deck = Deck(30);
        var (state, _) = GameEngine.InitializeGame("s1", "p1", "p2", deck, deck);
        state.Turn.ActivePlayerId.Should().Be("p1");
        state.Turn.Number.Should().Be(1);
    }

    private static IReadOnlyList<Card> Deck(int size) =>
        Enumerable.Range(1, size)
            .Select(i => GameStateBuilder.MinionCard($"card{i}"))
            .ToList();
}
```

- [ ] **Step 2: Run — expect failures (GameEngine doesn't exist)**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: compile error or `Failed: 5`

- [ ] **Step 3: Write `GameEngine.cs` with `InitializeGame`**

```csharp
// CCG.GameLogic/Engine/GameEngine.cs
namespace CCG.GameLogic.Engine;

using CCG.GameLogic.Actions;
using CCG.GameLogic.Events;
using CCG.GameLogic.Handlers;
using CCG.GameLogic.Models;

public class GameEngine(CardHandlerRegistry registry)
{
    private readonly ActionValidator _validator = new();

    public static (GameState State, IReadOnlyList<GameEvent> Events) InitializeGame(
        string sessionId,
        string player1Id,
        string player2Id,
        IReadOnlyList<Card> deck1,
        IReadOnlyList<Card> deck2)
    {
        var shuffled1 = Shuffle(deck1);
        var shuffled2 = Shuffle(deck2);

        var p1Hand = shuffled1.Take(3).ToList();
        var p1Deck = shuffled1.Skip(3).ToList();
        var p2Hand = shuffled2.Take(4).ToList();
        var p2Deck = shuffled2.Skip(4).ToList();

        var state = new GameState(
            SessionId: sessionId,
            Player1: new PlayerState(player1Id, 30, 0, 0, p1Hand, p1Deck),
            Player2: new PlayerState(player2Id, 30, 0, 0, p2Hand, p2Deck),
            Board: Array.Empty<MinionOnBoard>(),
            Turn: new TurnState(player1Id, 1),
            Timer: new TimerState(75),
            Phase: GamePhase.WaitingForPlayers,
            WinnerId: null
        );

        return (state, Array.Empty<GameEvent>());
    }

    public (GameState State, IReadOnlyList<GameEvent> Events) Process(
        GameState state, GameAction action)
    {
        var validation = _validator.Validate(state, action);
        if (!validation.IsValid)
            throw new InvalidOperationException(validation.Error);

        return action switch
        {
            PlayCardAction a => ProcessPlayCard(state, a),
            AttackAction a   => ProcessAttack(state, a),
            EndTurnAction a  => ProcessEndTurn(state, a),
            ConcedeAction a  => ProcessConcede(state, a),
            _                => throw new InvalidOperationException("Unknown action")
        };
    }

    private static IReadOnlyList<T> Shuffle<T>(IReadOnlyList<T> source)
    {
        var list = source.ToList();
        var rng = Random.Shared;
        for (var i = list.Count - 1; i > 0; i--)
        {
            var j = rng.Next(i + 1);
            (list[i], list[j]) = (list[j], list[i]);
        }
        return list;
    }

    // Implemented in Tasks 11–13
    private (GameState, IReadOnlyList<GameEvent>) ProcessPlayCard(GameState state, PlayCardAction action) =>
        (state, Array.Empty<GameEvent>());

    private (GameState, IReadOnlyList<GameEvent>) ProcessAttack(GameState state, AttackAction action) =>
        (state, Array.Empty<GameEvent>());

    private (GameState, IReadOnlyList<GameEvent>) ProcessEndTurn(GameState state, EndTurnAction action) =>
        (state, Array.Empty<GameEvent>());

    private (GameState, IReadOnlyList<GameEvent>) ProcessConcede(GameState state, ConcedeAction action) =>
        (state, Array.Empty<GameEvent>());
}
```

- [ ] **Step 4: Run — expect pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: `Passed: 5`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/GameEngine.cs CCG.GameLogic.Tests/Engine/GameEngineTests.cs
git commit -m "feat(gamelogic): add GameEngine with InitializeGame"
```

---

## Task 11: GameEngine — ProcessPlayCard

**Files:**
- Modify: `CCG.GameLogic/Engine/GameEngine.cs`
- Modify: `CCG.GameLogic.Tests/Engine/GameEngineTests.cs`

- [ ] **Step 1: Write failing tests**

Append to `GameEngineTests.cs`:

```csharp
    // --- PlayCard ---

    [Fact]
    public void PlayCard_MinionCard_AddsMinionToBoard()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2, attack: 3, health: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.PlayCardAction("p1", "c1", null));

        newState.Board.Should().HaveCount(1);
        newState.Board[0].OwnerId.Should().Be("p1");
        newState.Board[0].Attack.Should().Be(3);
        newState.Board[0].Health.Should().Be(2);
    }

    [Fact]
    public void PlayCard_DeductsManaAndRemovesCardFromHand()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.PlayCardAction("p1", "c1", null));

        newState.GetPlayer("p1").Mana.Should().Be(1);
        newState.GetPlayer("p1").Hand.Should().BeEmpty();
    }

    [Fact]
    public void PlayCard_EmitsCardPlayedAndMinionSummonedEvents()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var (_, events) = _sut.Process(state, new CCG.GameLogic.Actions.PlayCardAction("p1", "c1", null));

        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.CardPlayedEvent);
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.MinionSummonedEvent);
    }

    [Fact]
    public void PlayCard_ChargeMinion_CanAttackImmediately()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2, charge: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.PlayCardAction("p1", "c1", null));

        newState.Board[0].CanAttack.Should().BeTrue();
    }

    [Fact]
    public void PlayCard_NormalMinion_CannotAttackImmediately()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.PlayCardAction("p1", "c1", null));

        newState.Board[0].CanAttack.Should().BeFalse();
    }
```

- [ ] **Step 2: Run — expect failures**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests.PlayCard"
```

Expected: `Failed: 5`

- [ ] **Step 3: Implement `ProcessPlayCard`**

Replace the skeleton in `GameEngine.cs`:

```csharp
private (GameState, IReadOnlyList<GameEvent>) ProcessPlayCard(
    GameState state, PlayCardAction action)
{
    var player = state.GetPlayer(action.PlayerId);
    var card = player.Hand.First(c => c.Id == action.CardId);

    // Deduct mana and remove from hand
    var updatedPlayer = player with
    {
        Mana = player.Mana - card.ManaCost,
        Hand = player.Hand.Where(c => c.Id != action.CardId).ToList()
    };
    var stateAfterPlay = state.WithPlayer(updatedPlayer);

    // Invoke handler
    var handler = registry.GetHandler(card.HandlerKey);
    var (stateAfterHandler, handlerEvents) = handler.OnPlay(stateAfterPlay, card, action.TargetId);

    var allEvents = new List<GameEvent>
    {
        new CardPlayedEvent(action.PlayerId, action.CardId, action.TargetId)
    };
    allEvents.AddRange(handlerEvents);

    return (stateAfterHandler, allEvents);
}
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: `Passed: 10`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/GameEngine.cs CCG.GameLogic.Tests/Engine/GameEngineTests.cs
git commit -m "feat(gamelogic): implement PlayCard action processing"
```

---

## Task 12: GameEngine — ProcessAttack

**Files:**
- Modify: `CCG.GameLogic/Engine/GameEngine.cs`
- Modify: `CCG.GameLogic.Tests/Engine/GameEngineTests.cs`

- [ ] **Step 1: Write failing tests**

Append to `GameEngineTests.cs`:

```csharp
    // --- Attack ---

    [Fact]
    public void Attack_MinionVsHero_DealsCorrectDamage()
    {
        var attacker = GameStateBuilder.Minion("m1", "p1", attack: 4, canAttack: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p2Health: 30,
            board: new[] { attacker });

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.AttackAction("p1", "m1", "p2"));

        newState.GetPlayer("p2").Health.Should().Be(26);
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.DamageDealtEvent);
    }

    [Fact]
    public void Attack_MinionVsMinion_BothTakeDamage()
    {
        var attacker = GameStateBuilder.Minion("m1", "p1", attack: 3, health: 2, canAttack: true);
        var defender = GameStateBuilder.Minion("m2", "p2", attack: 2, health: 4);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { attacker, defender });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.AttackAction("p1", "m1", "m2"));

        newState.Board.First(m => m.MinionId == "m1").Health.Should().Be(0);
        newState.Board.First(m => m.MinionId == "m2").Health.Should().Be(1);
    }

    [Fact]
    public void Attack_DeadMinionsRemovedFromBoard()
    {
        var attacker = GameStateBuilder.Minion("m1", "p1", attack: 4, health: 2, canAttack: true);
        var defender = GameStateBuilder.Minion("m2", "p2", attack: 4, health: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { attacker, defender });

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.AttackAction("p1", "m1", "m2"));

        newState.Board.Should().BeEmpty();
        events.OfType<CCG.GameLogic.Events.MinionDiedEvent>().Should().HaveCount(2);
    }

    [Fact]
    public void Attack_AttackerExhaustedAfterAttacking()
    {
        var attacker = GameStateBuilder.Minion("m1", "p1", attack: 2, health: 5, canAttack: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { attacker });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.AttackAction("p1", "m1", "p2"));

        newState.Board.First(m => m.MinionId == "m1").CanAttack.Should().BeFalse();
    }

    [Fact]
    public void Attack_ReducesOpponentHealthToZero_EmitsGameEnded()
    {
        var attacker = GameStateBuilder.Minion("m1", "p1", attack: 30, canAttack: true);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p2Health: 10,
            board: new[] { attacker });

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.AttackAction("p1", "m1", "p2"));

        newState.Phase.Should().Be(GamePhase.Ended);
        newState.WinnerId.Should().Be("p1");
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.GameEndedEvent);
    }
```

- [ ] **Step 2: Run — expect failures**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests.Attack"
```

Expected: `Failed: 5`

- [ ] **Step 3: Implement `ProcessAttack`**

Replace the skeleton in `GameEngine.cs`:

```csharp
private (GameState, IReadOnlyList<GameEvent>) ProcessAttack(
    GameState state, AttackAction action)
{
    var events = new List<GameEvent>();
    var attacker = state.Board.First(m => m.MinionId == action.AttackerId);
    var opponent = state.GetOpponent(action.PlayerId);

    // Exhaust attacker
    var board = state.Board.Select(m =>
        m.MinionId == action.AttackerId ? m with { CanAttack = false } : m
    ).ToList();
    state = state with { Board = board };

    // Is target a hero or minion?
    if (action.TargetId == opponent.PlayerId)
    {
        // Attack hero
        var newHealth = opponent.Health - attacker.Attack;
        events.Add(new DamageDealtEvent(action.AttackerId, action.TargetId, attacker.Attack));
        var updatedOpponent = opponent with { Health = newHealth };
        state = state.WithPlayer(updatedOpponent);

        if (newHealth <= 0)
        {
            state = state with { Phase = GamePhase.Ended, WinnerId = action.PlayerId };
            events.Add(new GameEndedEvent(action.PlayerId, "Health depleted"));
        }
    }
    else
    {
        // Attack minion
        var defender = state.Board.First(m => m.MinionId == action.TargetId);

        var updatedAttacker = attacker with { Health = attacker.Health - defender.Attack };
        var updatedDefender = defender with { Health = defender.Health - attacker.Attack };

        events.Add(new DamageDealtEvent(action.AttackerId, action.TargetId, attacker.Attack));
        events.Add(new DamageDealtEvent(action.TargetId, action.AttackerId, defender.Attack));

        var newBoard = state.Board
            .Select(m => m.MinionId == attacker.MinionId ? updatedAttacker
                       : m.MinionId == defender.MinionId ? updatedDefender
                       : m)
            .ToList();
        state = state with { Board = newBoard };

        // Remove dead minions
        (state, var deathEvents) = RemoveDeadMinions(state, registry);
        events.AddRange(deathEvents);
    }

    return (state, events);
}

private static (GameState, IReadOnlyList<GameEvent>) RemoveDeadMinions(
    GameState state, CardHandlerRegistry registry)
{
    var events = new List<GameEvent>();
    var dead = state.Board.Where(m => m.Health <= 0).ToList();

    foreach (var minion in dead)
    {
        events.Add(new MinionDiedEvent(minion.MinionId, minion.OwnerId));
        var handler = registry.GetHandler(null); // deathrattles via handler in future
        var (newState, deathrattleEvents) = handler.OnDeath(state, minion);
        state = newState;
        events.AddRange(deathrattleEvents);
    }

    state = state with { Board = state.Board.Where(m => m.Health > 0).ToList() };
    return (state, events);
}
```

Also add to the using imports at top of `GameEngine.cs`:
```csharp
using CCG.GameLogic.Events;
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: `Passed: 15`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/GameEngine.cs CCG.GameLogic.Tests/Engine/GameEngineTests.cs
git commit -m "feat(gamelogic): implement Attack action with damage, death, and win condition"
```

---

## Task 13: GameEngine — EndTurn and Concede

**Files:**
- Modify: `CCG.GameLogic/Engine/GameEngine.cs`
- Modify: `CCG.GameLogic.Tests/Engine/GameEngineTests.cs`

- [ ] **Step 1: Write failing tests**

Append to `GameEngineTests.cs`:

```csharp
    // --- EndTurn ---

    [Fact]
    public void EndTurn_SwitchesActivePlayer()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", player2Id: "p2");

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.Turn.ActivePlayerId.Should().Be("p2");
    }

    [Fact]
    public void EndTurn_IncrementsTurnNumber()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", turnNumber: 3);

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.Turn.Number.Should().Be(4);
    }

    [Fact]
    public void EndTurn_NextPlayerManaIncrements()
    {
        // p2 starts with maxMana 2 — after EndTurn from p1 it should be 3
        var state = new GameState(
            "s1",
            GameStateBuilder.Player("p1", mana: 3),
            GameStateBuilder.Player("p2", mana: 2) with { MaxMana = 2 },
            Array.Empty<MinionOnBoard>(),
            new TurnState("p1", 4),
            new TimerState(75),
            GamePhase.InProgress,
            null
        );

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.GetPlayer("p2").MaxMana.Should().Be(3);
        newState.GetPlayer("p2").Mana.Should().Be(3);
    }

    [Fact]
    public void EndTurn_ManaCapAt10()
    {
        var state = new GameState(
            "s1",
            GameStateBuilder.Player("p1", mana: 10) with { MaxMana = 10 },
            GameStateBuilder.Player("p2", mana: 10) with { MaxMana = 10 },
            Array.Empty<MinionOnBoard>(),
            new TurnState("p1", 20),
            new TimerState(75),
            GamePhase.InProgress,
            null
        );

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.GetPlayer("p2").MaxMana.Should().Be(10);
    }

    [Fact]
    public void EndTurn_NextPlayerDrawsCard()
    {
        var card = GameStateBuilder.MinionCard("deck1");
        var p2 = GameStateBuilder.Player("p2", mana: 1) with
        {
            Deck = new[] { card },
            Hand = Array.Empty<Card>()
        };
        var state = new GameState(
            "s1",
            GameStateBuilder.Player("p1"),
            p2,
            Array.Empty<MinionOnBoard>(),
            new TurnState("p1", 2),
            new TimerState(75),
            GamePhase.InProgress,
            null
        );

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.GetPlayer("p2").Hand.Should().HaveCount(1);
        newState.GetPlayer("p2").DeckSize.Should().Be(0);
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.CardDrawnEvent);
    }

    [Fact]
    public void EndTurn_EmitsTurnEndedAndTurnStartedEvents()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        var (_, events) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.TurnEndedEvent);
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.TurnStartedEvent);
    }

    [Fact]
    public void EndTurn_OpponentMinionsBecomeFresh()
    {
        var exhausted = GameStateBuilder.Minion("m1", "p2", canAttack: false);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1",
            board: new[] { exhausted });

        var (newState, _) = _sut.Process(state, new CCG.GameLogic.Actions.EndTurnAction("p1"));

        newState.Board[0].CanAttack.Should().BeTrue();
    }

    // --- Concede ---

    [Fact]
    public void Concede_EndsGameWithOpponentAsWinner()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");

        var (newState, events) = _sut.Process(state, new CCG.GameLogic.Actions.ConcedeAction("p1"));

        newState.Phase.Should().Be(GamePhase.Ended);
        newState.WinnerId.Should().Be("p2");
        events.Should().ContainSingle(e => e is CCG.GameLogic.Events.GameEndedEvent);
    }
```

- [ ] **Step 2: Run — expect failures**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: 8 new failures.

- [ ] **Step 3: Implement `ProcessEndTurn` and `ProcessConcede`**

Replace the two skeletons in `GameEngine.cs`:

```csharp
private (GameState, IReadOnlyList<GameEvent>) ProcessEndTurn(
    GameState state, EndTurnAction action)
{
    var events = new List<GameEvent> { new TurnEndedEvent(action.PlayerId) };
    var opponent = state.GetOpponent(action.PlayerId);

    // Refresh opponent's minions
    var freshBoard = state.Board.Select(m =>
        m.OwnerId == opponent.PlayerId ? m with { CanAttack = true } : m
    ).ToList();
    state = state with { Board = freshBoard };

    // Increment opponent mana (cap 10, replenish fully)
    var newMaxMana = Math.Min(opponent.MaxMana + 1, 10);
    var updatedOpponent = opponent with { Mana = newMaxMana, MaxMana = newMaxMana };
    state = state.WithPlayer(updatedOpponent);

    // Draw card for opponent
    if (updatedOpponent.Deck.Count > 0)
    {
        var drawn = updatedOpponent.Deck[0];
        var afterDraw = updatedOpponent with
        {
            Hand = updatedOpponent.Hand.Append(drawn).ToList(),
            Deck = updatedOpponent.Deck.Skip(1).ToList()
        };
        state = state.WithPlayer(afterDraw);
        events.Add(new CardDrawnEvent(opponent.PlayerId, drawn, afterDraw.Hand.Count, afterDraw.DeckSize));
    }

    // Switch active player
    var newTurn = new TurnState(opponent.PlayerId, state.Turn.Number + 1);
    state = state with { Turn = newTurn, Timer = new TimerState(75) };

    events.Add(new TurnStartedEvent(opponent.PlayerId, 75));
    return (state, events);
}

private static (GameState, IReadOnlyList<GameEvent>) ProcessConcede(
    GameState state, ConcedeAction action)
{
    var opponent = state.GetOpponent(action.PlayerId);
    var newState = state with { Phase = GamePhase.Ended, WinnerId = opponent.PlayerId };
    return (newState, new[] { new GameEndedEvent(opponent.PlayerId, "Concede") });
}
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~GameEngineTests"
```

Expected: `Passed: 23`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Engine/GameEngine.cs CCG.GameLogic.Tests/Engine/GameEngineTests.cs
git commit -m "feat(gamelogic): implement EndTurn and Concede action processing"
```

---

## Task 14: DefaultCardHandler — standard effect interpreter

**Files:**
- Modify: `CCG.GameLogic/Handlers/DefaultCardHandler.cs`
- Create: `CCG.GameLogic.Tests/Handlers/DefaultCardHandlerTests.cs`

- [ ] **Step 1: Write failing tests**

```csharp
// CCG.GameLogic.Tests/Handlers/DefaultCardHandlerTests.cs
namespace CCG.GameLogic.Tests.Handlers;

using System.Text.Json;
using CCG.GameLogic.Events;
using CCG.GameLogic.Handlers;
using CCG.GameLogic.Models;
using CCG.GameLogic.Tests.TestHelpers;
using FluentAssertions;

public class DefaultCardHandlerTests
{
    private readonly DefaultCardHandler _sut = new();

    [Fact]
    public void OnPlay_DealDamageSpell_DealsCorrectDamageToTargetedMinion()
    {
        var target = GameStateBuilder.Minion("m1", "p2", health: 5);
        var state = GameStateBuilder.InProgress(board: new[] { target });
        var spell = SpellWithEffects("s1", $$$"""[{"type":"deal_damage","amount":3}]""", targeted: true);

        var (newState, events) = _sut.OnPlay(state, spell, "m1");

        newState.Board.First(m => m.MinionId == "m1").Health.Should().Be(2);
        events.Should().ContainSingle(e => e is DamageDealtEvent de && de.Amount == 3);
    }

    [Fact]
    public void OnPlay_DealDamageSpell_KillsMinion_EmitsMinionDied()
    {
        var target = GameStateBuilder.Minion("m1", "p2", health: 2);
        var state = GameStateBuilder.InProgress(board: new[] { target });
        var spell = SpellWithEffects("s1", $$$"""[{"type":"deal_damage","amount":5}]""", targeted: true);

        var (newState, events) = _sut.OnPlay(state, spell, "m1");

        newState.Board.Should().BeEmpty();
        events.Should().ContainSingle(e => e is MinionDiedEvent);
    }

    [Fact]
    public void OnPlay_DrawCardsEffect_DrawsCorrectNumberOfCards()
    {
        var card1 = GameStateBuilder.MinionCard("deck1");
        var card2 = GameStateBuilder.MinionCard("deck2");
        var p1 = GameStateBuilder.Player("p1") with { Deck = new[] { card1, card2 } };
        var state = new GameState("s", p1, GameStateBuilder.Player("p2"),
            Array.Empty<MinionOnBoard>(), new TurnState("p1", 1),
            new TimerState(75), GamePhase.InProgress, null);
        var spell = SpellWithEffects("s1", $$$"""[{"type":"draw_cards","count":2}]""");

        var (newState, events) = _sut.OnPlay(state, spell, null);

        newState.GetPlayer("p1").Hand.Should().HaveCount(2);
        events.OfType<CardDrawnEvent>().Should().HaveCount(2);
    }

    [Fact]
    public void OnPlay_GiveBuffEffect_BuffsTargetMinion()
    {
        var target = GameStateBuilder.Minion("m1", "p1", attack: 2, health: 3);
        var state = GameStateBuilder.InProgress(board: new[] { target });
        var spell = SpellWithEffects("s1", $$$"""[{"type":"give_buff","attack_bonus":2,"health_bonus":1}]""", targeted: true);

        var (newState, _) = _sut.OnPlay(state, spell, "m1");

        var buffed = newState.Board.First(m => m.MinionId == "m1");
        buffed.Attack.Should().Be(4);
        buffed.Health.Should().Be(4);
    }

    private static Card SpellWithEffects(string id, string effectsJson, bool targeted = false) =>
        new Card(id, "Test Spell", 3, CardType.Spell, CardRarity.Common,
            JsonDocument.Parse(
                $$$"""{"targeted":{{{targeted.ToString().ToLower()}}},"effects":{{{effectsJson}}}}"""
            ).RootElement.Clone());
}
```

- [ ] **Step 2: Run — expect failures**

```bash
dotnet test CCG.GameLogic.Tests --filter "FullyQualifiedName~DefaultCardHandlerTests"
```

Expected: `Failed: 4` (PlaySpell returns empty)

- [ ] **Step 3: Implement `PlaySpell` in `DefaultCardHandler.cs`**

Replace `PlaySpell` and add helper methods:

```csharp
private static (GameState, IReadOnlyList<GameEvent>) PlaySpell(
    GameState state, Card card, string? targetId)
{
    var events = new List<GameEvent>();

    if (!card.Definition.TryGetProperty("effects", out var effectsEl))
        return (state, events);

    foreach (var effect in effectsEl.EnumerateArray())
    {
        var type = effect.GetProperty("type").GetString();
        (state, var effectEvents) = type switch
        {
            "deal_damage" => ApplyDealDamage(state, effect, targetId),
            "draw_cards"  => ApplyDrawCards(state, effect, card),
            "give_buff"   => ApplyGiveBuff(state, effect, targetId),
            _             => (state, (IReadOnlyList<GameEvent>)Array.Empty<GameEvent>())
        };
        events.AddRange(effectEvents);
    }

    return (state, events);
}

private static (GameState, IReadOnlyList<GameEvent>) ApplyDealDamage(
    GameState state, JsonElement effect, string? targetId)
{
    if (targetId is null) return (state, Array.Empty<GameEvent>());
    var amount = effect.GetProperty("amount").GetInt32();
    var events = new List<GameEvent> { new DamageDealtEvent("spell", targetId, amount) };

    // Target is a minion
    var targetMinion = state.Board.FirstOrDefault(m => m.MinionId == targetId);
    if (targetMinion is not null)
    {
        var updated = targetMinion with { Health = targetMinion.Health - amount };
        var newBoard = state.Board.Select(m => m.MinionId == targetId ? updated : m).ToList();
        state = state with { Board = newBoard };

        if (updated.Health <= 0)
        {
            state = state with { Board = state.Board.Where(m => m.MinionId != targetId).ToList() };
            events.Add(new MinionDiedEvent(targetId, targetMinion.OwnerId));
        }
    }
    else
    {
        // Target is a hero
        var player = state.Player1.PlayerId == targetId ? state.Player1 : state.Player2;
        state = state.WithPlayer(player with { Health = player.Health - amount });
    }

    return (state, events);
}

private static (GameState, IReadOnlyList<GameEvent>) ApplyDrawCards(
    GameState state, JsonElement effect, Card sourceCard)
{
    var count = effect.GetProperty("count").GetInt32();
    var events = new List<GameEvent>();
    var player = state.GetPlayer(state.Turn.ActivePlayerId);

    for (var i = 0; i < count && player.Deck.Count > 0; i++)
    {
        var drawn = player.Deck[0];
        player = player with
        {
            Hand = player.Hand.Append(drawn).ToList(),
            Deck = player.Deck.Skip(1).ToList()
        };
        events.Add(new CardDrawnEvent(player.PlayerId, drawn, player.Hand.Count, player.DeckSize));
    }

    state = state.WithPlayer(player);
    return (state, events);
}

private static (GameState, IReadOnlyList<GameEvent>) ApplyGiveBuff(
    GameState state, JsonElement effect, string? targetId)
{
    if (targetId is null) return (state, Array.Empty<GameEvent>());

    var atkBonus = effect.TryGetProperty("attack_bonus", out var a) ? a.GetInt32() : 0;
    var hpBonus = effect.TryGetProperty("health_bonus", out var h) ? h.GetInt32() : 0;

    var targetMinion = state.Board.FirstOrDefault(m => m.MinionId == targetId);
    if (targetMinion is null) return (state, Array.Empty<GameEvent>());

    var buffed = targetMinion with
    {
        Attack = targetMinion.Attack + atkBonus,
        Health = targetMinion.Health + hpBonus
    };
    var newBoard = state.Board.Select(m => m.MinionId == targetId ? buffed : m).ToList();
    return (state with { Board = newBoard }, Array.Empty<GameEvent>());
}
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameLogic.Tests
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add CCG.GameLogic/Handlers/DefaultCardHandler.cs CCG.GameLogic.Tests/Handlers/DefaultCardHandlerTests.cs
git commit -m "feat(gamelogic): implement DefaultCardHandler spell effects (deal_damage, draw_cards, give_buff)"
```

---

## Task 15: PlayerToken

**Files:**
- Create: `CCG.GameServer/Sessions/PlayerToken.cs`
- Create: `CCG.GameServer.Tests/Sessions/PlayerTokenTests.cs`

- [ ] **Step 1: Write failing tests**

```csharp
// CCG.GameServer.Tests/Sessions/PlayerTokenTests.cs
namespace CCG.GameServer.Tests.Sessions;

using CCG.GameServer.Sessions;
using FluentAssertions;

public class PlayerTokenTests
{
    private const string Secret = "test-secret-key-32-bytes-minimum!";

    [Fact]
    public void Encode_Decode_RoundTrip_Succeeds()
    {
        var token = PlayerToken.Encode("session1", "player1", TokenRole.Player, Secret,
            expiresIn: TimeSpan.FromHours(4));

        var claims = PlayerToken.Decode(token, Secret);

        claims.Should().NotBeNull();
        claims!.SessionId.Should().Be("session1");
        claims.PlayerId.Should().Be("player1");
        claims.Role.Should().Be(TokenRole.Player);
    }

    [Fact]
    public void Decode_WrongSecret_ReturnsNull()
    {
        var token = PlayerToken.Encode("s1", "p1", TokenRole.Player, Secret,
            expiresIn: TimeSpan.FromHours(1));

        var claims = PlayerToken.Decode(token, "wrong-secret-that-is-also-long-enough!");

        claims.Should().BeNull();
    }

    [Fact]
    public void Decode_ExpiredToken_ReturnsNull()
    {
        var token = PlayerToken.Encode("s1", "p1", TokenRole.Player, Secret,
            expiresIn: TimeSpan.FromSeconds(-1));

        var claims = PlayerToken.Decode(token, Secret);

        claims.Should().BeNull();
    }

    [Fact]
    public void Encode_SpectatorRole_DecodesCorrectly()
    {
        var token = PlayerToken.Encode("s1", "spec1", TokenRole.Spectator, Secret,
            expiresIn: TimeSpan.FromHours(4));

        var claims = PlayerToken.Decode(token, Secret);

        claims!.Role.Should().Be(TokenRole.Spectator);
    }
}
```

- [ ] **Step 2: Run — expect compile failure (file doesn't exist)**

```bash
dotnet test CCG.GameServer.Tests --filter "FullyQualifiedName~PlayerTokenTests"
```

- [ ] **Step 3: Write `PlayerToken.cs`**

```csharp
// CCG.GameServer/Sessions/PlayerToken.cs
namespace CCG.GameServer.Sessions;

using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

public enum TokenRole { Player, Spectator }

public record TokenClaims(string SessionId, string PlayerId, TokenRole Role);

public static class PlayerToken
{
    public static string Encode(
        string sessionId, string playerId, TokenRole role,
        string secret, TimeSpan expiresIn)
    {
        var payload = JsonSerializer.Serialize(new
        {
            sessionId,
            playerId,
            role = role.ToString(),
            exp = DateTimeOffset.UtcNow.Add(expiresIn).ToUnixTimeSeconds()
        });

        var payloadBytes = Encoding.UTF8.GetBytes(payload);
        var payloadB64 = Convert.ToBase64String(payloadBytes);
        var sig = Sign(payloadB64, secret);
        return $"{payloadB64}.{sig}";
    }

    public static TokenClaims? Decode(string token, string secret)
    {
        var parts = token.Split('.');
        if (parts.Length != 2) return null;

        var expectedSig = Sign(parts[0], secret);
        if (!CryptographicOperations.FixedTimeEquals(
                Encoding.UTF8.GetBytes(expectedSig),
                Encoding.UTF8.GetBytes(parts[1])))
            return null;

        var payload = JsonDocument.Parse(
            Encoding.UTF8.GetString(Convert.FromBase64String(parts[0])));

        var exp = payload.RootElement.GetProperty("exp").GetInt64();
        if (DateTimeOffset.UtcNow.ToUnixTimeSeconds() > exp) return null;

        return new TokenClaims(
            SessionId: payload.RootElement.GetProperty("sessionId").GetString()!,
            PlayerId: payload.RootElement.GetProperty("playerId").GetString()!,
            Role: Enum.Parse<TokenRole>(payload.RootElement.GetProperty("role").GetString()!)
        );
    }

    private static string Sign(string data, string secret)
    {
        var keyBytes = Encoding.UTF8.GetBytes(secret);
        var dataBytes = Encoding.UTF8.GetBytes(data);
        var hash = HMACSHA256.HashData(keyBytes, dataBytes);
        return Convert.ToBase64String(hash);
    }
}
```

- [ ] **Step 4: Run — expect all pass**

```bash
dotnet test CCG.GameServer.Tests --filter "FullyQualifiedName~PlayerTokenTests"
```

Expected: `Passed: 4`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameServer/Sessions/PlayerToken.cs CCG.GameServer.Tests/Sessions/PlayerTokenTests.cs
git commit -m "feat(gameserver): add HMAC-SHA256 player token encode/decode"
```

---

## Task 16: ConnectionManager

**Files:**
- Create: `CCG.GameServer/WebSockets/ConnectionManager.cs`

- [ ] **Step 1: Write `ConnectionManager.cs`**

```csharp
// CCG.GameServer/WebSockets/ConnectionManager.cs
namespace CCG.GameServer.WebSockets;

using System.Collections.Concurrent;
using System.Net.WebSockets;
using CCG.GameServer.Sessions;

public record Connection(WebSocket Socket, string PlayerId, TokenRole Role);

public class ConnectionManager
{
    private readonly ConcurrentDictionary<string, List<Connection>> _sessions = new();

    public void Add(string sessionId, Connection connection)
    {
        _sessions.AddOrUpdate(sessionId,
            _ => new List<Connection> { connection },
            (_, list) => { lock (list) { list.Add(connection); } return list; });
    }

    public void Remove(string sessionId, string playerId)
    {
        if (!_sessions.TryGetValue(sessionId, out var list)) return;
        lock (list) { list.RemoveAll(c => c.PlayerId == playerId); }
    }

    public IReadOnlyList<Connection> GetConnections(string sessionId)
    {
        if (!_sessions.TryGetValue(sessionId, out var list)) return Array.Empty<Connection>();
        lock (list) { return list.ToList(); }
    }

    public Connection? GetConnection(string sessionId, string playerId)
    {
        if (!_sessions.TryGetValue(sessionId, out var list)) return null;
        lock (list) { return list.FirstOrDefault(c => c.PlayerId == playerId); }
    }

    public void RemoveSession(string sessionId) => _sessions.TryRemove(sessionId, out _);
}
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/WebSockets/ConnectionManager.cs
git commit -m "feat(gameserver): add ConnectionManager for tracking WebSocket connections"
```

---

## Task 17: GameSession — action queue and processing loop

**Files:**
- Create: `CCG.GameServer/Sessions/GameSession.cs`
- Create: `CCG.GameServer.Tests/Sessions/GameSessionTests.cs`

- [ ] **Step 1: Write failing tests**

```csharp
// CCG.GameServer.Tests/Sessions/GameSessionTests.cs
namespace CCG.GameServer.Tests.Sessions;

using CCG.GameLogic.Actions;
using CCG.GameLogic.Engine;
using CCG.GameLogic.Handlers;
using CCG.GameLogic.Models;
using CCG.GameLogic.Tests.TestHelpers;
using CCG.GameServer.Sessions;
using CCG.GameServer.WebSockets;
using FluentAssertions;

public class GameSessionTests
{
    private static GameSession CreateSession(
        GameState? initialState = null,
        ConnectionManager? connections = null)
    {
        var state = initialState ?? GameStateBuilder.InProgress();
        var cm = connections ?? new ConnectionManager();
        var engine = new GameEngine(new CardHandlerRegistry());
        return new GameSession("s1", state, engine, cm,
            platformApiUrl: "http://platform/internal",
            tokenSecret: "test-secret-key-32-bytes-minimum!");
    }

    [Fact]
    public async Task EnqueueAction_ValidAction_UpdatesState()
    {
        var card = GameStateBuilder.MinionCard("c1", manaCost: 2);
        var state = GameStateBuilder.InProgress(activePlayerId: "p1", p1Mana: 3,
            p1Hand: new[] { card });
        var session = CreateSession(state);
        await session.StartAsync(CancellationToken.None);

        await session.EnqueueActionAsync(new PlayCardAction("p1", "c1", null));
        await Task.Delay(100); // allow processing loop to run

        session.CurrentState.Board.Should().HaveCount(1);
        await session.StopAsync(CancellationToken.None);
    }

    [Fact]
    public async Task EnqueueAction_InvalidAction_StateUnchanged()
    {
        var state = GameStateBuilder.InProgress(activePlayerId: "p1");
        var session = CreateSession(state);
        await session.StartAsync(CancellationToken.None);

        // p2 tries to play a card on p1's turn
        await session.EnqueueActionAsync(new EndTurnAction("p2"));
        await Task.Delay(100);

        session.CurrentState.Turn.ActivePlayerId.Should().Be("p1");
        await session.StopAsync(CancellationToken.None);
    }
}
```

- [ ] **Step 2: Run — expect compile error**

```bash
dotnet test CCG.GameServer.Tests --filter "FullyQualifiedName~GameSessionTests"
```

- [ ] **Step 3: Write `GameSession.cs` core**

```csharp
// CCG.GameServer/Sessions/GameSession.cs
namespace CCG.GameServer.Sessions;

using System.Net.WebSockets;
using System.Text;
using System.Text.Json;
using System.Threading.Channels;
using CCG.GameLogic.Actions;
using CCG.GameLogic.Engine;
using CCG.GameLogic.Events;
using CCG.GameLogic.Models;
using CCG.GameServer.WebSockets;

public class GameSession
{
    private readonly string _sessionId;
    private readonly GameEngine _engine;
    private readonly ConnectionManager _connections;
    private readonly string _platformApiUrl;
    private readonly string _tokenSecret;
    private readonly Channel<GameAction> _actionChannel;
    private GameState _state;
    private CancellationTokenSource _cts = new();

    public GameState CurrentState => _state;

    public GameSession(
        string sessionId,
        GameState initialState,
        GameEngine engine,
        ConnectionManager connections,
        string platformApiUrl,
        string tokenSecret)
    {
        _sessionId = sessionId;
        _state = initialState;
        _engine = engine;
        _connections = connections;
        _platformApiUrl = platformApiUrl;
        _tokenSecret = tokenSecret;
        _actionChannel = Channel.CreateBounded<GameAction>(
            new BoundedChannelOptions(100) { FullMode = BoundedChannelFullMode.Wait });
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        _ = ProcessActionsAsync(_cts.Token);
    }

    public Task StopAsync(CancellationToken _)
    {
        _cts.Cancel();
        return Task.CompletedTask;
    }

    public async Task EnqueueActionAsync(GameAction action) =>
        await _actionChannel.Writer.WriteAsync(action);

    private async Task ProcessActionsAsync(CancellationToken ct)
    {
        await foreach (var action in _actionChannel.Reader.ReadAllAsync(ct))
        {
            try
            {
                var (newState, events) = _engine.Process(_state, action);
                _state = newState;
                await BroadcastEventsAsync(events, ct);

                if (newState.Phase == GamePhase.Ended)
                    await HandleGameEndedAsync(newState, ct);
            }
            catch (InvalidOperationException)
            {
                // Invalid action — discard silently
            }
        }
    }

    // Implemented in Task 18
    protected virtual Task BroadcastEventsAsync(
        IReadOnlyList<GameEvent> events, CancellationToken ct) =>
        Task.CompletedTask;

    // Implemented in Task 24
    private Task HandleGameEndedAsync(GameState state, CancellationToken ct) =>
        Task.CompletedTask;
}
```

- [ ] **Step 4: Run — expect pass**

```bash
dotnet test CCG.GameServer.Tests --filter "FullyQualifiedName~GameSessionTests"
```

Expected: `Passed: 2`

- [ ] **Step 5: Commit**

```bash
git add CCG.GameServer/Sessions/GameSession.cs CCG.GameServer.Tests/Sessions/GameSessionTests.cs
git commit -m "feat(gameserver): add GameSession with action channel and processing loop"
```

---

## Task 18: GameSession — scoped event broadcasting

**Files:**
- Modify: `CCG.GameServer/Sessions/GameSession.cs`

- [ ] **Step 1: Implement `BroadcastEventsAsync`**

Replace the `BroadcastEventsAsync` stub and add helpers in `GameSession.cs`:

```csharp
protected override async Task BroadcastEventsAsync(
    IReadOnlyList<GameEvent> events, CancellationToken ct)
{
    var connections = _connections.GetConnections(_sessionId);
    foreach (var connection in connections)
    {
        foreach (var evt in events)
        {
            var scoped = ScopeEvent(evt, connection.PlayerId, connection.Role);
            if (scoped is null) continue;
            await SendAsync(connection.Socket, scoped, ct);
        }
    }
}

// Returns null to suppress the event for this recipient
private static GameEvent? ScopeEvent(GameEvent evt, string recipientId, TokenRole role) =>
    evt switch
    {
        // Opponent sees CardDrawn with card hidden
        CardDrawnEvent e when e.PlayerId != recipientId && role == TokenRole.Player =>
            e with { DrawnCard = null },
        _ => evt
    };

public async Task SendSnapshotAsync(string playerId, CancellationToken ct)
{
    var connection = _connections.GetConnection(_sessionId, playerId);
    if (connection is null) return;

    var player = _state.GetPlayer(playerId);
    var opponent = _state.GetOpponent(playerId);

    var snapshot = new StateSnapshotEvent(
        MyHealth: player.Health,
        MyMana: player.Mana,
        MyMaxMana: player.MaxMana,
        MyHand: player.Hand,
        MyDeckSize: player.DeckSize,
        OpponentHealth: opponent.Health,
        OpponentMana: opponent.Mana,
        OpponentMaxMana: opponent.MaxMana,
        OpponentHandCount: opponent.Hand.Count,
        OpponentDeckSize: opponent.DeckSize,
        Board: _state.Board,
        Turn: _state.Turn,
        Timer: _state.Timer
    );

    await SendAsync(connection.Socket, snapshot, ct);
}

private static async Task SendAsync(WebSocket socket, GameEvent evt, CancellationToken ct)
{
    if (socket.State != WebSocketState.Open) return;
    var json = JsonSerializer.Serialize(evt, new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    });
    var bytes = Encoding.UTF8.GetBytes(json);
    await socket.SendAsync(bytes, WebSocketMessageType.Text, true, ct);
}
```

Remove the `virtual` keyword from the original stub (`protected virtual Task BroadcastEventsAsync`) and make it `protected async Task`. The method above replaces it fully.

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/Sessions/GameSession.cs
git commit -m "feat(gameserver): implement scoped event broadcasting and state snapshot"
```

---

## Task 19: GameSession — turn timer

**Files:**
- Modify: `CCG.GameServer/Sessions/GameSession.cs`

- [ ] **Step 1: Add timer fields and wiring to `GameSession.cs`**

Add fields after `_cts`:

```csharp
private CancellationTokenSource _turnTimerCts = new();
```

Add this method:

```csharp
private async Task StartTurnTimerAsync(string activePlayerId, CancellationToken sessionCt)
{
    _turnTimerCts.Cancel();
    _turnTimerCts = CancellationTokenSource.CreateLinkedTokenSource(sessionCt);
    var timerToken = _turnTimerCts.Token;

    try
    {
        await Task.Delay(TimeSpan.FromSeconds(75), timerToken);
        // Timer expired — auto-end turn
        await EnqueueActionAsync(new EndTurnAction(activePlayerId));
    }
    catch (OperationCanceledException)
    {
        // Turn ended normally or session stopped
    }
}
```

Call `StartTurnTimerAsync` from `ProcessActionsAsync` after each successful action, when the game is `InProgress`:

```csharp
// Inside ProcessActionsAsync, after _state = newState:
if (newState.Phase == GamePhase.InProgress)
{
    _ = StartTurnTimerAsync(newState.Turn.ActivePlayerId, ct);
}
```

Also start the timer when the game transitions from `WaitingForPlayers` to `InProgress`. Add a `BothPlayersConnected` trigger method:

```csharp
public async Task NotifyPlayerConnectedAsync(string playerId, CancellationToken ct)
{
    if (_state.Phase == GamePhase.WaitingForPlayers)
    {
        var connections = _connections.GetConnections(_sessionId);
        var playerIds = new[] { _state.Player1.PlayerId, _state.Player2.PlayerId };
        var bothConnected = playerIds.All(pid =>
            connections.Any(c => c.PlayerId == pid && c.Role == TokenRole.Player));

        if (bothConnected)
        {
            _state = _state with { Phase = GamePhase.InProgress };
            await BroadcastEventsAsync(
                new[] { new TurnStartedEvent(_state.Turn.ActivePlayerId, 75) }, ct);
            _ = StartTurnTimerAsync(_state.Turn.ActivePlayerId, ct);
        }
    }

    await SendSnapshotAsync(playerId, ct);
}
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/Sessions/GameSession.cs
git commit -m "feat(gameserver): add turn timer with auto-EndTurn on expiry"
```

---

## Task 20: GameSession — reconnection

**Files:**
- Modify: `CCG.GameServer/Sessions/GameSession.cs`

- [ ] **Step 1: Add reconnection state and disconnect handler**

Add fields:

```csharp
private readonly Dictionary<string, (DateTimeOffset DisconnectedAt, List<GameEvent> Buffer)>
    _disconnected = new();
private readonly List<GameEvent> _broadcastBuffer = new();
```

Add `NotifyPlayerDisconnectedAsync`:

```csharp
public void NotifyPlayerDisconnected(string playerId)
{
    _connections.Remove(_sessionId, playerId);
    _disconnected[playerId] = (DateTimeOffset.UtcNow, new List<GameEvent>(_broadcastBuffer));
    _ = ForfeitAfterTimeoutAsync(playerId);
}

private async Task ForfeitAfterTimeoutAsync(string playerId)
{
    await Task.Delay(TimeSpan.FromSeconds(60));
    if (_disconnected.ContainsKey(playerId))
    {
        await EnqueueActionAsync(new ConcedeAction(playerId));
        _disconnected.Remove(playerId);
    }
}
```

Update `BroadcastEventsAsync` to also append to the buffer:

```csharp
// At top of BroadcastEventsAsync, before the loop:
_broadcastBuffer.AddRange(events);
```

Add `HandleReconnectAsync`:

```csharp
public async Task HandleReconnectAsync(
    string playerId, WebSocket socket, TokenRole role, CancellationToken ct)
{
    if (!_disconnected.TryGetValue(playerId, out var entry)) return;

    _disconnected.Remove(playerId);
    var connection = new Connection(socket, playerId, role);
    _connections.Add(_sessionId, connection);

    // Send current state snapshot
    await SendSnapshotAsync(playerId, ct);

    // Replay buffered events since disconnect
    foreach (var evt in entry.Buffer)
    {
        var scoped = ScopeEvent(evt, playerId, role);
        if (scoped is not null)
            await SendAsync(socket, scoped, ct);
    }
}

public bool IsDisconnected(string playerId) => _disconnected.ContainsKey(playerId);
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/Sessions/GameSession.cs
git commit -m "feat(gameserver): add 60-second reconnection window with event buffer and auto-forfeit"
```

---

## Task 21: WebSocketHandler

**Files:**
- Create: `CCG.GameServer/WebSockets/WebSocketHandler.cs`

- [ ] **Step 1: Write `WebSocketHandler.cs`**

```csharp
// CCG.GameServer/WebSockets/WebSocketHandler.cs
namespace CCG.GameServer.WebSockets;

using System.Net.WebSockets;
using System.Text;
using System.Text.Json;
using CCG.GameLogic.Actions;
using CCG.GameServer.Sessions;

public class WebSocketHandler(
    SessionManager sessionManager,
    ConnectionManager connectionManager,
    IConfiguration configuration)
{
    private readonly string _secret = configuration["TokenSecret"]!;

    public async Task HandleAsync(HttpContext context)
    {
        if (!context.WebSockets.IsWebSocketRequest)
        {
            context.Response.StatusCode = 400;
            return;
        }

        var token = context.Request.Query["token"].ToString();
        var claims = PlayerToken.Decode(token, _secret);
        if (claims is null)
        {
            context.Response.StatusCode = 401;
            return;
        }

        var session = sessionManager.Get(claims.SessionId);
        if (session is null)
        {
            context.Response.StatusCode = 404;
            return;
        }

        using var socket = await context.WebSockets.AcceptWebSocketAsync();
        var connection = new Connection(socket, claims.PlayerId, claims.Role);

        if (session.IsDisconnected(claims.PlayerId))
        {
            await session.HandleReconnectAsync(claims.PlayerId, socket, claims.Role,
                context.RequestAborted);
        }
        else
        {
            connectionManager.Add(claims.SessionId, connection);
            await session.NotifyPlayerConnectedAsync(claims.PlayerId, context.RequestAborted);
        }

        await ReceiveLoopAsync(socket, session, claims, context.RequestAborted);
        session.NotifyPlayerDisconnected(claims.PlayerId);
    }

    private static async Task ReceiveLoopAsync(
        WebSocket socket,
        GameSession session,
        TokenClaims claims,
        CancellationToken ct)
    {
        var buffer = new byte[4096];
        while (socket.State == WebSocketState.Open && !ct.IsCancellationRequested)
        {
            var result = await socket.ReceiveAsync(buffer, ct);
            if (result.MessageType == WebSocketMessageType.Close) break;
            if (claims.Role == TokenRole.Spectator) continue; // spectators cannot send actions

            var json = Encoding.UTF8.GetString(buffer, 0, result.Count);
            var action = ParseAction(json, claims.PlayerId);
            if (action is not null)
                await session.EnqueueActionAsync(action);
        }
    }

    private static GameAction? ParseAction(string json, string playerId)
    {
        try
        {
            using var doc = JsonDocument.Parse(json);
            var type = doc.RootElement.GetProperty("type").GetString();
            return type switch
            {
                "PlayCard" => new PlayCardAction(
                    playerId,
                    doc.RootElement.GetProperty("cardId").GetString()!,
                    doc.RootElement.TryGetProperty("targetId", out var t)
                        ? t.GetString() : null),
                "Attack" => new AttackAction(
                    playerId,
                    doc.RootElement.GetProperty("attackerId").GetString()!,
                    doc.RootElement.GetProperty("targetId").GetString()!),
                "EndTurn" => new EndTurnAction(playerId),
                "Concede"  => new ConcedeAction(playerId),
                _          => null
            };
        }
        catch { return null; }
    }
}
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/WebSockets/WebSocketHandler.cs
git commit -m "feat(gameserver): add WebSocketHandler — auth, receive loop, action parsing"
```

---

## Task 22: SessionManager

**Files:**
- Create: `CCG.GameServer/Sessions/SessionManager.cs`

- [ ] **Step 1: Write `SessionManager.cs`**

```csharp
// CCG.GameServer/Sessions/SessionManager.cs
namespace CCG.GameServer.Sessions;

using System.Collections.Concurrent;
using CCG.GameLogic.Engine;
using CCG.GameLogic.Handlers;
using CCG.GameLogic.Models;
using CCG.GameServer.WebSockets;

public class SessionManager(
    ConnectionManager connectionManager,
    IConfiguration configuration)
{
    private readonly ConcurrentDictionary<string, GameSession> _sessions = new();
    private readonly string _platformApiUrl = configuration["PlatformApiUrl"]!;
    private readonly string _tokenSecret = configuration["TokenSecret"]!;

    public (GameSession Session, string P1Token, string P2Token) Create(
        string sessionId,
        string player1Id,
        string player2Id,
        IReadOnlyList<Card> deck1,
        IReadOnlyList<Card> deck2)
    {
        var registry = new CardHandlerRegistry();
        var engine = new GameEngine(registry);
        var (initialState, _) = GameEngine.InitializeGame(sessionId, player1Id, player2Id, deck1, deck2);

        var session = new GameSession(sessionId, initialState, engine, connectionManager,
            _platformApiUrl, _tokenSecret);

        _sessions[sessionId] = session;
        _ = session.StartAsync(CancellationToken.None);

        var expiry = TimeSpan.FromHours(4);
        var p1Token = PlayerToken.Encode(sessionId, player1Id, TokenRole.Player, _tokenSecret, expiry);
        var p2Token = PlayerToken.Encode(sessionId, player2Id, TokenRole.Player, _tokenSecret, expiry);

        return (session, p1Token, p2Token);
    }

    public GameSession? Get(string sessionId) =>
        _sessions.TryGetValue(sessionId, out var s) ? s : null;

    public void Remove(string sessionId)
    {
        if (_sessions.TryRemove(sessionId, out var session))
        {
            session.StopAsync(CancellationToken.None);
            connectionManager.RemoveSession(sessionId);
        }
    }
}
```

- [ ] **Step 2: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add CCG.GameServer/Sessions/SessionManager.cs
git commit -m "feat(gameserver): add SessionManager — create, index, and dispose sessions"
```

---

## Task 23: Internal HTTP API

**Files:**
- Create: `CCG.GameServer/Api/Models/CreateSessionRequest.cs`
- Create: `CCG.GameServer/Api/Models/CreateSessionResponse.cs`
- Create: `CCG.GameServer/Api/InternalController.cs`

- [ ] **Step 1: Write DTOs**

```csharp
// CCG.GameServer/Api/Models/CreateSessionRequest.cs
namespace CCG.GameServer.Api.Models;

using CCG.GameLogic.Models;

public record PlayerDeckDto(string PlayerId, IReadOnlyList<Card> Deck);

public record CreateSessionRequest(
    string SessionId,
    PlayerDeckDto Player1,
    PlayerDeckDto Player2
);
```

```csharp
// CCG.GameServer/Api/Models/CreateSessionResponse.cs
namespace CCG.GameServer.Api.Models;

public record CreateSessionResponse(
    string SessionId,
    string P1Token,
    string P2Token
);
```

- [ ] **Step 2: Write `InternalController.cs`**

```csharp
// CCG.GameServer/Api/InternalController.cs
namespace CCG.GameServer.Api;

using CCG.GameServer.Api.Models;
using CCG.GameServer.Sessions;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("internal")]
public class InternalController(SessionManager sessionManager) : ControllerBase
{
    [HttpPost("session/create")]
    public IActionResult CreateSession([FromBody] CreateSessionRequest request)
    {
        var (_, p1Token, p2Token) = sessionManager.Create(
            request.SessionId,
            request.Player1.PlayerId,
            request.Player2.PlayerId,
            request.Player1.Deck,
            request.Player2.Deck
        );

        return Ok(new CreateSessionResponse(request.SessionId, p1Token, p2Token));
    }
}
```

- [ ] **Step 3: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 4: Commit**

```bash
git add CCG.GameServer/Api/
git commit -m "feat(gameserver): add POST /internal/session/create endpoint"
```

---

## Task 24: PlatformApiClient

**Files:**
- Create: `CCG.GameServer/Api/PlatformApiClient.cs`

- [ ] **Step 1: Write `PlatformApiClient.cs`**

```csharp
// CCG.GameServer/Api/PlatformApiClient.cs
namespace CCG.GameServer.Api;

using System.Text;
using System.Text.Json;

public record SessionResultPayload(
    string SessionId,
    string WinnerId,
    string LoserId,
    string Reason,
    DateTimeOffset EndedAt
);

public class PlatformApiClient(HttpClient httpClient)
{
    public async Task ReportResultAsync(SessionResultPayload payload, CancellationToken ct)
    {
        var json = JsonSerializer.Serialize(payload,
            new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase });
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        await httpClient.PostAsync("/internal/session/result", content, ct);
    }
}
```

- [ ] **Step 2: Wire `HandleGameEndedAsync` in `GameSession.cs`**

Replace the stub in `GameSession.cs`:

Add field:
```csharp
private readonly PlatformApiClient? _platformClient;
```

Update constructor to accept `PlatformApiClient?`:
```csharp
public GameSession(
    string sessionId,
    GameState initialState,
    GameEngine engine,
    ConnectionManager connections,
    string platformApiUrl,
    string tokenSecret,
    PlatformApiClient? platformClient = null)
{
    // ... existing assignments ...
    _platformClient = platformClient;
}
```

Replace `HandleGameEndedAsync`:
```csharp
private async Task HandleGameEndedAsync(GameState state, CancellationToken ct)
{
    if (_platformClient is null || state.WinnerId is null) return;

    var opponent = state.GetOpponent(state.WinnerId);
    var payload = new SessionResultPayload(
        SessionId: _sessionId,
        WinnerId: state.WinnerId,
        LoserId: opponent.PlayerId,
        Reason: "GameEnded",
        EndedAt: DateTimeOffset.UtcNow
    );
    await _platformClient.ReportResultAsync(payload, ct);
}
```

- [ ] **Step 3: Verify build**

```bash
dotnet build CCG.GameServer
```

Expected: `Build succeeded.`

- [ ] **Step 4: Commit**

```bash
git add CCG.GameServer/Api/PlatformApiClient.cs CCG.GameServer/Sessions/GameSession.cs
git commit -m "feat(gameserver): add PlatformApiClient for result reporting"
```

---

## Task 25: Program.cs — DI and middleware

**Files:**
- Modify: `CCG.GameServer/Program.cs`
- Create: `CCG.GameServer/appsettings.json` additions

- [ ] **Step 1: Update `appsettings.json`**

Add to the JSON object in `CCG.GameServer/appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" }
  },
  "AllowedHosts": "*",
  "TokenSecret": "change-me-in-production-min-32-chars!!",
  "PlatformApiUrl": "http://platform-api:3000"
}
```

- [ ] **Step 2: Write `Program.cs`**

```csharp
// CCG.GameServer/Program.cs
using CCG.GameServer.Api;
using CCG.GameServer.Sessions;
using CCG.GameServer.WebSockets;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddSingleton<ConnectionManager>();
builder.Services.AddSingleton<SessionManager>();
builder.Services.AddScoped<WebSocketHandler>();
builder.Services.AddHttpClient<PlatformApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["PlatformApiUrl"]!);
});

var app = builder.Build();

app.UseWebSockets(new WebSocketOptions { KeepAliveInterval = TimeSpan.FromSeconds(30) });

app.Map("/ws", async context =>
{
    var handler = context.RequestServices.GetRequiredService<WebSocketHandler>();
    await handler.HandleAsync(context);
});

app.MapControllers();
app.Run();
```

- [ ] **Step 3: Final build and test run**

```bash
dotnet build CCG.sln
dotnet test CCG.sln
```

Expected: `Build succeeded.`, all tests pass.

- [ ] **Step 4: Commit**

```bash
git add CCG.GameServer/Program.cs CCG.GameServer/appsettings.json
git commit -m "feat(gameserver): wire DI, WebSocket middleware, and startup in Program.cs"
```

---

## Self-Review

### Spec coverage

| Spec requirement | Task(s) |
|---|---|
| POST /internal/session/create | Task 23 |
| POST /internal/session/result (outbound) | Task 24 |
| Session Manager — creates sessions, issues tokens, tracks sessions | Tasks 22, 15 |
| GameSession — owns authoritative state, serialises action processing | Task 17 |
| GameSession — turn timer | Task 19 |
| GameSession — event broadcasting | Task 18 |
| GameSession — reconnection buffer | Task 20 |
| WebSocket Handler — auth on connect | Task 21 |
| WebSocket Handler — routes to action queue | Task 21 |
| WebSocket Handler — spectator connections | Task 21 |
| Action types: PlayCard, Attack, EndTurn, Concede | Task 4 |
| Event types: all 8 + StateSnapshot | Task 5 |
| CardDrawn scoped per recipient | Task 18 |
| GameState shape | Tasks 2–3 |
| GamePhase: WaitingForPlayers → InProgress → Ended | Tasks 2, 19 |
| Reconnection — 60s window, snapshot + buffer on reconnect | Task 20 |
| Reconnection — forfeit on timeout | Task 20 |
| CCG.GameLogic — no framework deps | Task 1 (classlib target) |
| ICardHandler + CardHandlerRegistry + DefaultCardHandler | Task 6, 14 |
| Turn time limit: 75 seconds | Tasks 19, 13 |
| Win condition (health ≤ 0) | Task 12 |

**No gaps found.**

### Placeholder scan

No TBD, TODO, or "similar to Task N" references found. All code steps are complete.

### Type consistency

- `CardDrawnEvent` — defined with `DrawnCard` in Task 5, used with `DrawnCard` in Tasks 13, 18. ✓
- `MinionDiedEvent` — defined with `(MinionId, OwnerId)` in Task 5, emitted correctly in Tasks 12, 14. ✓
- `TokenClaims` — defined in Task 15, consumed in Task 21. ✓
- `Connection` — defined in Task 16, used in Tasks 18, 21. ✓
- `SessionResultPayload` — defined and consumed in Task 24. ✓
- `GameEngine.Process` — returns `(GameState, IReadOnlyList<GameEvent>)` throughout. ✓
