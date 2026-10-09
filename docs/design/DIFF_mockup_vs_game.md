# Dhandha: what each version has

Checked on 2026-10-08. Nothing was changed inside either version.

## The two versions
| Version | Dates (first edit to last edit) | Size | What it is |
|---|---|---|---|
| `2026-06-15__2026-06-25__from-LinkRight-mission_job_switch-PWA` | 15 Jun to 25 Jun 2026 | 439 files, 118 commits | The real game. React app, server, PWA, GitHub repo, live demo. |
| `2026-06-21__2026-06-21__from-Projects-Dhandha_PWA` | 21 Jun 2026 | 1 file (`index.html`) | A design mockup of the home screen (lobby). Not a game. |

## What the big version (V2) has that the mockup lacks
Everything that makes it a game: rules, cards, turns, online play, hotspot play, pass-and-play, results.

## What the mockup (V1) has that V2 lacks
The game modes in V1 are already in V2 (`HomeScreen.jsx`: Pass & Play, Hotspot Khelo, Online Khelo).
Only the look is new. It was a redesign idea for the home screen:
- One screen, no scrolling. The lobby fits the phone.
- Dark navy and amber colours (`#0F172A`, `#1E293B`, `#10B981`, `#F59E0B`). V2 uses none of them.
- A top bar with cash (Rs 25,000) and level (Level 12).
- "Play Online" as the big first button, with a live player count.
- Rules moved into a help drawer, out of the main screen.

## Verdict
V2 is the game. V1 is a design idea that was never applied.
To consolidate: apply the V1 home-screen design to `src/components/screens/HomeScreen.jsx` in V2, then keep V1 only as history.
Waiting for Satvik's yes.

## Open item in V2
V2 has work that was never committed (from June): 6 files changed, 2 files moved into `docs/`, plus `.beads` changes.
It is untouched. Satvik decides: commit it, or drop it.
