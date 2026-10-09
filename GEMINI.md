# Monopoly Deal — Project Collaboration Protocol

Guidelines for the Monopoly Deal React PWA workspace, coordinating the Main orchestrator and the sandboxed multi-agent session.

---

## 🛠️ Project Commands

| Operation | Command |
|---|---|
| **Build Project** | `npm run build` |
| **Run Dev Server** | `npm run dev` |
| **Headless Simulation** | `node scripts/simulate.js` |

## 🧠 GitNexus Codebase Context
- **Index ID**: `Monopoly-Deal-card-game-indian-version`
- **MCP Analysis**: Run impact analysis before modifying any symbol:
  ```bash
  gitnexus_impact({target: "symbolName", direction: "upstream"})
  ```

---

## 👥 Team Roles & Responsibilities

Roles and responsibilities are dynamically defined by the active team profile in the global [team.yaml](file:///Users/satvikjain/.claude/team.yaml). For this workspace, refer to `STATE.md` to identify the active `team_profile` (e.g. `software_development`).

### Role-Based Protocols:
- **Lead Role**: Responsible for Analysis (Step 1), Review (Step 6), Reflection (Step 7). Does not write production code.
- **Architect/Planner Role**: Responsible for Planning (Step 2), Architecture/Solution (Step 3), QA (Step 5). Does not write production code.
- **Implementer Role**: Responsible for Implementation & Code Delivery (Step 4). Works only from approved plans and designs.

---

## 🔄 Sprint Workflow Steps

1. **Step 1: ANALYSIS (Lead)** → Research problem/constraints → Post to `STATE.md`
2. **Step 2: PLANNING (Architect)** → Define scope/milestones → Post to `STATE.md`
3. **Step 3: SOLUTION (Architect)** → Define architecture/strategy → Post to `STATE.md`
4. **Step 4: IMPLEMENT (Implementer)** → Produce working code/tests
5. **Step 5: TEST & QA (Architect)** → Validate acceptance criteria
6. **Step 6: REVIEW/UAT (Lead)** → Verify UX, logic, and production readiness
7. **Step 7: REFLECTION (Lead)** → Capture lessons learned in `RULES.md`

---

## 📯 Handoff & Notification Protocol

### 1. Notification Step (MANDATORY)
Every time you finish a step or write a sign-off to `STATE.md`, you **must** send a direct tmux notification to all other worker agents in the active session:
```bash
tmux send-keys -t <session_prefix>:<target_agent_window> "📌 [HH:MM IST] @your_name → @target: <summary ≤12 words> — read STATE.md§<line>" Enter
```
*(Lookup `<session_prefix>` and target agent names in [team.yaml](file:///Users/satvikjain/.claude/team.yaml) under the active team profile).*

### 2. State & Files Protocol
- **CONTEXT.md**: Project truth. Owned by Architect role.
- **STATE.md**: Active task, blockers, handoff history. Owned by All.
- **RULES.md**: Lessons learned. Append-only. Owned by Lead role.
- **No 4th markdown file**: All information must go into the three files above.

---

## 📖 Global Operating Rules
This project follows the global operating manuals and protocols. Refer to:
- [Global Operating Manual](file:///Users/satvikjain/.gemini/GEMINI.md)
- [Global Multi-Agent Protocol](file:///Users/satvikjain/.claude/team.yaml)
- [Global Tool Manual](file:///Users/satvikjain/.claude/tools.yaml)
- [Global Project Structure](file:///Users/satvikjain/.claude/project_structure.yaml)
