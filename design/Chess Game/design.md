# Online Chess Platform - System Design Deep Dive

# Overview

This design represents a scalable real-time online chess platform similar to Chess.com or Lichess.

The architecture focuses on:

* Real-time multiplayer chess gameplay
* Low latency move synchronization
* Matchmaking
* Spectator mode
* Leaderboards and ratings
* Massive scale handling
* Event-driven persistence
* Eventually consistent analytics systems

The design intentionally separates:

* Real-time gameplay path
* Persistence path
* Analytics path
* Ranking path
* Historical replay path

This separation ensures gameplay remains extremely fast even when downstream systems are slow.

---

# Functional Requirements

The system supports:

1. Hundreds of thousands to millions of chess games
2. Different time controls

   * Bullet
   * Blitz
   * Rapid
   * Classical
3. Leaderboards and ranking
4. Matchmaking
5. Game replay and history
6. Spectator mode
7. Recording finished games
8. Elo rating updates
9. Multiple concurrent games
10. Real-time move synchronization

---

# Non-Functional Requirements

## Scalability

The design targets:

* Millions of DAU
* Hundreds of millions of games/day
* Extremely high concurrent websocket connections

## Low Latency

Chess gameplay requires:

* Very low move propagation latency
* Fast clock synchronization
* Real-time game state consistency

Target latency:

* Move propagation: ~50ms-150ms
* Clock drift: minimal

## High Availability

Even if:

* analytics fails
* leaderboard service fails
* persistence pipeline slows down

The active games should continue.

## Fault Isolation

The architecture isolates:

* gameplay systems
* analytics systems
* persistence systems
* leaderboard systems

---

# High-Level Architecture

The design contains these major components:

1. Web Client
2. Mobile Client
3. API Gateway / Load Balancer
4. Game Servers
5. Record Server
6. Redis Streams
7. Leaderboard Service
8. Redis Leaderboard Cache
9. MongoDB / Game DB
10. User DB
11. CDC Queue Processing

The architecture follows:

* Event-driven design
* CQRS-style separation
* Write-optimized gameplay systems
* Read-optimized leaderboard systems

---

# Core Entities

# User

```json
{
  "id": "uuid",
  "username": "player123",
  "rating": 1800,
  "country": "IN",
  "created_at": "timestamp"
}
```

---

# Game

```json
{
  "id": "game_uuid",
  "white_user_id": "u1",
  "black_user_id": "u2",
  "time_control": "5+0",
  "status": "ACTIVE",
  "winner": null,
  "created_at": "timestamp",
  "finished_at": null
}
```

---

# Move

```json
{
  "game_id": "g1",
  "move_number": 12,
  "fen": "board_state",
  "move": "Nf3",
  "player": "white",
  "timestamp": "ts"
}
```

---

# Why FEN is Important

FEN (Forsyth–Edwards Notation) stores:

* entire board state
* active player
* castling rights
* en passant
* move counters

This allows:

* game recovery
* replay
* spectators joining mid-game
* crash recovery

---

# Client Layer

# Web Client

The browser application:

Responsibilities:

* Render chessboard
* Maintain websocket connection
* Send player moves
* Receive opponent moves
* Show timers
* Spectator support
* Handle reconnects

Technologies:

* React
* WebSocket
* Local move prediction

---

# Mobile Client

Same responsibilities as web client but optimized for:

* unstable networks
* battery optimization
* reconnection handling

---

# API Gateway / Load Balancer

The API Gateway is the entry point.

Responsibilities:

* Authentication
* Rate limiting
* Routing requests
* Websocket upgrade handling
* Sticky session routing
* TLS termination

---

# Why Sticky Sessions?

A chess game is stateful.

Both players should ideally connect to the same game server.

Benefits:

* lower synchronization overhead
* no cross-server communication per move
* easier clock synchronization

Possible strategies:

1. Hash(game_id)
2. Consistent hashing
3. Redis-backed routing registry

---

# Game Server

This is the most important component.

Responsibilities:

* Maintain active games in memory
* Validate moves
* Enforce chess rules
* Maintain clocks
* Broadcast moves
* Handle resignations
* Handle draw offers
* Detect checkmate/stalemate
* Push events to streams

---

# Why Keep Active Games in Memory?

Because database writes are too slow for every move.

In-memory state gives:

* microsecond reads
* low-latency updates
* fast broadcasts

A typical game server maintains:

```text
Map<game_id, GameState>
```

---

# Game State Structure

```json
{
  "game_id": "g1",
  "fen": "current_board",
  "moves": [],
  "white_clock": 120,
  "black_clock": 95,
  "last_move_ts": "timestamp",
  "status": "ACTIVE"
}
```

---

# Move Flow

# Step 1

Client sends move:

```json
{
  "game_id": "g1",
  "move": "Nf3"
}
```

# Step 2

Game server validates:

* legal move
* correct player turn
* clock not expired
* game still active

# Step 3

Game state updated in memory.

# Step 4

Move broadcasted to:

* opponent
* spectators

# Step 5

Event written to Redis Streams.

---

# Why Redis Streams?

Redis Streams act as:

* durable event log
* async communication bus
* replayable stream

Benefits:

* decouples gameplay from persistence
* enables retries
* multiple consumers
* replay support

---

# Redis Streams Topics

Possible topics:

```text
moves_stream
finished_games_stream
leaderboard_events
analytics_events
```

---

# Record Server

The Record Server consumes game events.

Responsibilities:

* Persist completed games
* Store move history
* Store replay data
* Archive games

Important:

Gameplay servers should NOT directly write to DB synchronously.

Reason:

Database latency would impact move latency.

---

# Why Asynchronous Persistence?

If every move required:

```text
move -> DB write -> ACK
```

then gameplay latency becomes database latency.

Instead:

```text
move -> in-memory update -> publish event
```

Persistence becomes eventual.

---

# MongoDB / Game DB

The design suggests MongoDB for game storage.

Why MongoDB?

Benefits:

* flexible schema
* append-heavy workloads
* easy move array storage
* horizontal scaling
* massive write throughput

A game document may look like:

```json
{
  "_id": "game123",
  "white_user_id": "u1",
  "black_user_id": "u2",
  "moves": [
    {"move": "e4", "fen": "..."},
    {"move": "e5", "fen": "..."}
  ],
  "winner": "u1",
  "time_control": "3+0",
  "created_at": "ts",
  "finished_at": "ts"
}
```

---

# MongoDB Collection Design

# games Collection

Indexes:

```javascript
{ _id: 1 }
{ white_user_id: 1, created_at: -1 }
{ black_user_id: 1, created_at: -1 }
{ finished_at: -1 }
{ status: 1 }
```

Why?

* fetch user history
* recent games queries
* replay browsing
* active game filtering

---

# Move Storage Strategies

## Strategy 1 - Embedded Moves

```json
{
  "moves": []
}
```

Pros:

* easy replay
* single read

Cons:

* document grows large

---

## Strategy 2 - Separate Move Collection

```json
{
  "game_id": "g1",
  "move_number": 10,
  "move": "Nf3"
}
```

Indexes:

```javascript
{ game_id: 1, move_number: 1 }
```

Better for:

* very large games
* analysis engines
* move streaming

---

# Leaderboard Service

This service manages:

* Elo ratings
* ranking queries
* top players

Responsibilities:

* consume finished games
* calculate Elo
* update rankings
* publish leaderboard changes

---

# Why Separate Leaderboard Service?

Leaderboard calculations should not block gameplay.

This separation allows:

* independent scaling
* isolated failures
* async updates

---

# Redis Sorted Sets for Rankings

The design mentions Redis cache.

Redis Sorted Sets are ideal.

Example:

```text
ZADD leaderboard 1850 user123
```

Queries:

```text
ZREVRANGE leaderboard 0 99
```

Benefits:

* O(logN) insert
* instant top-k queries
* real-time leaderboards

---

# Leaderboard Database Design

# Redis Structure

```text
leaderboard:global
leaderboard:country:IN
leaderboard:blitz
leaderboard:bullet
```

---

# User DB

Stores:

* profiles
* auth info
* ratings
* preferences
* settings

Suggested DB:

* PostgreSQL
* MySQL

Why relational DB?

Because user/account data needs:

* transactions
* consistency
* constraints

---

# User Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  username VARCHAR(32) UNIQUE,
  email VARCHAR(255) UNIQUE,
  rating INT,
  created_at TIMESTAMP
);
```

Indexes:

```sql
CREATE UNIQUE INDEX idx_username ON users(username);
CREATE INDEX idx_rating ON users(rating DESC);
```

---

# CDC Queue Processing

CDC = Change Data Capture.

Purpose:

* observe DB changes
* trigger downstream updates

Responsibilities:

* update Elo history
* analytics aggregation
* notifications
* ranking recalculation

Possible technologies:

* Debezium
* Kafka Connect
* DynamoDB Streams

---

# Clock Synchronization Design

This is a difficult problem.

The diagram discusses several strategies.

---

# Strategy 1 - Central Server Time

Server owns authoritative clock.

Pros:

* consistent
* cheat resistant

Cons:

* constant updates required

---

# Strategy 2 - Client Local Clock

Client estimates timer locally.

Pros:

* smooth UI
* less server traffic

Cons:

* possible drift

Most systems use hybrid approaches.

---

# Recommended Hybrid Clock Design

1. Server maintains authoritative time
2. Client predicts locally
3. Periodic clock correction packets
4. Reconciliation during reconnects

---

# Spectator Mode

The design mentions observing games.

Implementation:

* spectators subscribe to game channels
* receive move broadcasts
* no write permissions

Optimization:

Use fanout channels.

---

# Spectator Scaling Problem

Large games may have:

* 100K+ spectators

Solution:

* pub/sub fanout
* edge websocket gateways
* CDN-assisted streaming

---

# Matchmaking Design

Currently marked partially out of scope.

Possible implementation:

---

# Matchmaking Queue

Players enter queue:

```json
{
  "user_id": "u1",
  "rating": 1700,
  "time_control": "3+0"
}
```

---

# Matchmaking Algorithm

## Step 1

Find nearby ratings:

```text
±50 Elo
```

## Step 2

Expand range gradually:

```text
±100
±200
```

## Step 3

Create game session.

---

# Matchmaking Data Structure

Redis Sorted Sets:

```text
ZADD matchmaking:3+0 1700 user123
```

Search:

```text
ZRANGEBYSCORE
```

---

# Fault Tolerance

# Game Server Crash Recovery

Because events are streamed:

* replay from Redis Streams
* restore game state
* reconnect clients

---

# Persistent Snapshots

Optionally:

Every N moves:

```text
save snapshot
```

This reduces replay recovery time.

---

# Reconnection Handling

If client disconnects:

1. reconnect websocket
2. fetch latest game state
3. reconcile missing moves
4. continue game

---

# Consistency Model

The architecture intentionally uses:

# Strong Consistency

For:

* active game state
* clock handling
* move ordering

# Eventual Consistency

For:

* leaderboards
* analytics
* archives
* recommendations

This balance improves scalability.

---

# Scaling the Game Servers

Game servers are horizontally scalable.

Possible partitioning:

```text
hash(game_id) % N
```

Benefits:

* deterministic routing
* balanced distribution

---

# Hot Partition Problem

Large tournaments may overload one server.

Solutions:

* dynamic shard migration
* virtual nodes
* consistent hashing
* regional balancing

---

# Regional Deployment

Multi-region architecture:

Regions:

* US
* EU
* India
* Asia

Players routed to nearest region.

Benefits:

* lower latency
* better gameplay experience

---

# Cross-Region Gameplay

Problem:

Players from different regions.

Possible solutions:

1. choose midpoint region
2. choose lower latency player
3. global routing layer

---

# Anti-Cheat Extensions (Out of Scope)

Possible future system.

---

# Cheat Detection Pipeline

Inputs:

* move timings
* engine similarity
* suspicious accuracy
* tab switching
* behavioral patterns

---

# ML-Based Detection

Potential models:

* anomaly detection
* sequence models
* reinforcement analysis

---

# Analysis Engine

Could integrate:

* Stockfish clusters
* GPU analysis workers

---

# Tournament System (Out of Scope)

Potential implementation.

Features:

* Swiss tournaments
* Round robin
* knockout

Additional entities:

```json
{
  "tournament_id": "t1",
  "players": [],
  "rounds": []
}
```

---

# Chat System (Out of Scope)

Architecture:

* websocket rooms
* moderation pipeline
* profanity filtering

Storage:

* Cassandra
* DynamoDB

---

# Notifications System (Out of Scope)

Events:

* game invite
* challenge accepted
* tournament start
* friend online

Possible implementation:

* Kafka
* notification workers
* APNS/FCM

---

# Analytics Pipeline

Possible metrics:

* DAU
* game completion rate
* average session time
* matchmaking latency

Architecture:

```text
Redis Streams -> Kafka -> Spark/Flink -> Data Warehouse
```

---

# API Design

# POST /move

Request:

```json
{
  "game_id": "g1",
  "move": "e4"
}
```

Response:

```json
{
  "success": true
}
```

---

# GET /game/{id}

Returns:

* current state
* move history
* timers

---

# GET /leaderboard

Returns:

* rankings
* ratings
* top players

---

# Security Considerations

# Prevent Move Tampering

Use:

* authenticated websockets
* signed sessions
* server-side validation

---

# Rate Limiting

Prevent:

* spam requests
* websocket abuse
* bot attacks

---

# DDoS Protection

Use:

* CDN
* WAF
* edge rate limiting

---

# Observability

Critical metrics:

* move latency
* websocket disconnects
* matchmaking wait time
* server CPU
* Redis lag
* stream consumer lag

---

# Logging

Structured logs:

```json
{
  "game_id": "g1",
  "move": "Nf3",
  "latency_ms": 12
}
```

---

# Technologies Summary

| Component       | Suggested Tech        |
| --------------- | --------------------- |
| API Gateway     | NGINX / Envoy         |
| Game Servers    | Go / Java / Rust      |
| Websocket Layer | Socket.IO / native ws |
| Streams         | Redis Streams / Kafka |
| Game DB         | MongoDB               |
| User DB         | PostgreSQL            |
| Cache           | Redis                 |
| Matchmaking     | Redis Sorted Sets     |
| Monitoring      | Prometheus + Grafana  |
| Logging         | ELK Stack             |

---

# End-to-End Flow

# Game Creation

1. User enters matchmaking
2. Match found
3. Game server assigned
4. Clients connected
5. Game state initialized

---

# During Gameplay

1. Player sends move
2. Game server validates
3. State updated in memory
4. Opponent notified
5. Spectators notified
6. Event streamed asynchronously

---

# Game Completion

1. Winner determined
2. Finished game event emitted
3. Record server persists game
4. Leaderboard updates Elo
5. Analytics pipelines triggered

---

# Why This Architecture Works Well

This design succeeds because it:

* keeps gameplay path extremely lightweight
* separates hot and cold paths
* uses asynchronous persistence
* isolates failures
* supports massive scale
* enables replayability
* supports future extensibility

---

# Final Architectural Insights

This is a strong industry-style architecture because:

1. Real-time systems are isolated from slow systems
2. Event streaming decouples components
3. Redis enables low-latency operations
4. MongoDB handles massive append-heavy storage
5. Leaderboards are optimized separately
6. Failure domains are isolated
7. Scaling can happen independently per service

This is very similar to architectures used in:

* Chess.com
* Lichess
* Real-time gaming systems
* Multiplayer collaborative systems
* Live trading dashboards
