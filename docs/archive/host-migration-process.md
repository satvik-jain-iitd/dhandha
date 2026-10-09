# Host Migration Process
### Product: Dhandha — Monopoly Deal (Indian Version)

> **Status:** Design / Pre-implementation — This feature is DEFERRED from the main timer sprint. Do not implement until the core timer mechanism is stable and tested.

---

## Why Host Migration Exists

In Dhandha's online multiplayer, the **Host** is the single source of truth for all game state. Hosts:
- Run the turn timer (clock authority)
- Process every `GAME_ACTION` from guests
- Broadcast `GAME_STATE` to all guests after every action
- Manage the Cloudflare Durable Object room

If the Host disconnects — even temporarily — **the game pauses for all guests**. They cannot receive state updates because the host is the only broadcaster.

Host migration = transferring the host role to another connected player so the game can continue without the original host.

---

## Why This Is Hard (Honest Assessment)

Before designing a solution, it's important to understand why this is non-trivial:

### 1. Single Source of Truth Problem
Guests receive `GAME_STATE` but they do not "own" it authoritatively. If the host disconnects mid-action (processed a `GAME_ACTION` but not yet broadcast the updated `GAME_STATE`), the new host's last known state is **1 action behind** the real state.

### 2. Split-Brain Risk
When all 4 guests simultaneously receive `PLAYER_LEFT` (host disconnected), they all know the host is gone. If all 4 try to elect themselves as the new host at the same moment, you get multiple hosts broadcasting conflicting states.

### 3. No Server-Side Game Logic
The Cloudflare Worker is a **pure relay** — it passes messages, has no game state, and runs no game logic. Host election requires either:
- The Worker to track who the host is and manage the transition, OR
- A client-side election protocol where guests agree on who becomes host

### 4. Reconnecting Original Host
When the original host comes back, they must:
- Discover the room still exists
- Not try to re-assert themselves as host
- Receive current state from the new host
- Participate as a regular guest
- This requires the Worker and all clients to recognize and handle this case

---

## Proposed Migration Flow

### Phase 1: Detect Host Disconnection

```
Host device disconnects
  │
  ├── Cloudflare Worker detects WebSocket closure (immediate)
  │     → broadcasts PLAYER_LEFT { playerId, wasHost: true } to all guests
  │
  └── All guests receive the message simultaneously
        → Each guest starts a SHORT random delay (50ms–300ms) before attempting election
          (prevents simultaneous election attempts)
```

### Phase 2: New Host Election (Deterministic)

To avoid split-brain, election must be **deterministic** — all players must agree on the same new host without negotiation.

**Algorithm:** New host = the guest with the **lowest player index** among currently connected players.

```
Players: [Aman(0), Priya(1), Rohan(2-HOST-disconnected), Neha(3), Satvik(4)]

Connected after Rohan leaves: Aman(0), Priya(1), Neha(3), Satvik(4)

New host = Aman (index 0, lowest connected index)
```

All guests independently compute this — no negotiation needed. Because it's purely deterministic (lowest index wins), there's no ambiguity.

**Aman's device then:**
1. Declares itself host by sending `HOST_CLAIM { newHostId: 'aman', lastKnownState: <GAME_STATE> }` to the Cloudflare Worker
2. Worker validates that Rohan is indeed disconnected and Aman is the lowest connected index
3. Worker stores `currentHostId = 'aman'`
4. Worker broadcasts `HOST_CHANGED { newHostId: 'aman' }` to all guests
5. Aman starts running the turn timer and broadcasting `GAME_STATE`

### Phase 3: State Reconciliation

The 1-action-behind problem must be addressed:

```
Last known GAME_STATE by guests:
  - Aman's device: state at t=47s (most recent broadcast)
  - Priya's device: state at t=46s (slightly older)
  - etc.

The new host (Aman) uses HIS last known state as the authoritative state.
He broadcasts it to all guests immediately after claiming host role.
All guests update to Aman's version.
```

> **Accepted trade-off:** If the original host processed a `GAME_ACTION` but hadn't broadcast the result yet, that action is **lost**. In practice: one card play might be rolled back. This is acceptable for a casual friend-group game — the player who played that card will notice and can replay it.

---

## Phase 4: Original Host Reconnects

When Rohan's device reconnects (same room code):

```
Rohan reconnects to room via same room code
  │
  ├── Cloudflare Worker checks: is Rohan still in the player list? YES
  │     → Does NOT re-grant host role
  │     → Sends REJOIN_ACK { currentHostId: 'aman', yourRole: 'guest' }
  │
  ├── Rohan's device receives REJOIN_ACK
  │     → Switches to guest mode (no longer runs timer or broadcasts)
  │
  └── New host (Aman) detects Rohan rejoined
        → Sends current GAME_STATE to Rohan's device
        → Rohan sees the board as-is and waits for his next turn
```

**Rohan's turns during absence:** Were auto-skipped by the turn timer (already handled by the main timer mechanism). His hand, properties, and bank are preserved exactly as they were when he disconnected.

---

## Cloudflare Worker Changes Required

The Worker currently just relays messages. For host migration, it needs:

| New responsibility | Why needed |
|-------------------|------------|
| Track `currentHostId` per room | Know which player is the host at any moment |
| Track which players are connected per room | Know who's available for election |
| Validate `HOST_CLAIM` messages | Prevent unauthorized host claims |
| Broadcast `HOST_CHANGED` to all clients | Inform everyone of the new authority |
| Handle `REJOIN_ACK` for returning original host | Ensure they rejoin as guest |

> **Note:** The Durable Object already maintains a room per game code (`getByName`). This makes adding per-room state (like `currentHostId`) relatively straightforward — it's the same DO instance that already exists.

---

## New Message Types

| Message | Direction | Payload | Meaning |
|---------|-----------|---------|---------|
| `HOST_CLAIM` | Client → Worker | `{ newHostId, lastKnownState }` | "I am the new host" |
| `HOST_CHANGED` | Worker → All clients | `{ newHostId }` | "The new host is X" |
| `REJOIN_ACK` | Worker → Rejoining client | `{ currentHostId, yourRole }` | "You're back, here's your role" |
| `GAME_STATE_SYNC` | New host → Rejoining guest | Full `GAME_STATE` | State catch-up on rejoin |

---

## Edge Cases to Handle

| Scenario | Resolution |
|----------|-----------|
| New host (Aman) also disconnects before claiming | Next lowest index (Priya) becomes host via same algorithm |
| Two guests simultaneously send `HOST_CLAIM` | Worker accepts the one with lower player index; rejects the other |
| Only 1 guest remains after host disconnects | That guest becomes host; but game may need to end if min player count not met |
| Original host reconnects but tries to resume as host (bug/race) | Worker rejects their `HOST_CLAIM` since `currentHostId` is already set to someone else |
| Original host reconnects after game ends | Room has closed; they see "Game already ended" screen |
| New host's device is very slow (broadcasts lag) | Other players experience delays — acceptable trade-off for this trust model |

---

## What This Replaces (From Main Doc Section 9)

In the current session-progress-saving-mechanism.md, the interim fallback for host disconnects is:
> Game pauses → guests see "Host disconnected — waiting..." → 2-minute reconnect window → if host doesn't return, game ends with current board scores.

**Once host migration is implemented, this interim fallback is replaced** by the Phase 1–4 flow above. The 2-minute-wait screen is no longer needed.

---

## Implementation Checklist (For When This Sprint Starts)

- [ ] Cloudflare Worker: add `currentHostId` and `connectedPlayers` to Durable Object state
- [ ] Worker: handle `HOST_CLAIM` message — validate, set new host, broadcast `HOST_CHANGED`
- [ ] Worker: on WebSocket close, check if it was the host; if yes, broadcast `PLAYER_LEFT { wasHost: true }`
- [ ] Worker: handle reconnect to existing room — send `REJOIN_ACK` with current host info
- [ ] Client (App.jsx): on receiving `HOST_CHANGED` — update local host reference, stop/start timer if applicable
- [ ] Client (App.jsx): if self is elected new host — switch to host mode, broadcast current state immediately
- [ ] Client (App.jsx): on receiving `REJOIN_ACK { yourRole: 'guest' }` — suppress host behaviors even if was previously host
- [ ] Client (App.jsx): new host sends `GAME_STATE_SYNC` to rejoining player on their reconnect
- [ ] Client (GameScreen.jsx): show "Host changed — [Aman] is now the host" toast to all players

---

## Dependencies

This feature depends on:
1. ✅ Core turn timer mechanism (from session-progress-saving-mechanism.md) being stable
2. ✅ Guest reconnect flow (same room code rejoins) working correctly
3. 🔴 Cloudflare Worker refactor to support per-room metadata beyond just message relay

---

## Estimated Complexity

| Area | Effort |
|------|--------|
| Cloudflare Worker changes | High — stateful logic in Durable Object |
| Client-side host election | Medium — deterministic algorithm, relatively clean |
| State reconciliation on migration | Medium — accepting 1-action-behind as trade-off |
| Reconnect original host as guest | Medium — new role-switching logic |
| **Total** | **~2 sprints** |

---

*Document created: 2026-06-22*
*Status: Design — DEFERRED. Implement AFTER core timer sprint is complete and stable.*
*Owner: Satvik Jain*
