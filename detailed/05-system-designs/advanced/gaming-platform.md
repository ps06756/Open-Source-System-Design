# Design a Gaming Platform

## 1. Requirements

### Functional Requirements
- Real-time multiplayer games
- Matchmaking
- Leaderboards
- Player profiles and progression
- In-game chat
- Game state persistence

### Non-Functional Requirements
- Low latency (< 50ms for real-time games)
- High availability
- Fairness (anti-cheat)
- Global reach
- Support millions of concurrent players

### Scale Estimates
- 100M registered users
- 10M concurrent players peak
- 1M active game sessions
- 100K messages per second

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Game Clients                        │  │
│  │            (PC, Console, Mobile)                     │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  API Gateway                          │  │
│  │        (Auth, Rate Limiting, Routing)                │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│      ┌────────────────────┼────────────────────┐            │
│      ▼                    ▼                    ▼            │
│ ┌──────────┐       ┌──────────────┐     ┌──────────────┐   │
│ │Matchmaking│      │ Game Servers │     │  Platform    │   │
│ │ Service   │      │  (Realtime)  │     │  Services    │   │
│ └──────────┘       └──────────────┘     │ (Profile,    │   │
│                                          │  Leaderboard)│   │
│                                          └──────────────┘   │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Game State Storage                       │  │
│  │           (Redis + Persistent DB)                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Real-Time Game Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Real-Time Game Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client-Server Model (Authoritative Server):               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌────────────────────────────────────────────────┐   │ │
│  │  │              Game Server                        │   │ │
│  │  │         (Source of Truth)                       │   │ │
│  │  │                                                  │   │ │
│  │  │  • Runs game simulation                         │   │ │
│  │  │  • Validates player actions                     │   │ │
│  │  │  • Broadcasts state to clients                  │   │ │
│  │  │  • Tick rate: 20-60 updates/second             │   │ │
│  │  └────────────────────────────────────────────────┘   │ │
│  │         ▲           ▲           ▲                      │ │
│  │         │           │           │                      │ │
│  │    ┌────┴────┐ ┌────┴────┐ ┌────┴────┐                │ │
│  │    │ Client 1│ │ Client 2│ │ Client 3│                │ │
│  │    │ (Input) │ │ (Input) │ │ (Input) │                │ │
│  │    └─────────┘ └─────────┘ └─────────┘                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Game Loop on Server:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  while game_running:                                   │ │
│  │      # 1. Process inputs from all clients             │ │
│  │      for client in clients:                           │ │
│  │          process_input(client.pending_inputs)         │ │
│  │                                                         │ │
│  │      # 2. Update game state                           │ │
│  │      update_physics()                                  │ │
│  │      update_game_logic()                               │ │
│  │                                                         │ │
│  │      # 3. Send state to clients                       │ │
│  │      for client in clients:                           │ │
│  │          send_state(client, game_state)               │ │
│  │                                                         │ │
│  │      # 4. Wait for next tick                          │ │
│  │      sleep(tick_interval)  # 16ms for 60 tick        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Latency Compensation Techniques:                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Client-Side Prediction                             │ │
│  │     Client simulates locally, reconciles with server  │ │
│  │                                                         │ │
│  │  2. Lag Compensation                                   │ │
│  │     Server rewinds state to when player shot          │ │
│  │                                                         │ │
│  │  3. Interpolation                                      │ │
│  │     Smooth movement between server updates            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Matchmaking

```
┌─────────────────────────────────────────────────────────────┐
│                      Matchmaking                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Goal: Create fair and fun matches quickly                 │
│                                                              │
│  Matchmaking Factors:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Skill Rating (MMR/Elo)                             │ │
│  │     Similar skill = competitive game                   │ │
│  │                                                         │ │
│  │  2. Latency                                             │ │
│  │     Group players near same server                     │ │
│  │                                                         │ │
│  │  3. Wait Time                                           │ │
│  │     Expand criteria as wait increases                  │ │
│  │                                                         │ │
│  │  4. Party Size                                          │ │
│  │     5-stack vs 5-stack, not vs 5 solos                │ │
│  │                                                         │ │
│  │  5. Game Mode                                           │ │
│  │     Ranked vs Casual                                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Matchmaking Algorithm:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Player enters queue                                │ │
│  │     { player_id, mmr: 1500, region: NA, mode: ranked }│ │
│  │                                                         │ │
│  │  2. Find compatible players                            │ │
│  │     MMR range: 1500 ± 100 (expands over time)         │ │
│  │     Same region, same mode                            │ │
│  │                                                         │ │
│  │  3. Form teams with balanced MMR                       │ │
│  │     Team 1 avg MMR ≈ Team 2 avg MMR                   │ │
│  │                                                         │ │
│  │  4. Assign to game server                              │ │
│  │     Select server with lowest latency to all players  │ │
│  │                                                         │ │
│  │  5. Notify all players, start game                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Queue Data Structure:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Redis Sorted Sets:                                    │ │
│  │  Key: queue:{region}:{mode}                           │ │
│  │  Score: MMR                                            │ │
│  │  Member: player_id                                     │ │
│  │                                                         │ │
│  │  ZRANGEBYSCORE queue:NA:ranked 1400 1600 LIMIT 0 10   │ │
│  │  → Find 10 players with MMR 1400-1600                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Leaderboards

```
┌─────────────────────────────────────────────────────────────┐
│                     Leaderboards                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirements:                                               │
│  • Get top N players                                        │
│  • Get player's rank                                        │
│  • Update scores in real-time                              │
│  • Handle millions of players                              │
│                                                              │
│  Redis Sorted Set Implementation:                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Key: leaderboard:{game}:{season}                      │ │
│  │  Score: Player's score/MMR                             │ │
│  │  Member: player_id                                     │ │
│  │                                                         │ │
│  │  Operations:                                            │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  # Update score                                   │ │ │
│  │  │  ZADD leaderboard:fortnite:s1 2500 player123     │ │ │
│  │  │                                                    │ │ │
│  │  │  # Get top 100                                    │ │ │
│  │  │  ZREVRANGE leaderboard:fortnite:s1 0 99 WITHSCORES│ │
│  │  │                                                    │ │ │
│  │  │  # Get player rank                                │ │ │
│  │  │  ZREVRANK leaderboard:fortnite:s1 player123      │ │ │
│  │  │  → 54321 (rank 54,322)                           │ │ │
│  │  │                                                    │ │ │
│  │  │  # Get players around me                         │ │ │
│  │  │  ZREVRANGE ... rank-5 rank+5                     │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  Complexity: O(log N) for all operations              │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  For very large leaderboards:                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Approximate ranking:                                  │ │
│  │  • Bucket players by score range                      │ │
│  │  • Exact rank within bucket                           │ │
│  │  • "You're rank ~50,000" instead of exact            │ │
│  │                                                         │ │
│  │  Sharding:                                              │ │
│  │  • Regional leaderboards                              │ │
│  │  • Periodic aggregation for global                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Game Server Management

```
┌─────────────────────────────────────────────────────────────┐
│              Game Server Management                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Server Fleet Management:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │              Fleet Manager                       │  │ │
│  │  │  • Track available servers                       │  │ │
│  │  │  • Auto-scale based on demand                   │  │ │
│  │  │  • Health monitoring                            │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                    │                                   │ │
│  │      ┌─────────────┼─────────────┐                    │ │
│  │      ▼             ▼             ▼                    │ │
│  │  ┌────────┐   ┌────────┐   ┌────────┐                │ │
│  │  │Server 1│   │Server 2│   │Server 3│                │ │
│  │  │ IDLE   │   │ 5/10   │   │ 10/10  │                │ │
│  │  └────────┘   └────────┘   └────────┘                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Session Lifecycle:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Matchmaker finds players                          │ │
│  │  2. Request server from Fleet Manager                 │ │
│  │  3. Allocate server, configure game settings          │ │
│  │  4. Send connection info to players                   │ │
│  │  5. Players connect, game starts                      │ │
│  │  6. Game ends, save results                           │ │
│  │  7. Return server to pool                             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Container-based approach (Kubernetes):                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  • Each game session = new pod                        │ │
│  │  • Spin up on demand, terminate after game            │ │
│  │  • Services like Agones (Google) for game servers     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Anti-Cheat

```
┌─────────────────────────────────────────────────────────────┐
│                      Anti-Cheat                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Types of Cheats:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  • Wallhacks: See through walls                        │ │
│  │  • Aimbots: Auto-aim assistance                        │ │
│  │  • Speed hacks: Move faster than allowed               │ │
│  │  • Exploits: Abuse bugs                                │ │
│  │  • DDOS: Lag opponents                                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Prevention Layers:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Client-side detection                              │ │
│  │     • Monitor for known cheat signatures              │ │
│  │     • Detect memory modifications                     │ │
│  │     • Example: EasyAntiCheat, BattlEye                │ │
│  │                                                         │ │
│  │  2. Server-side validation (most important)           │ │
│  │     • Never trust client                              │ │
│  │     • Validate all actions server-side               │ │
│  │     • Speed > max speed? Reject                      │ │
│  │     • Shooting through wall? Reject                  │ │
│  │                                                         │ │
│  │  3. Statistical analysis                               │ │
│  │     • Accuracy too high over time                     │ │
│  │     • Inhuman reaction times                          │ │
│  │     • Flag for review                                 │ │
│  │                                                         │ │
│  │  4. Player reporting + review                         │ │
│  │     • Let players report                              │ │
│  │     • Review replays                                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Real-time** | UDP + custom protocol | Low latency |
| **Game Servers** | Authoritative server | Cheat prevention |
| **Matchmaking** | Redis sorted sets | Fast skill-based matching |
| **Leaderboards** | Redis sorted sets | O(log N) rank queries |
| **Server Management** | Kubernetes/Agones | Auto-scaling |

---

[Back to System Designs](../README.md)
