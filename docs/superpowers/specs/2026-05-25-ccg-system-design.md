# CCG System Design

**Date:** 2026-05-25
**Status:** Approved
**Scope:** Top-level system architecture — each subsystem has its own spec

---

## Overview

An online 1v1 collectible card game (Hearthstone-clone for initial scope). Players register, build decks from their card collection, queue for ranked matches based on MMR, and play real-time turn-based games against each other. Games can be spectated. The server is fully authoritative — the client is a pure event renderer.

### Out of scope for initial release
- More than 1v1 formats
- Card acquisition economy (packs, crafting, shop)
- Mobile-specific builds (desktop/mobile both targeted via game engine, but no app store pipeline in scope)
- Ranked seasons, ladders, or end-of-season rewards
- Social features (friends list, chat, replays)

---

## Architecture: Core Split

Two deployable services plus a game engine client.

| Service | Language | Responsibility |
|---|---|---|
| Game Server | C# / .NET | Authoritative game state, WebSocket connections, action processing, event broadcasting |
| Platform API | TypeScript / Node.js | Auth, card collection, deck management, matchmaking, MMR |
| Client | Unity or Godot (TBD) | Renders game state from events, sends player actions |

**Why this split:** The Game Server holds thousands of persistent WebSocket connections and processes discrete state machine transitions. The Platform API is stateless HTTP traffic. These have different scaling characteristics and different runtime requirements, making the seam natural. Two services is the right complexity for the 1k–10k concurrent player target: independent scaling without the operational overhead of full microservices.

**Shared game logic:** Card rules, game state structure, and action processing logic live in a standalone `CCG.GameLogic` C# class library with no engine or framework dependencies. The Game Server references it as the authoritative engine. If Unity is chosen as the client, it can reference the same library for local state rendering.

`CCG.GameLogic` maintains a `CardHandlerRegistry`. Every card has a JSON record for static attributes (cost, attack, health, keywords, standard effects). Cards with unique text effects additionally have a registered C# class implementing `ICardHandler`. The card's JSON record carries an optional `handler` field naming its class; cards without one use `DefaultCardHandler`, which executes standard effects directly from JSON. This keeps standard-mechanic cards as pure data while giving unique-effect cards full code expressiveness — the boundary between data and code is explicit.

---

## Subsystems and Build Order

Each subsystem is a separate spec → plan → implementation cycle, built in dependency order:

| Order | Subsystem | Service | Key deliverable |
|---|---|---|---|
| 1 | Game Server | C# / .NET | WebSocket sessions, authoritative game loop, action/event protocol |
| 2 | Auth & Accounts | Platform API | Register, login, JWT issuance, refresh tokens |
| 3 | Collection & Decks | Platform API | Card catalog, player collection, deck CRUD and validation |
| 4 | Matchmaking | Platform API | Queue, MMR-based pairing, session handoff to Game Server |
| 5 | Client | Unity / Godot | All screens, REST+WebSocket connections, event rendering |

---

## Data Flow: Full Game Lifecycle

### Phase 1 — Auth & Setup
1. Client POSTs credentials to Platform API → receives JWT access token + refresh token
2. Client fetches card collection and deck list
3. Player selects a deck and POSTs to `/queue/join`

### Phase 2 — Matchmaking
1. Platform API enqueues the player in Redis (sorted set by MMR)
2. Matchmaker finds two players within MMR range (range expands over wait time)
3. Platform API POSTs to Game Server `/internal/session/create` with both players' IDs and full deck lists
4. Game Server initialises a `GameSession`, returns `{ sessionId, p1Token, p2Token }`
5. Platform API delivers `{ ws_url, playerToken }` to both clients (via polling or push — see Open Questions #3)

### Phase 3 — Live Game
1. Both clients open WebSocket to Game Server, authenticate with their player token
2. Game Server sends initial `GameState` snapshot to both players
3. Active player sends an action → Game Server validates → mutates state → broadcasts events to both players and spectators
4. Client renders events only — never mutates local state speculatively

### Phase 4 — Game End
1. Game Server broadcasts `GameEnded` event to players and spectators
2. Game Server POSTs result to Platform API `/internal/session/result`
3. Platform API updates MMR (Elo), writes match record to PostgreSQL, clears session from Redis

---

## Game Server

**Runtime:** ASP.NET Core, C# / .NET

### Internal components

- **Internal HTTP API** — bound to internal network only, never publicly exposed. Endpoints: `POST /internal/session/create`, `POST /internal/session/result`
- **Session Manager** — creates `GameSession` instances, issues player tokens, tracks active sessions, handles timeout and cleanup
- **GameSession** — one instance per active match. Owns the authoritative `GameState`, serialises action processing (no concurrent state mutation), runs the turn timer, manages event broadcasting and reconnection buffer
- **WebSocket Handler** — authenticates tokens on connect, routes messages to the correct session's action queue, detects disconnects, manages spectator connections

### Action types (client → server)
```
PlayCard   { cardId, targets? }
Attack     { attackerId, targetId }
EndTurn    { }
Concede    { }
```

### Event types (server → client, scoped per recipient)
```
CardDrawn        owner sees card identity; opponent sees hand count increment only
CardPlayed       { playerId, cardId, targets }
MinionSummoned   { minionId, attack, health }
DamageDealt      { sourceId, targetId, amount }
MinionDied       { minionId }
TurnStarted      { playerId, timeLimit }
TurnEnded        { playerId }
GameEnded        { winnerId, reason }
```

### GameState shape (CCG.GameLogic)
```
players   [ { health, mana, hand, deckSize } ]
board     [ minionList per side ]
turn      { activePlayerId, number }
timer     { secondsRemaining }
phase     WaitingForPlayers | InProgress | Ended
```

### Reconnection
A disconnecting player has a 60-second window to reconnect using their original player token. On reconnect the Game Server sends a full state snapshot plus buffered events. If the window expires the game is forfeited to the opponent and the result is reported to the Platform API.

### Spectators
Spectators connect to the Game Server with a spectator token issued by the Platform API. They receive the same event stream scoped to public information only (opponent hand cards remain hidden).

---

## Platform API

**Runtime:** Node.js, TypeScript

Three internal modules deployed as a single service.

### Auth module
- `POST /auth/register` — create account, hash password with bcrypt, issue tokens
- `POST /auth/login` — validate credentials, issue JWT access token (short-lived) + refresh token
- `POST /auth/refresh` — exchange refresh token for new access token
- `POST /auth/logout` — revoke refresh token

### Collection module
- `GET  /cards` — full card catalog (server-defined, not player-generated)
- `GET  /collection` — player's owned cards and quantities
- `GET  /decks` — player's decks
- `POST /decks` — create deck (validated: 30 cards, max 2 copies non-legendary, max 1 legendary, all cards owned)
- `PUT  /decks/:id` — update deck (same validation)
- `DELETE /decks/:id` — delete deck

Deck validation runs on save and again at queue join to catch any collection changes since last save.

**Card data model:** Each card record stores static attributes in columns (`mana_cost`, `type`, `rarity`) and a `definition jsonb` field containing structured effect descriptors for standard mechanics (keywords, common effect patterns) and an optional `handler` key naming the `ICardHandler` implementation for cards with unique text effects. Cards without a `handler` use `DefaultCardHandler` in `CCG.GameLogic`. The exact `definition` JSON schema — effect descriptor format, keyword encoding, targeting rules — is resolved in the Collection subsystem spec.

### Matchmaking module
- `POST /queue/join` — enqueue player with selected deck; validates deck and checks player is not already in a session
- `POST /queue/leave` — dequeue player
- `GET  /queue/status` — polled by client every 2–3s; returns waiting | matched (with connection info)
- `POST /internal/session/result` — called by Game Server on game end; updates MMR and writes match record

**MMR system:** Elo, starting MMR 1000, K-factor 32. Queue matching starts at ±50 MMR, expands by 25 every 30 seconds, caps at ±300.

---

## Client

**Engine:** Unity or Godot (to be decided in the Client subsystem spec)

### Screen state machine
```
Login → Main Menu → [Collection] → Queue → Game Board → Post-Game → Main Menu
```
Any screen returns to Login on token expiry or logout.

### Connection architecture
| Connection | When active | Purpose |
|---|---|---|
| REST (Platform API) | Login through Queue | Auth, collection, deck management, queue polling |
| WebSocket (Game Server) | Game Board only | Send actions, receive event stream |

### Event handling
The client maintains a local `GameState` built entirely from server events. It never mutates state speculatively. Each event type maps to a UI update and animation. The client is a view over an event stream — game logic correctness is entirely the server's responsibility.

---

## Data Storage

### PostgreSQL (persistent)

**Auth**
- `players` — id (uuid), username, email, password_hash, created_at
- `refresh_tokens` — id, player_id, token_hash, expires_at, revoked

**Collection**
- `cards` — id, name, mana_cost, type (enum), rarity (enum), definition (jsonb: standard effect descriptors + optional `handler` key for unique-text cards)
- `player_cards` — (player_id, card_id) PK, quantity
- `decks` — id, player_id, name, card_list (jsonb: [{ card_id, quantity }])

**Matchmaking & History**
- `player_mmr` — player_id PK, mmr, wins, losses, updated_at
- `matches` — id, session_id (uuid, audit reference only — no FK, sessions are ephemeral in Redis), player1_id, player2_id, winner_id, p1_mmr_before, p2_mmr_before, p1_mmr_after, p2_mmr_after, ended_at

### Redis (ephemeral)

| Key pattern | Type | Contents | TTL |
|---|---|---|---|
| `queue:ranked` | Sorted set | player_id → MMR score | Managed manually |
| `queue:meta:{player_id}` | Hash | deck_id, queued_at | 10 minutes |
| `session:{session_id}` | Hash | ws_url, p1_id, p2_id, status | 4 hours |
| `player:session:{player_id}` | String | session_id | Matches session TTL |

---

## Open Questions (deferred to subsystem specs)

1. **Game engine choice** — Unity vs Godot. Decided in the Client spec.
2. **Card definition schema** — the handler pattern is decided (see Architecture section). The exact `jsonb` shape for standard effect descriptors, keyword encoding, and targeting rules is resolved in the Collection spec.
3. **Match notification mechanism** — polling vs SSE vs persistent WebSocket for queue status. Decided in the Matchmaking spec.
4. **Spectator token issuance** — how spectators obtain a token and discover live sessions. Decided in the Matchmaking or Game Server spec.
5. **Turn time limits** — duration per turn, overtime rules. Decided in the Game Server spec.
