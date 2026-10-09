# Project Context — Monopoly Deal (Indian Version)

## Project

**Name:** Monopoly Deal (Indian Version, "Dhandha")
**Objective:** React + Vite PWA implementing the Monopoly Deal card game with pass-and-play LAN and WebRTC multiplayer support. Full game rules, action modals, and player scoring via immutable state management.

## Problem

Players need a digital card game platform that supports three play modes: local (pass-and-play), LAN (QR-pairing via WebRTC), and (future) online multiplayer. Game state must be synchronized across devices, actions must be validated, and special cards (Sabotage, Insurance, custom rules for 3+ players) must be enforced. UI must show hand, properties, bank, and contextual action modals per game phase.

**Target Users:**
- Casual players (family game nights, friends)
- Platform: Mobile (iOS/Android PWA) + Desktop

**Pain Points:**
- Multiplayer sync failures (before Durable Object rewrite)
- Pass-and-play privacy (opponent sees hand during setup)
- Missing Sabotage card implementation
- Ambiguous action UI (which cards am I seeing? whose properties?)

## Success

**Acceptance Criteria:**
1. Pass-and-play mode works without privacy leaks (modal hides opponent hand)
2. WebRTC pairing works peer-to-peer via QR, with room fallback via signaling server
3. All 8 action modals display correct card context (hand, own properties, opponent properties per action)
4. Sabotage card swaps target's property with multi-opponent selection logic
5. Deal Breaker + Just Say No blocking behavior confirmed
6. Insurance blocks Deal Breaker only (not Sly/Forced Deal); optional pop-up
7. Scoring phase calculates property sets correctly and ranks players
8. Custom 3+ player deck (110 cards with Sabotage + Insurance) works in multiplayer
9. Full PWA (installable, offline fallback, service worker)
10. Beads issue tracking in sync; no 6th markdown file

## Constraints

**Technical:**
- React 19 + Vite 8 (ESM only)
- MUI 9 + Emotion for styling
- useReducer for immutable state (no Redux)
- WebRTC for P2P (no relay unless signaling required)
- Cloudflare Durable Objects for multiplayer room state (new, post-sprint-1)
- GitHub Pages static hosting (no server-side code in main build)

**Business:**
- Free-to-play, no monetization
- No authentication (localStorage for config only)
- No backend database (room state ephemeral via Durable Objects)

**Timeline:**
- Sprint 1 (Jun 17-20): Core game, pass-and-play, privacy + action UI fixes
- Sprint 2 (pending): Sabotage + Insurance, 3+ player custom deck, full multiplayer sync

## Architecture

For system layout, directory maps, execution traces, state flow diagram, enums, and component sheets, see the master [ARCHITECTURE.md](file:///Users/satvikjain/Downloads/Mission-Job-Switch/PWA/monopoly-deal-fix/ARCHITECTURE.md).


## Decisions

Durable architecture/tech decisions. Append dated entries below — never delete old ones, mark superseded ones with `[SUPERSEDED — see <date>]` instead.

| Date | Decision | Why |
|------|----------|-----|
| 2026-06-20 | Multiplayer relay rewritten as a Cloudflare Durable Object (`RoomDO`, one instance per room via `getByName`) using the Hibernatable WebSockets API. | Plain in-memory `Map` in the Worker had no cross-isolate affinity — host and guest could land on different edges, breaking roster sync. |
| 2026-06-20 | 3-file project memory system (CONTEXT.md / STATE.md / RULES.md) replaces 5-file protocol. All architecture, problem, decisions live in CONTEXT.md; execution state + handoffs in STATE.md; lessons in RULES.md. | Simpler coordination, faster handoffs, less file churn. Beads owns task tracking; STATE.md owns execution state. |
