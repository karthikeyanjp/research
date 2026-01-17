# Flow-Next & Ralph Mode Deep Dive

**Saved:** 2026-01-17
**Source:** @gmickel (Gordon Mickel)
**Repo:** https://github.com/gmickel/gmickel-claude-marketplace ⭐380

---

## 🏗️ What is Flow-Next?

A Claude Code plugin by Gordon Mickel that implements a **plan-first, file-based** workflow for AI coding agents.

**Core Philosophy:** *"Process failures, not model failures."*

---

## Problems It Solves

| Problem | Solution |
|---------|----------|
| Starting to code before understanding | **Plan-first** workflow |
| Forgetting the plan mid-implementation | **Re-anchoring** every task |
| Context window fills up | **Fresh session** per task |
| Single model blind spots | **Cross-model review** gates |
| Infinite retry loops | **Auto-block** stuck tasks |

---

## 🔄 Architecture: Ralph Mode (Autonomous)

```
┌──────────────────────────────────────────┐
│  ralph.sh (external bash loop)           │
│  ┌────────────────────────────────────┐  │
│  │ while tasks remain:                │  │
│  │   1. Spawn fresh claude -p session │  │
│  │   2. Re-anchor (read specs, git)   │  │
│  │   3. Execute ONE task              │  │
│  │   4. Cross-model review            │  │
│  │   5. If SHIP → next task           │  │
│  │   6. If fail N times → auto-block  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

**Key insight:** Loop is OUTSIDE Claude. Each iteration = brand new context. File I/O (`.flow/` directory) is the state, not the transcript.

---

## 🆚 Anthropic's ralph-wiggum vs Flow-Next Ralph

| Aspect | Anthropic's Plugin | Flow-Next |
|--------|-------------------|-----------|
| **Session** | Single, accumulating | Fresh each iteration |
| **Context** | Grows → drifts | Clean slate always |
| **Failed attempts** | Pollute future | Gone with session |
| **Re-anchoring** | ❌ None | ✅ Every task |
| **Review** | Self-review only | Cross-model gates |
| **Stuck detection** | Max iterations | Auto-blocks task |
| **State** | In-memory transcript | File I/O (auditable) |

---

## 🚦 Quality Gates (3 Layers)

### 1. Multi-Model Reviews
- Uses **RepoPrompt** (macOS) or **Codex CLI** (cross-platform)
- Second model reviews first model's work
- *"Two models catch what one misses"*

### 2. Receipt-Based Gating
```json
{"type":"impl_review","id":"fn-1.1","mode":"rp","timestamp":"..."}
```
No receipt = no progress. Treats agent as **untrusted actor**.

### 3. Review Loops Until SHIP
Reviews block until:
```xml
<verdict>SHIP</verdict>
```
Not just "flag and continue" — actually blocks progress.

---

## 🆕 Plan-Sync Subagent (v0.12+)

> *"Specs lie. Code doesn't."*

After every task:
1. Diffs spec vs actual implementation
2. Patches downstream task specs with real APIs/names/types
3. Keeps plan aligned with reality

Critical for overnight runs with 100+ subtasks.

---

## 🛠️ Commands Reference

```bash
/flow-next:plan         # Research codebase, create epic + tasks
/flow-next:work         # Execute tasks with re-anchoring  
/flow-next:interview    # Deep spec refinement (40+ questions)
/flow-next:plan-review  # Cross-model plan review
/flow-next:impl-review  # Cross-model implementation review
/flow-next:ralph-init   # Scaffold autonomous loop
```

---

## 🧪 Running Ralph

```bash
# Install
/plugin marketplace add https://github.com/gmickel/gmickel-claude-marketplace
/plugin install flow-next
/flow-next:setup

# Test one iteration first
scripts/ralph/ralph_once.sh

# Full autonomous run
scripts/ralph/ralph.sh

# With live monitoring
scripts/ralph/ralph.sh --watch verbose
```

---

## 💡 Key Takeaways for Platform Engineers

1. **Treat AI agents like distributed systems** — receipt-based gating, idempotent operations, file I/O as state
2. **Fresh context > accumulated context** — prevents drift
3. **External orchestration** — bash loop controls agent, not vice versa
4. **Multi-model validation** — same principle as code review
5. **Plan-sync** — keep specs aligned with implementation reality

This is basically **GitOps for AI agents**. The `.flow/` directory is your source of truth, not the conversation.

---

## Resources

- **Main Repo:** https://github.com/gmickel/gmickel-claude-marketplace
- **OpenCode Port:** https://github.com/gmickel/flow-next-opencode
- **Author Twitter:** https://x.com/gmickel
- **TUI Monitor:** `bun add -g @gmickel/flow-next-tui`

---

## Original Tweet (Bookmarked)

https://x.com/gmickel/status/2009939771171434867
