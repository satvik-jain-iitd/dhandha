# Rules & Lessons — Monopoly Deal

Append-only. One entry per mistake discovered during work. Build over time; never delete or summarize.

---

## Template

```yaml
- mistake: <description of what went wrong>
  root_cause: <why it happened>
  prevention_rule: <how to avoid next time>
  added_on: <YYYY-MM-DD>
```

---

## Lessons Logged

- mistake: Hand cards in ActionModal payment/selection sheets rendered as plain Typography boxes instead of colorful <Card> component, breaking strategic decision-making
  root_cause: Two read-only hand blocks (PaymentSheet + PlayerContextView) built inline Typography with bordered Box wrappers instead of reusing the existing <Card mini showValue /> pattern — even though Card was already imported and used for cash/property rows in the same file
  prevention_rule: Any place showing a hand card (even read-only) must reuse <Card card={card} mini showValue /> instead of inline Typography. Search for hand.map(card => in ActionModal/selection components before adding new blocks. Visual consistency = color per card type.
  added_on: 2026-06-20

- mistake: "Agent (Sonu + Aman) completed work and updated STATE.md without sending tmux notification to waiting agents (Sanika)"
  root_cause: "Assumed shared file visibility would alert waiting agents; did not realize agents cache file reads and miss background updates from other agents"
  prevention_rule: "EVERY agent, EVERY time work is done, MUST send direct tmux notification to ALL other team members. Non-negotiable. Checklist: (1) write to STATE.md, (2) send tmux msgs to both other agents with exact line reference. Read CLAUDE.md§AGENT COMPLETION CHECKLIST before finishing any work."
  incident: "monopoly-deal-fix-aii sprint, 2026-06-20 23:15–23:30 IST. Sonu signed off in STATE.md§156–179 + Aman signed off in STATE.md§125–154. Neither sent tmux notification. Sanika re-read STATE.md per Main's signal, found sign-offs, but only because Main detected silent write and manually escalated. Cost: 10+ min blockage. Bad precedent if repeated."
  added_on: 2026-06-20

- mistake: "Main (Claude Code) failed to launch agent CLIs correctly. Tried 'claude code' instead of 'opencode'; tried local tmux send without waiting for CLI initialization; did not understand that each agent needs a separate CLI instance in their tmux pane."
  root_cause: "Main did not have documented procedure for agent CLI launch. Made assumptions about command names and tmux automation without verifying against actual CLI behavior. Did not account for CLI initialization delay or need for cd to project root."
  prevention_rule: "BEFORE launching agents, read ~/.claude/team.yaml (agent_launch_procedure section) and project CLAUDE.md (Multi-Agent CLI Launch section). Use EXACT commands from documentation — no variations. Use 'opencode' for Sonu/Aman (Tech Lead / SDE2), 'agy' for Sanika (SDE1, fallback to opencode if quota exceeded). Always sleep 3 seconds after each launch to allow CLI initialization. Always verify with tmux capture-pane before assuming agent is ready."
  incident: "2026-06-21 11:00-11:10 IST. Main tried 'claude code' + failed. User had to manually launch 'opencode' in all three panes. Root cause: no documented procedure existed. Solution: added agent_launch_procedure to team.yaml + Multi-Agent CLI Launch section to CLAUDE.md with EXACT commands and sleep timing."
  added_on: 2026-06-21

- mistake: "Original plan called for mount-useEffect + setScreen to restore game state, causing 1-frame home screen flash before game appears"
  root_cause: Assumed setScreen must be called imperatively after state init. Did not realize useState can derive initial value from sync localStorage read at component top level.
  prevention_rule: "For sync-capable persistence (localStorage), use `useRef(loadGame())` + `useState(ref ? 'target' : 'default')` pattern. This sets both state AND screen before first render — zero flash. Reserve mount-useEffect + setScreen for async persistence (IndexedDB, network) where the init value isn't available synchronously. This pattern was used in monopoly-deal game-progress-persistence and eliminated the 1-frame flicker entirely."
  added_on: 2026-06-21
  deprecation_note: "Auto-persistence was removed on 2026-06-22 (Part A: session picker replaced auto-restore). The zero-flash pattern is no longer used in App.jsx — sessions are now manually selected via HomeScreen. Rule preserved for reference if sync persistence is re-added."

- mistake: "Auto-persistence (localStorage) caused game state leak across browser profiles on shared devices. Link sharing exposed saved games to unintended users."
  root_cause: "localStorage is per-origin per-browser, NOT per-user. App auto-saved after every action and auto-restored on load, with no user isolation."
  prevention_rule: "For client-only PWAs, never auto-save + auto-restore game state without user confirmation. If you need session persistence: (1) use a manual session picker UI where user selects which session to resume, (2) cap at 5 sessions, (3) never auto-navigate to game screen from saved state. For multiplayer, move state to server-side (Durable Objects SQLite) where it's isolated per-room, not per-browser."
  added_on: 2026-06-22
  phase: "post-ship reflection"

- mistake: "Double Rent card could be played even when player had no Rent card in hand or <2 plays remaining, wasting the card"
  root_cause: No validation guards on DOUBLE_RENT — neither in UI layer (PlayOptions) nor reducer layer (useGameState.js handler). Assumed player would always play Double Rent correctly, ignoring the follow-up Rent requirement.
  prevention_rule: "For action cards that require a follow-up card (like Double Rent → Rent), always add dual-layer guards: (1) reducer-level early return to prevent invalid state, (2) UI-level disabled button with reason text. The two guards are independent — one is defense-in-depth for the other. The reason text in UI should be specific ('Rent card nahi hai haath mein') not generic ('Cannot play')."
  added_on: 2026-06-22

- mistake: "Selected card translateY(-10px) lift + orange outline clipped inside CardHand container"
  root_cause: "Setting overflowX: 'auto' caused CSS to implicitly set overflowY: 'auto' (CSS spec: if one axis is not 'visible', the other cannot be 'visible'). This clipped the upward translate and outline at the container boundary."
  prevention_rule: "Whenever overflowX is set to any value other than 'visible' in a scrolling container that also contains elements with vertical transforms (translateY, scale transforms), always explicitly set overflowY: 'visible' to prevent the implicit CSS cascade from clipping the upward lift. This applies to any card/selection UI where selected cards rise above the row."
  added_on: 2026-06-23

- mistake: "endTurn() left stale PLAY turnTimeout/turnStartedAt in game state after advancing to DRAW phase, causing ~1 frame of a 90s timer in the DRAW UI before startTurn() reset it"
  root_cause: "Turn state fields (turnTimeout, turnStartedAt) were only initialized in initGame() and reset in startTurn(). endTurn() advanced the phase and player index but did not touch timer fields, letting the previous PLAY timeout leak into DRAW."
  prevention_rule: "Whenever game state has phase-dependent derived fields (like timer data), every function that changes the phase must also update those fields to match the new phase. In reactive UI, stale derived fields will render for at least one frame before the correct function (startTurn in this case) overwrites them. Apply the 'update on every phase change' rule."
  added_on: 2026-06-23

- mistake: "Timer color thresholds were specified as absolute seconds (>45s green, 22-45s amber, <22s red), which only works for 90s PLAY timer — for 30s DRAW timer it can never be >45s"
  root_cause: "Spec wrote absolute thresholds tuned to PLAY duration without considering DRAW's shorter 30s window. Absolute thresholds don't generalize across timer durations."
  prevention_rule: "For any UI component that renders different timer durations, use percentage-based thresholds (>50%, 25-50%, <25%) instead of absolute time values. Percentage-based thresholds automatically adapt to any duration and maintain consistent visual meaning (half+ remaining = green, quarter+ = amber, critical = red)."
  added_on: 2026-06-23

- mistake: "cloudflared tunnel route dns added a CNAME under the wrong Cloudflare account because it used the default cert.pem for API auth, not the tunnel credentials file"
  root_cause: "cloudflared tunnel route dns authenticates via cert.pem (linkright account). The tunnel credentials file (letsdwelo account) is only for tunnel runtime auth, not API operations."
  prevention_rule: "For DNS management on different Cloudflare accounts, never use cloudflared tunnel route dns from a server authenticated to a different account. Either: (1) use Cloudflare API directly with an API token for the target account, (2) login cloudflared separately for each account, or (3) add DNS records via Cloudflare Dashboard."
  added_on: 2026-06-23

- mistake: "PM2 cloudflared start command omitted --credentials-file, causing the tunnel to use default cert.pem auth and default ingress config (port 8000) instead of the correct tunnel/port"
  root_cause: "cloudflared launched as 'pm2 start cloudflared -- tunnel run' without UUID or credentials. This reverted to cert.pem-based auth even though config file declared the correct tunnel."
  prevention_rule: "When starting a cloudflared tunnel via PM2, always specify: --credentials-file + tunnel UUID. Full command: pm2 start cloudflared --name <name> -- --credentials-file <path.json> tunnel run <uuid>. Verify config.yml has correct ingress rules for THIS tunnel — replacing default config.yml breaks other tunnels."
  added_on: 2026-06-23

- mistake: "Host migration broken in production — HOST_CHANGED never sent when host DC'd in-game because wasHost always evaluated to false"
  root_cause: "Dynamic room.hostName getter returns connectedPlayers[0].name, where connectedPlayers filters by wsMap.has(p.name). The wasHost = (room.hostName === name) check was placed AFTER wsMap.delete(name), so the just-disconnected player was already filtered out — wasHost was always false regardless of who disconnected."
  prevention_rule: "When working with dynamic getters that depend on in-memory collections (wsMap, players, etc.), always capture state-dependent values BEFORE mutating those collections. The pattern: (1) read → (2) delete → (3) use read value. NEVER: (1) delete → (2) read (you already deleted the data the getter depends on). This applies to any computed property whose backing collection is about to change."
  added_on: 2026-06-23

- mistake: "agent-browser snapshot -i didn't capture game mode buttons as clickable refs — only @c cursor-interactive refs worked"
  root_cause: "Game mode 'cards' (Pass & Play, Online, etc.) used div-based click handlers on parent containers, not ARIA button roles. agent-browser's accessibility tree snapshot (-i) filters to ARIA-role elements; cursor-interactive scan (-C) was needed to detect the actual click targets."
  prevention_rule: "When using agent-browser for E2E testing: (1) always include -C flag in snapshot to capture cursor-interactive elements, (2) for complex SPAs, use a combination of -i (ARIA) and -C (cursor) to find all clickable elements, (3) if a click target isn't found in the accessibility tree, try clicking the parent container via CSS selector with html/inspect."
  added_on: 2026-06-23

- mistake: "game.dhandha.letsdwelo.in URL used for E2E testing without verifying DNS first — was NXDOMAIN"
  root_cause: "Assumed the domain existed because it was mentioned in the task description. No DNS check was done before attempting to browse."
  prevention_rule: "Before any E2E test session: (1) verify target URL resolves via dig/nslookup, (2) verify HTTP status via curl -I, (3) if NXDOMAIN or 404, investigate actual deployment URL before proceeding. Don't trust stated URLs — verify them first."
  added_on: 2026-06-23

- mistake: "HOST_CHANGED handler only degraded old host to guest but never upgraded a guest to new host, leaving the promoted player stuck in guest mode"
  root_cause: "Handler was written with only one if-branch (host→guest). The reverse transition (guest→host) was simply not considered."
  prevention_rule: "For any multi-role state message handler that changes actor identity, always write ALL state transitions explicitly — both upgrade and downgrade paths. The transition matrix helps: enumerate currentRole × newRole → action. If only one transition is handled, the reverse is a guaranteed bug."
  added_on: 2026-06-23

- mistake: "handleMessage useCallback closed over mpGuestState but didn't include it in deps array — when HOST_CHANGED arrived, the callback might read stale state"
  root_cause: "React's useCallback captures closure variables at creation time. Adding state vars to deps causes re-creation on every update, which breaks WS message subscription. Used mpGuestState from closure instead of a ref."
  prevention_rule: "When a stable callback (like a WS message handler) needs the latest value of a frequently-changing state variable, use a useRef to store it and read ref.current at access time. This avoids: (1) stale closure reads, (2) unnecessary useCallback re-creation, (3) WS re-subscription churn. Pattern: write to ref.current alongside setState, read ref.current inside callback."
  added_on: 2026-06-23

- mistake: "Race condition: HOST_CHANGED could arrive before the newly-promoted player's first GAME_STATE, leaving them with no game state to init"
  root_cause: "Host disconnect and state broadcast are async. A player who just joined mid-game (no mpGuestState yet) could be promoted before receiving their first GAME_STATE packet."
  prevention_rule: "When a handler depends on state data that arrives via a different message type, use a pending/init flag pattern: (1) set a ref flag when data isn't available yet, (2) check the flag in the message handler that delivers the required data, (3) clear the flag after dispatch. This handles inter-message race conditions without coupling message ordering."
  added_on: 2026-06-23

- mistake: "Error 1033 persisted 15+ min because tunnel was connected but using wrong ingress config (oracle-linkright port 8000 instead of dhandha port 3001)"
  root_cause: "Tunnel connections register successfully even with wrong ingress rules. Tunnel health metrics look fine, but traffic doesn't reach the correct local service."
  prevention_rule: "When debugging tunnel error 1033: (1) verify tunnel connections are registered, (2) verify ingress rules match the local service port, (3) test local service directly (curl localhost:PORT/health), (4) check config.yml is the one actually being used. Never assume connected tunnel = correctly routing traffic."
  added_on: 2026-06-23

- mistake: "deepseek_* AI scratch files (deepseek_markdown_20260624_0d0d52.md, deepseek_yaml_20260624_530174.yaml) accumulated in repo root as untracked throwaway artifacts, duplicating CLAUDE.md and team.yaml content"
  root_cause: "AI CLI tools can generate temporary scratch files during multi-agent sessions. These files are left behind when the tool exits, and nobody notices until a cleanup audit."
  prevention_rule: "Run `git status -s` at the end of every sprint or before every commit. Look for unexpected untracked .md/.yaml files with AI-tool naming patterns (deepseek_*, gemini_*, claude_*, scratch_*, *_202*). Delete these before committing. They clutter the workspace and confuse future readers."
  added_on: 2026-06-25

- mistake: "STATE.md grew to 79,889 bytes (~19,972 tokens) — 4x the 5,000 token limit — because closed sprint handoffs were never compressed per project_structure.yaml overflow procedures"
  root_cause: "No one was monitoring STATE.md's token budget. Each sprint appended ~25-40KB of detailed handoffs, and no compression step was built into the sprint completion checklist."
  prevention_rule: "Add a token budget check to EVERY sprint's sign-off checklist: `wc -c STATE.md` — if >15,000 bytes (~3,750 tokens), run the overflow procedure IMMEDIATELY before closing the sprint. The procedure is documented in project_structure.yaml overflow_procedures.state: merge closed sprints into a single `## Handoff Summaries` paragraph. Never let STATE.md cross 20,000 bytes."
  added_on: 2026-06-25

- mistake: "Pre-implementation design docs (docs/host-migration-process.md, docs/session-progress-saving-mechanism.md) remained as active docs/ files after the feature shipped, creating stale documentation that contradicts implemented behavior"
  root_cause: "Design docs are written before implementation but there is no habit of archiving them when the feature ships. The 'implemented' signal in STATE.md doesn't trigger a docs cleanup step."
  prevention_rule: "When a feature ships (UAT sign-off), add a final sub-step: check if any design docs in docs/ exist for that feature. If yes, either (1) update them to match implemented behavior, or (2) archive them to docs/archive/ with a note 'superseded by implementation on YYYY-MM-DD'. Design docs that don't match reality are worse than no docs."
  added_on: 2026-06-25

- mistake: "Sanika (Implementer), tasked only with STATE.md compression (delete old verbose entries, insert a pre-drafted summary block, copy-paste two entries verbatim), ran `git checkout -- STATE.md` mid-task. This silently reverted the entire working file to the last git commit, destroying ~700 lines of uncommitted handoff history accumulated across multiple prior sessions (Sprint 3b Timer UI, Sprint 3 Host Migration UAT/Reflection, Sprint 4 E2E QA, an undocumented wrong-password/timer-skip bugfix sprint) — none of it had ever been committed."
  root_cause: "First compression attempt botched (prepended the summary on top of old content instead of replacing it, so the file grew instead of shrank). Agent then reached for `git checkout` as a 'reset to clean state' move without realizing the working tree held ~700 lines of session history that existed nowhere else — not in any commit, not in any stash. The approved plan only specified rm/mkdir/mv/git add; git checkout was never in scope. The `only_from_approved_plans` restriction was violated by an implicit 'helpful' shortcut."
  prevention_rule: "Implementer agents must NEVER run `git checkout -- <file>`, `git reset`, `git stash`, or any command that discards working-tree content unless that exact command is explicitly listed in the approved plan. If an edit attempt goes wrong, the correct recovery is to re-read the file and retry the edit precisely — never 'reset and start over' on a file that may hold uncommitted, unrecoverable session state. STATE.md/CONTEXT.md/RULES.md are append-mostly logs with no other backing store; treat them as more fragile than source code, which is at least diffable against git history. Orchestrator should commit STATE.md/RULES.md/CONTEXT.md periodically during long sessions so 'last commit' is never more than one sprint stale."
  incident: "2026-06-25 00:46-01:30 IST repo-cleanup-audit sprint, Step 4. Main (orchestrator) caught the regression via STATE.md mtime/size monitoring (file grew then suddenly shrank, known headings vanished), interrupted the agent before it could compound the damage with a hallucinated 'reconstruction', and recovered ~90% of the lost content verbatim from its own prior tool-call transcript in the same conversation. The remaining gap (one bugfix sprint's narrative — the code itself was safely committed under d8f8a5d/c8951fa/e13e376/d0c3d9b) was reconstructed from `git log --oneline -- STATE.md` and `git show --stat` of those commits — accurate but not the original prose. Zero production code was affected; this was pure process-documentation loss."
  added_on: 2026-06-25
