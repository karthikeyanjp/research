# Flow-Next: Stages & Phases Explained

**Created:** 2026-01-17  
**Based on:** flow-next-deep-dive.md  
**Author:** Gordon Mickel (@gmickel)

---

## Overview

Flow-Next is a **plan-first, file-based** workflow for AI coding agents. It treats AI like a distributed system with receipts, gates, and external orchestration.

---

## High-Level Workflow

```mermaid
flowchart TD
    subgraph "📋 PLANNING PHASE"
        A[Start] --> B["/flow-next:interview"]
        B --> C["/flow-next:plan"]
        C --> D["/flow-next:plan-review"]
        D -->|SHIP| E[Plan Approved]
        D -->|REVISE| C
    end

    subgraph "🔨 IMPLEMENTATION PHASE"
        E --> F["/flow-next:work"]
        F --> G[Execute Task]
        G --> H["/flow-next:impl-review"]
        H -->|SHIP| I{More Tasks?}
        H -->|REVISE| G
        H -->|BLOCKED| J[Auto-Block Task]
        I -->|Yes| K[Plan-Sync]
        K --> F
        I -->|No| L[✅ Complete]
    end

    J --> M[Manual Intervention]
    M --> F
```

---

## The Ralph Mode Loop (Autonomous)

```mermaid
flowchart TD
    subgraph "🤖 RALPH.SH - External Bash Loop"
        R1[ralph.sh starts] --> R2[Check: Tasks remaining?]
        R2 -->|No| R3[🎉 All Done]
        R2 -->|Yes| R4[Spawn FRESH Claude session]
        R4 --> R5[Re-anchor: Read specs + git status]
        R5 --> R6[Execute ONE task]
        R6 --> R7[Cross-model review]
        R7 -->|SHIP| R8[Write receipt]
        R7 -->|FAIL| R9{Attempts < N?}
        R9 -->|Yes| R4
        R9 -->|No| R10[Auto-block task]
        R8 --> R11[Plan-Sync: Update specs]
        R10 --> R11
        R11 --> R2
    end

    style R4 fill:#e1f5fe
    style R7 fill:#fff3e0
```

**Key Insight:** The loop runs OUTSIDE Claude. Each iteration = brand new context window. Failed attempts don't pollute future runs.

---

## Phase Definitions

### 📋 `/flow-next:plan`
**Purpose:** Research codebase and create structured work breakdown

| What it does | Why it matters |
|--------------|----------------|
| Analyzes existing code patterns | Prevents reinventing the wheel |
| Creates Epic spec (high-level) | Documents the "what" and "why" |
| Breaks into Task specs | Atomic units of work (~4 hrs max) |
| Identifies dependencies | Proper execution order |

**Output:** `.flow/epics/EPIC-XXX.md` + `.flow/tasks/TASK-XXX-NNN.md`

---

### 🎤 `/flow-next:interview`
**Purpose:** Deep requirements gathering before planning

| What it does | Why it matters |
|--------------|----------------|
| Asks 40+ clarifying questions | Catches ambiguity early |
| Refines scope and constraints | Prevents scope creep |
| Documents decisions | Creates audit trail |

**When to use:** Complex features, unclear requirements, new domains

---

### 🔍 `/flow-next:plan-review`
**Purpose:** Cross-model validation of the plan

```mermaid
flowchart LR
    A[Plan Created<br/>by Model A] --> B[Review<br/>by Model B]
    B -->|"<verdict>SHIP</verdict>"| C[✅ Approved]
    B -->|"<verdict>REVISE</verdict>"| D[🔄 Back to Planning]
```

**Why cross-model?** Same model reviewing its own work has blind spots. Different model catches different issues.

---

### 🔨 `/flow-next:work`
**Purpose:** Execute tasks with re-anchoring

**The Re-Anchoring Pattern:**
```mermaid
flowchart TD
    W1[Start Work] --> W2[Read Epic spec]
    W2 --> W3[Read THIS Task spec]
    W3 --> W4[Check git status]
    W4 --> W5[Review related code]
    W5 --> W6[Implement ONLY this task]
    W6 --> W7[Commit changes]
```

**Why re-anchor?** Fresh sessions don't remember the plan. Re-reading specs ensures alignment every time.

---

### ✅ `/flow-next:impl-review`
**Purpose:** Cross-model code review

| Verdict | Meaning | Action |
|---------|---------|--------|
| `SHIP` | Code meets spec | Proceed to next task |
| `REVISE` | Issues found | Retry implementation |
| `BLOCK` | Fundamental problem | Auto-block, needs human |

**Receipt generated:**
```json
{
  "type": "impl_review",
  "task_id": "TASK-001-003",
  "verdict": "SHIP",
  "reviewer": "claude-sonnet",
  "timestamp": "2026-01-17T21:00:00Z"
}
```

---

### 🔄 `/flow-next:ralph-init`
**Purpose:** Scaffold the autonomous loop infrastructure

**Creates:**
```
scripts/ralph/
├── ralph.sh          # Main loop (bash)
├── ralph_once.sh     # Single iteration (for testing)
├── config.env        # Settings (max retries, models, etc.)
└── hooks/            # Custom pre/post task hooks
```

---

## Plan-Sync Subagent

```mermaid
flowchart LR
    PS1[Task Completed] --> PS2[Diff: Spec vs Reality]
    PS2 --> PS3{Divergence?}
    PS3 -->|No| PS4[Continue]
    PS3 -->|Yes| PS5[Patch downstream specs]
    PS5 --> PS6[Update API names/types]
    PS6 --> PS4
```

**Purpose:** Specs drift from reality during implementation. Plan-Sync keeps them aligned.

**Example:** Task 1 creates `UserService.authenticate()` but spec said `AuthService.login()`. Plan-Sync updates Tasks 2-10 to reference the real method name.

---

## State Machine: Task Lifecycle

```mermaid
stateDiagram-v2
    [*] --> pending: Task created
    pending --> in_progress: /flow-next:work
    in_progress --> review: Implementation done
    review --> done: SHIP verdict
    review --> in_progress: REVISE verdict
    review --> blocked: Max retries exceeded
    blocked --> pending: Manual unblock
    done --> [*]
```

---

## File Structure (`.flow/` directory)

```
.flow/
├── state.json              # Current workflow state
├── epics/
│   └── EPIC-001.md         # Epic-level spec
├── tasks/
│   ├── TASK-001-001.md     # Task specs
│   ├── TASK-001-002.md
│   └── TASK-001-003.md
├── research/
│   └── RES-001.md          # Research findings
├── reviews/
│   ├── plan-review-001.md  # Plan review receipts
│   └── impl-review-001-001.md
└── receipts/
    └── receipts.jsonl      # All receipts (append-only)
```

---

## Quality Gates Summary

```mermaid
flowchart TD
    subgraph "Gate 1: Plan Review"
        G1A[Plan] --> G1B{Cross-Model<br/>Review}
        G1B -->|SHIP| G1C[✅ Pass]
        G1B -->|REVISE| G1A
    end

    subgraph "Gate 2: Impl Review"
        G2A[Code] --> G2B{Cross-Model<br/>Review}
        G2B -->|SHIP| G2C[✅ Pass]
        G2B -->|REVISE| G2A
        G2B -->|N fails| G2D[🚫 Block]
    end

    subgraph "Gate 3: Plan-Sync"
        G3A[Reality] --> G3B{Matches<br/>Spec?}
        G3B -->|Yes| G3C[✅ Continue]
        G3B -->|No| G3D[Patch Specs]
        G3D --> G3C
    end

    G1C --> G2A
    G2C --> G3A
```

---

## TL;DR - What Each Command Does

| Command | Phase | One-liner |
|---------|-------|-----------|
| `interview` | Pre-planning | Deep requirements Q&A |
| `plan` | Planning | Create epic + task breakdown |
| `plan-review` | Planning | Cross-model plan validation |
| `work` | Implementation | Execute one task (re-anchored) |
| `impl-review` | Implementation | Cross-model code review |
| `ralph-init` | Setup | Create autonomous loop scripts |

---

## Key Principles

1. **Plan before code** — No implementation without specs
2. **Fresh context always** — Each task = new session
3. **Re-anchor every time** — Read specs before working
4. **Cross-model review** — Two models > one model
5. **Receipts gate progress** — No receipt = no advance
6. **File I/O is state** — `.flow/` is source of truth, not transcript
7. **External orchestration** — Bash controls Claude, not vice versa

---

*This is GitOps for AI agents.* 🚀
