# PM Case Study: Session & Progress Saving Mechanism
### Product: Dhandha — Monopoly Deal (Indian Version)

---

## 1. Product Overview

### What Is Dhandha?

**Dhandha** is a free-to-play Progressive Web App (PWA) built on React + Vite that digitizes **Monopoly Deal** — a fast-paced card game — with an Indian twist. All property cards use Indian cities (Mumbai, Delhi, Bengaluru, etc.) and currencies are in Crores (₹Cr). The app is installable on mobile and desktop via "Add to Home Screen" and works offline.

**GitHub repo:** `satvik-jain-iitd/Monopoly-Deal-card-game-indian-version`
**App name in UI:** Dhandha
**App URL:** Deployed on Netlify / GitHub Pages (static)

---

### What Is Monopoly Deal? (Game Rules Summary)

Monopoly Deal is a card game for 2–6 players. The goal is to be the **first player to collect 3 complete property sets**.

**Card Types:**
| Card | What it does |
|------|-------------|
| **Property Cards** | Placed in your property area. Collect full color sets to win. |
| **Money Cards** | Placed in your bank. Used to pay rent/debts. |
| **Action Cards** | Special moves: steal properties, collect rent, protect yourself, etc. |
| **Rent Cards** | Charge other players for your properties. |
| **Wild Property Cards** | Can be assigned to any color. |

**Key Action Cards in This Game:**
| Card | Effect |
|------|--------|
| Deal Breaker | Steal an opponent's **entire complete set** |
| Sly Deal | Steal a **single property** from an incomplete set |
| Forced Deal | Swap one of your properties for one of theirs |
| Debt Collector | Demand ₹5Cr from any one player |
| Just Say No (Nahi!) | Block any action card against you |
| Pass Go | Draw 2 extra cards |
| Birthday | Everyone pays you ₹2Cr |
| Double Rent | Double the next rent you charge |
| House / Hotel (Ghar/Hotel) | Add to complete sets to increase rent |
| Insurance *(custom)* | Blocks a Deal Breaker only |
| Sabotage *(custom)* | Swap two opponents' properties with each other |

**Win Condition:** First player to complete **3 full property sets** (any color) wins.

**Turn Structure:**
1. **Draw** 2 cards (or 5 if hand is empty)
2. **Play** up to 3 cards (as property, bank, or action)
3. **End turn** (discard down to 7 if over limit)

**Property Sets (examples):**
| Color | Cards Needed | Full Set Rent |
|-------|-------------|--------------|
| Brown | 2 | ₹2Cr |
| Dark Blue | 2 | ₹8Cr |
| Green | 3 | ₹7Cr |
| Railroad | 4 | ₹4Cr |

**Scoring (Series Mode):**
After each game, players earn series points based on finish position:
- Formula: `n+1` for 1st place, `n+1-position` for each subsequent rank (where n = number of players)
- Example in a 5-player game: 1st=6pts, 2nd=4pts, 3rd=3pts, 4th=2pts, 5th=1pt
- Tiebreakers (in order): completed sets → property cards on table → bank cash → cards in hand

---

### Play Modes Available Today

| Mode | Description | How it works |
|------|-------------|-------------|
| **Pass & Play** | Offline, same device | Players physically pass the phone between turns. A "hand cover" screen prevents peeking. |
| **Local Multiplayer (LAN)** | Same WiFi, separate devices | One player acts as host on local server. Others connect via IP. |
| **Online Multiplayer (Cloud)** | Different networks | Uses Cloudflare Durable Objects as a WebSocket relay. One room per game code. |
| **Offline P2P (WebRTC)** | No internet, QR-based | Host generates a QR code. Guest scans it. Peer-to-peer connection via WebRTC. No server needed. |

---

### Target Users

- **Primary:** Friend groups and families in India playing casual card games
- **Platform:** Mobile-first (iOS/Android PWA via "Add to Home Screen") + Desktop browser
- **Typical session:** 3–6 players, 15–45 min per game, often multiple games in a session (series mode)
- **Context:** Often played in person — e.g., friends at a hostel/office, family gathering — where each person has their own phone but they're sitting together

---

### Tech Stack (Brief)

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + Vite 8 |
| Styling | MUI 9 + Emotion |
| State management | `useReducer` (immutable, no Redux) |
| PWA | vite-plugin-pwa + service worker |
| Multiplayer transport | WebSocket (Cloudflare Worker) + WebRTC (offline P2P) |
| Session storage | Browser `localStorage` only (no backend DB) |
| Hosting | GitHub Pages / Netlify (static) + Cloudflare Worker (relay) |

No user authentication. No backend database. All game state is either in-memory or browser localStorage.

---

## 2. Current State of Session Saving (As-Is)

### 2a. Offline / Pass-and-Play / Local Multiplayer

- Game state is **no longer auto-saved**. Auto-persistence was removed on 2026-06-22.
- On navigation (refresh/close), progress is lost.
- Users can manually save their game by navigating back to Home screen (which triggers `saveSession()`).
- Saved sessions appear on the Home screen under "Resume Saved Game" — user selects which to continue.
- Up to 5 sessions are kept; oldest auto-removed when limit exceeded.
- On app open (refresh / PWA restart), **no session is auto-restored**. User must pick from the list.

### 2b. Online Multiplayer (Cloud or WebRTC)

- Game state lives **only in memory** — it is NOT saved to localStorage.
- The **Host** holds the authoritative game state and broadcasts it to all Guests after every action via WebSocket.
- If a player refreshes or closes the tab:
  - Their WebSocket connection drops.
  - The Cloudflare Worker broadcasts a `PLAYER_LEFT` message to all remaining connected players.
  - **Currently: the game breaks.** There is no mechanism to continue or recover.

### 2c. Series / Standings

- Cumulative series standings persist in `localStorage` (key: `dhandha.series.v1`).
- The series is tied to a **specific group of player names**. If a new name combination starts a game, a fresh series begins.
- Standings track: total points, games played, wins, finish positions per player.

---

## 3. The Problem

### Scenario

Five friends — Aman, Priya, Rohan, Neha, Satvik — are playing an online multiplayer game of Dhandha. The game is 20 minutes in. Rohan gets an urgent call and has to leave.

**What happens today:**
- Rohan closes the app or presses Home.
- The WebSocket drops. All other 4 players see *"Koi player disconnect ho gaya!"*
- The game is **stuck or broken**. The remaining 4 players cannot continue.
- Even if Rohan comes back 10 minutes later, there is **no way to rejoin** the game mid-session.

### Why This Matters

1. **Kills the entire session** — one person leaving forces everyone to abandon the game.
2. **Real-world frequency** — In casual social gaming, interruptions (calls, bathroom breaks, etc.) are common.
3. **No recovery path** — Neither the disconnected player nor the host has any recourse.
4. **Series integrity** — If the game breaks mid-way, series points are not recorded for anyone.

---

## 4. Proposed Solution

### Core Insight: Not All Disconnects Are Equal

We need to distinguish between two fundamentally different types of player absence:

| Type | Trigger | Intent | What game should do |
|------|---------|--------|-------------------|
| **Soft Disconnect** | Network drop, phone locked, app backgrounded, accidental close | Unintentional — player wants to come back | Preserve their state; auto-skip their turn |
| **Intentional Quit** | Player manually clicks Home → confirms exit dialog | Deliberate — player is leaving the game | Dissolve their cards into the deck; apply penalty |

---

### Case 1: Soft Disconnect (Auto / Network / Accidental)

**What triggers it:**
- WebSocket drops without a deliberate `PLAYER_QUIT` message being sent
- Network timeout, phone going to sleep, accidental browser close

**What the game should do:**
1. Mark the player as `disconnected` (not removed) in game state.
2. Show a **"[Name] — Offline"** badge on their board to other players.
3. When it is the disconnected player's turn, **auto-skip** it silently.
4. Their **hand, properties, bank remain exactly as they were** — nothing is touched.
5. If the player **reconnects** using the same room code within the session, they are restored to the game from the current state and resume normally.
6. **No series penalty** applied.

**Edge case — host disconnects:**
- The Host holds the authoritative game state. If the host soft-disconnects, the game pauses for Guests (they receive no state updates).
- **Open question:** Should host role be migrated to another player automatically?

---

### Case 2: Intentional Quit (Home Button → Confirm Dialog)

**What triggers it:**
- Player manually clicks the **Home (🏠) button** inside the game screen.
- Confirmation dialog appears: *"Game chhod rahe ho? Tumhare cards deck mein chale jayenge aur series mein -5 points milenge."*
- Player confirms by tapping **"Game Chhodo"**.

**What the game should do — step by step:**
1. Collect all of the quitting player's cards:
   - Cards in **hand**
   - Cards in **properties** area (all colors, including wild cards and buildings/Ghar/Hotel)
   - Cards in **bank** (money + action cards played as money)
2. Return all collected cards to the **draw pile (deck)**.
3. **Shuffle** the deck.
4. **Remove the player** from the player list — they are gone permanently from this game.
5. Adjust `currentPlayerIndex` if needed (if it was their turn or a later player, shift index accordingly).
6. Record a **-5 series points penalty** immediately in `localStorage` series standings (even though the game is not over yet).
7. Broadcast updated game state to remaining players.
8. Show a **toast to remaining players:** *"[Name] ne game chhod diya. Unke cards deck mein gaye. (-5 series points)"*

---

### Anti Rage-Quit Mechanism

The **-5 series points penalty** is the key enforcement mechanism:

- Series points accumulate over an entire session (multiple games). A typical winner earns 6 points per game. Losing 5 points for quitting early is a significant deterrent.
- The penalty is recorded **immediately** on quit — not at game end — so it cannot be avoided.
- This discourages players from quitting when they're losing just to avoid a bad finish position.

---

### What Happens to Pending Actions at Quit?

If the quitting player is currently **owed money** or **owes money** to someone (e.g., mid-rent collection):

| Situation | Proposed Handling |
|-----------|-----------------|
| Quitting player owes rent | Debt is **forgiven** — other players don't receive the payment |
| Quitting player is owed rent | Remaining payers still pay their share to the quitting player's bank cards, which then go into the deck |

> **Open question:** Should the quitting player's debt be auto-paid from their cards before dissolution? This would be fairer but complex to implement.

---

## 5. Scoring Impact Analysis

### Current Scoring Formula (5-player game)
| Position | Points |
|----------|--------|
| 1st | 6 |
| 2nd | 4 |
| 3rd | 3 |
| 4th | 2 |
| 5th | 1 |

### After Proposed Change
- Quitting player receives: `(their series accumulated points) - 5`
- Remaining 4 players finish and receive normal position-based points from a **4-player scoring table** for that game:

| Position | Points (4-player table) |
|----------|------------------------|
| 1st | 5 |
| 2nd | 3 |
| 3rd | 2 |
| 4th | 1 |

> **Note:** The quitting player's cards re-entering the deck may **change who wins** the game — their property cards could be drawn and used by someone else to complete a set. This is intentional and makes quitting feel consequential.

---

## 6. Open Questions (Must Decide Before Implementation)

| # | Question | Options |
|---|----------|---------|
| 1 | **Auto-skip timeout:** When a soft-disconnected player's turn comes, how long do we wait before auto-skipping? | (a) Skip immediately, (b) 30-second timer, (c) Other players vote to skip |
| 2 | **Rejoin flow:** How does a soft-disconnected player reconnect? | (a) Same room code, no approval needed, (b) Host must approve rejoin |
| 3 | **Host disconnect (soft):** If the host soft-disconnects, should host role migrate to another player? | (a) Yes — auto-migrate to next player in list, (b) No — game pauses until host reconnects |
| 4 | **Minimum player count:** If players keep quitting until only 1 remains, what happens? | (a) Auto-end game, declare last remaining player as winner, (b) Keep going (1-player game is pointless, so auto-end makes more sense) |
| 5 | **Buildings (Ghar/Hotel) on quitting player's properties:** Do building cards also return to deck on quit? | (a) Yes — all cards go back, (b) No — buildings are discarded separately (they have ₹3/₹4 value) |
| 6 | **Pending payment at quit time:** What if the quitting player owes an unpaid rent/debt? | (a) Debt forgiven, (b) Auto-pay from their cards before dissolution |
| 7 | **-5 penalty edge case:** What if a player has fewer than 5 series points? Do they go negative? | (a) Allow negative (shows as -2, -3, etc.), (b) Floor at 0 |
| 8 | **UI for disconnected player's board:** How to show other players that someone is offline? | (a) Grayed-out board + "Offline" badge, (b) Collapse their board area, (c) Nothing visible |

---

## 7. Files to Modify (Engineering Reference)

> **Note: Do NOT start implementation until all Open Questions above are resolved.**

| File | What Changes |
|------|-------------|
| `src/game/gameLogic.js` | New function: `dissolvePlayer(state, playerId)` — collects all player cards, returns to deck, shuffles, removes player from list |
| `src/game/useGameState.js` | New reducer cases: `PLAYER_QUIT` (intentional) and `SKIP_DISCONNECTED_TURN` (auto-skip on soft disconnect) |
| `src/game/series.js` | New function: `applyQuitPenalty(playerName)` — deducts 5 points from series immediately on quit |
| `src/App.jsx` | (a) Handle `PLAYER_LEFT` from server: distinguish soft disconnect vs intentional quit via message type; (b) Update `handleGoHome` to dispatch `PLAYER_QUIT` before navigating home |
| `src/components/screens/GameScreen.jsx` | Update Home button confirm dialog — add warning about -5 series penalty and card dissolution |
| `worker/index.js` | Add new message type `PLAYER_QUIT` (intentional) separate from existing `PLAYER_LEFT` (natural disconnect) |

---

## 8. Success Metrics (How to Know This Worked)

| Metric | Target |
|--------|--------|
| Sessions that complete after 1 player leaves | >80% (currently 0%) |
| Player complaints about "game breaking" on disconnect | Drops to near 0 |
| Rage-quit rate (intentional Home press during losing game) | <10% of all games |
| Reconnect success rate for soft-disconnected players | >90% within 5 minutes |

---

## 9. Revised Solution: Turn Timer as Universal Mechanism

*Added: 2026-06-22 — decisions finalized in product discussion with Satvik*

### The Proposal in One Line
> Give every player a **1-minute turn timer**. When it expires, a 5-second grace window opens, then the turn auto-skips. This single mechanism — by default — also handles disconnected players: their timer expires, turn skips, game continues. No special disconnect-detection code required.

---

### Why This Is the Right Approach

| Strength | Explanation |
|----------|-------------|
| **Two problems, one mechanism** | Game pacing + disconnect recovery solved together. No new disconnect-detection infrastructure. |
| **Reduces tech debt** | No `PLAYER_LEFT` → detect → skip logic needed. Timer handles it universally by default. |
| **Keeps casual games moving** | Even without disconnects, prevents one slow player from holding everyone. |
| **Predictable UX** | Every player knows: 1 minute → 5-second grace → turn passes. Zero ambiguity. |

---

### Precise Timer Specification (All Decisions Finalized)

#### Turn Structure with Timer

```
Turn Start
  │
  ├── DRAW PHASE ──────────── 10-second timer
  │     Player must tap "Cards Draw Karo"
  │     If 10s elapses → turn auto-skips entirely (no cards drawn)
  │     Game advances to next player
  │
  └── (If cards drawn successfully)
        │
        ├── PLAY PHASE ─────── Remaining time from the full 60-second turn timer
        │     Covers: card selection, playing card, AND all follow-up selections
        │     (e.g., selecting which property to steal in Sly Deal — still on the clock)
        │     If 60s elapses → 5-second grace window opens
        │     If action completes within grace → that action is valid
        │     If grace also elapses → turn ends, all cards played so far are kept
        │
        └── DISCARD PHASE ─── On-clock (part of the 60s)
              If hand > 7 after playing, must discard to 7
              Auto-handled (last drawn cards auto-discarded — see Gap 2 resolution)
```

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Draw phase timer** | **10 seconds** | Quick tap; disconnected players skipped fast |
| **Full turn timer** | **60 seconds** | Enough for strategy; prevents delays |
| **Grace window** | **5 seconds** | Last-second plays still count |
| **Timer covers** | DRAW + PLAY + follow-up selections + DISCARD | Entire turn lifecycle — no pause exploits |
| **Timer pauses for** | Other players' responses only (rent payment, Just Say No, Forced Deal target's decision) | Responders need their own time |
| **Who runs the clock** | **Host device** — broadcasts `TURN_EXPIRED` to all guests | Pragmatic; reuses existing trust model |
| **Visible to** | All players — countdown shown on all screens | Transparency and urgency |
| **On 10s draw expiry** | Full turn skipped, no draw, game advances | Handles draw-phase disconnects fast |
| **On 60s play expiry** | 5s grace; then turn ends, cards played so far are kept | Partial turns are valid |

---

### Gap Resolutions (All Decided)

#### Gap 1: Timer covers action follow-ups — ✅ RESOLVED

**Decision:** The 1-minute timer covers **everything** — card selection, playing the card, AND the follow-up (e.g., selecting which property to steal). Timer does NOT pause after a card is played.

**Rationale:** Pausing after each card play would be exploitable. A player could play a card instantly and then take unlimited time on the target selection. The 1 minute is the total budget for the entire turn.

---

#### Gap 2: Hand size when turn is auto-skipped — ✅ RESOLVED

**Scenario A — Disconnected at DRAW phase:**
- 10s timer elapses → turn skipped entirely
- Player draws nothing → hand stays exactly as it was at end of last turn
- Clean and simple

**Scenario B — Disconnected after drawing, during PLAY phase:**
- Player drew 2 cards (hand is now 7+2 = 9 cards max)
- They disconnect before playing anything
- 60s timer elapses → **last 2 cards drawn are auto-discarded** to the discard pile
- Hand returns to exactly 7 cards
- Game advances to next player

**Logic for all future skipped turns (while still disconnected):**
- DRAW phase → 10s → skip (no draw) → hand stays at 7
- Hand never grows beyond 7 while disconnected
- No accumulation problem

---

#### Gap 3: Whose clock — ✅ RESOLVED

**Decision:** Host-side timer. Host runs the countdown on their device and broadcasts `TURN_EXPIRED` to all guests when it fires.

**Assumption:** Host has a stable connection (reasonable for a casual friend-group game).

---

#### Gap 4: Host migration — DEFERRED TO SEPARATE DOCUMENT

See [`host-migration-process.md`](./host-migration-process.md) for full design.

**For now:** If host soft-disconnects → game pauses → guests see "Host disconnected — waiting..." → host has 2 minutes to reconnect → if not, game ends and series points awarded based on current board state.

---

#### Gap 5: Reconnect — hand state — ✅ RESOLVED

**Decision:** When a soft-disconnected player reconnects using the same room code:
- Host sends current `GAME_STATE` to the reconnecting guest
- **Hand is restored exactly** as it was when they disconnected
- Board (properties, bank) also restored exactly
- Player sees a summary: *"3 turns passed while you were away"*
- They play their next turn normally when it comes around

---

### Actions Against Disconnected Players — Full Specification

> **Core rule:** A disconnected player cannot defend themselves. The game treats them as an absent landlord — anyone can act on their properties without resistance.

#### Defensive cards (Just Say No / Insurance) — do NOT auto-trigger

| Situation | Behavior |
|-----------|----------|
| Someone plays **Sly Deal** against disconnected player | Steal succeeds. Even if disconnected player has Just Say No in hand, it does not trigger. |
| Someone plays **Deal Breaker** against disconnected player | Steal succeeds. Just Say No and Insurance in disconnected player's hand are ignored. |
| Someone plays **Forced Deal** against disconnected player | Swap succeeds. No defense. |
| Someone plays **Sabotage** (custom) against disconnected player | Swap succeeds. No defense. |

**Rationale (Satvik's analogy):** Like leaving your property unattended — no one is there to say no. Real-world consequences for being absent.

---

#### Rent Collection and Debt Collector — Auto-Pay Logic

This is the most complex case. The disconnected player cannot choose how to pay.

**Problem:** For ₹12Cr rent, a player might have: ₹10+₹2, or ₹5+₹4+₹3, or ₹8+₹4, etc. Which combination to use is a player judgment call — but the player is absent.

**Proposed options (pick one before implementation):**

| Option | Algorithm | Fairness to Creditor | Fairness to Debtor | Code Complexity |
|--------|-----------|---------------------|--------------------|----------------|
| **A: Cash-First Greedy** *(Recommended)* | Pay from bank (highest denom first). If bank runs out before covering debt, pay from properties (lowest value incomplete sets first). May overpay. | ✅ High — gets paid | ✅ Reasonable — cash used before properties | 🟡 Medium |
| **B: Cash-Only, Forgive Remainder** | Pay all bank cash available. If insufficient, the rest is forgiven. No properties ever auto-transferred. | ⚠️ Low if debt > cash | ✅ High — properties protected | 🟢 Simple |
| **C: Full Asset Greedy (existing algorithm)** | Reuse existing `collectPayment(debtor, amount)` — mixes bank + properties sorted by value desc. | ✅ High | 🟡 OK — may give up valuable property before exhausting smaller bank notes | 🟢 Zero new code |
| **D: Minimum-Overpay Search** | Try all card subsets to find exact or minimum overpay combination | ✅ High | ✅ High | 🔴 Complex — exponential search |

> **Sanika's recommendation: Option A — Cash-First Greedy.**
>
> Intuition: "Pay with cash first, only use property if cash isn't enough." This mirrors what a reasonable player would do manually. Protects the disconnected player's board as much as possible, while still ensuring the creditor gets paid. Only slightly more code than Option C.

**Cash-First Greedy — exact algorithm:**

```
1. Sort disconnected player's bank cards by value descending
2. Draw from bank until debt is covered OR bank is exhausted
3. If debt still unpaid after bank:
   a. Collect all properties from INCOMPLETE sets only (never steal from complete sets — complete sets are always protected by game rules)
   b. Sort those properties by value ascending (give up cheapest ones first)
   c. Draw from properties until debt covered
4. If debt still unpaid after all non-complete assets: remainder is forgiven (debtor can't pay more than they have)
5. Transfer paid cards to creditor's bank (same as existing applyPayment logic)
```

> **Note:** Complete property sets are **never** auto-transferred for debt payment, even on disconnection. This is consistent with existing game rules where complete sets cannot be targeted by rent payment auto-selection.

---

### Updated Open Questions (For Final Decision Before Coding)

| # | Question | Options | Status |
|---|----------|---------|--------|
| T1 | Timer covers action follow-ups? | ✅ **Yes — entire turn on clock** | RESOLVED |
| T2 | Hand size on skipped turns | ✅ **10s draw skip + auto-discard last 2 if drew before disconnect** | RESOLVED |
| T3 | Timer authority | ✅ **Host-side clock** | RESOLVED |
| T4 | Host migration | ✅ **Deferred — separate doc** | RESOLVED |
| T5 | Reconnect hand restore | ✅ **Exact restore** | RESOLVED |
| T6 | Actions against disconnected players | ✅ **No auto-defense, proceed without resistance** | RESOLVED |
| **T7** | **Auto-pay algorithm for rent/debt collector** | **(A) Cash-first greedy, (B) Cash-only forgive rest, (C) Existing algorithm, (D) Optimal search** | **🔴 OPEN — decide before coding** |
| **T8** | **Timer in Pass-and-Play mode?** | (a) Yes — same 60s, (b) No — P&P is relaxed, timer optional/configurable | **🔴 OPEN** |
| **T9** | **Timer UI** — where shown? | (a) Countdown ring visible to all, (b) Only to current player, (c) No visual | **🔴 OPEN** |
| **T10** | **Does timer replace intentional quit flow?** | (a) Yes, (b) No — keep Home → confirm → -5 penalty + card dissolution for deliberate exits | **🔴 OPEN** |

---

### Trade-offs Summary

| Trade-off | Pro | Con |
|-----------|-----|-----|
| Single timer mechanism for all absences | Reduces code significantly | One mechanism handles many edge cases — needs careful testing |
| 10s draw phase timer | Fast skip for disconnected players | Legitimate slow-network player might miss their draw |
| Timer not pausing for action follow-ups | Cannot be exploited | Player mid-Sly-Deal selection when timer expires has incomplete action — needs edge case handling |
| Host-side clock | No server changes | Host device lag or manipulation possible (low risk in friend group context) |
| Cash-first greedy auto-pay | Intuitive, protects disconnected player's board | Still may overpay in cash (normal in Monopoly Deal — exact change not guaranteed) |
| No host migration (this sprint) | Avoids 3-sprint complexity | Host disconnect still breaks game temporarily |

---

## 10. Files to Modify (Engineering Reference — Updated)

> **Note: Do NOT start implementation until Open Questions T7–T10 above are resolved.**

| File | What Changes |
|------|-------------|
| `src/game/gameLogic.js` | (1) New `dissolvePlayer(state, playerId)` for intentional quit; (2) New `autoPayForDisconnected(state, debtorId, creditorId, amount)` using cash-first greedy; (3) New `autoDiscardLastDrawn(state, playerId, count)` for draw-then-disconnect scenario |
| `src/game/useGameState.js` | New reducer cases: `DRAW_TIMER_EXPIRED`, `PLAY_TIMER_EXPIRED`, `PLAYER_QUIT` |
| `src/game/series.js` | New `applyQuitPenalty(playerName, points = 5)` — records penalty immediately on intentional quit |
| `src/App.jsx` | (1) Start turn timer on `DRAW` phase; (2) Broadcast `TURN_EXPIRED` when timer fires (host only); (3) Handle received `TURN_EXPIRED` on guest side; (4) On reconnect: send full `GAME_STATE` to rejoining player |
| `src/components/screens/GameScreen.jsx` | (1) Render countdown timer UI; (2) Update Home button confirm dialog with -5 penalty warning; (3) Show "X turns passed while you were away" on reconnect |
| `worker/index.js` | (1) Relay new message type `TURN_EXPIRED`; (2) Relay `PLAYER_QUIT` separately from `PLAYER_LEFT`; (3) On reconnect to existing room — route `GAME_STATE_SYNC` from host to rejoining player |

---

## 11. Success Metrics (How to Know This Worked)

| Metric | Target |
|--------|--------|
| Sessions completing after 1 player disconnects | >80% (currently 0%) |
| Average turn time across all players | <45 seconds (timer working) |
| Rage-quit rate (intentional Home during losing game) | <10% of all games |
| Reconnect success rate for soft-disconnected guests | >90% within 5 minutes |
| Player complaints about "game breaking" on disconnect | Drops to near 0 |

---

*Document created: 2026-06-21*
*Timer proposal + gap resolutions added: 2026-06-22*
*Status: Design — 4 open questions (T7–T10) must be resolved before implementation*
*Owner: Satvik Jain*
