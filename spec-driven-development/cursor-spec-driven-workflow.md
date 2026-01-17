# Spec-Driven Workflow for Cursor

**Goal:** Implement Research → Plan → Implement workflow with:
- Specs persisted to Git
- Fresh context per task (prevent drift)
- Sub-agents for task implementation
- History preserved

---

## The Challenge

Cursor doesn't have built-in workflow orchestration like AWS Kiro or Flow-Next. But we can build it using:
1. **Convention + Directory Structure** — file-based state
2. **`.cursorrules`** — enforce workflow discipline
3. **Composer Agents** — sub-task execution
4. **Git** — version control for specs

---

## Architecture

```
project/
├── .cursor/
│   └── rules                    # Cursor rules (workflow enforcement)
├── .specs/                      # All specs live here (Git tracked)
│   ├── epics/
│   │   └── EPIC-001-feature.md  # Epic-level specs
│   ├── tasks/
│   │   ├── TASK-001-subtask.md  # Individual task specs
│   │   └── TASK-002-subtask.md
│   ├── research/
│   │   └── RES-001-findings.md  # Research outputs
│   └── reviews/
│       └── REV-001-feedback.md  # Review receipts
├── .workflow/
│   └── state.json               # Current workflow state
└── src/                         # Actual code
```

---

## Option 1: Manual Workflow with Conventions

### Phase 1: Research
Start a new Cursor chat:
```
/research EPIC-001

Explore the codebase to understand:
1. Current architecture for [feature area]
2. Existing patterns we should follow
3. Dependencies and constraints
4. Open questions

Output your findings to .specs/research/RES-001-findings.md
```

### Phase 2: Plan
New chat (fresh context):
```
/plan EPIC-001

Read: .specs/research/RES-001-findings.md

Create a detailed implementation plan:
1. Break into discrete tasks (max 4 hours each)
2. Define clear acceptance criteria per task
3. Identify dependencies between tasks
4. Note any risks or blockers

Output to: .specs/epics/EPIC-001-feature.md
Output tasks to: .specs/tasks/TASK-XXX-name.md (one per task)
```

### Phase 3: Implement (per task)
New chat per task (clean context):
```
/implement TASK-001

Read:
- .specs/epics/EPIC-001-feature.md (context)
- .specs/tasks/TASK-001-subtask.md (this task)

Implement ONLY this task.
Follow acceptance criteria exactly.
Do not scope creep into other tasks.

When done, update task status in .specs/tasks/TASK-001-subtask.md
```

---

## Option 2: Enforced via .cursorrules

```markdown
# .cursor/rules

## Workflow Enforcement

### Before ANY code changes:
1. Check if a spec exists in .specs/tasks/ for this work
2. If no spec exists, STOP and create one first
3. Reference the spec ID in your commit message

### Research Phase Rules:
- Output findings to .specs/research/
- Do NOT write code during research
- List all open questions at the end

### Planning Phase Rules:
- Read research findings first
- Break work into tasks < 4 hours each
- Each task gets its own .specs/tasks/TASK-XXX.md
- Include acceptance criteria in every task spec

### Implementation Phase Rules:
- Start by reading the task spec
- Implement ONLY what's in the spec
- If scope needs to change, update spec FIRST
- Mark task complete when acceptance criteria pass

### Spec File Format:
Every spec MUST include:
- ID (e.g., TASK-001)
- Title
- Status: [draft|ready|in-progress|review|done]
- Parent Epic (if applicable)
- Acceptance Criteria (checkboxes)
- Dependencies
- Notes/Context

### Commit Message Format:
[TASK-XXX] Brief description

Refs: .specs/tasks/TASK-XXX.md
```

---

## Option 3: Cursor Background Agents (Experimental)

Cursor's Composer supports "Background Agents" that can:
- Run in isolated contexts
- Execute tasks asynchronously
- Report back when done

### Workflow:
```
1. Main chat: Create spec, break into tasks
2. For each task: Spawn background agent
   - Agent gets clean context
   - Agent reads only its task spec
   - Agent implements and commits
3. Main chat: Review agent outputs
```

### Triggering sub-agents:
```
@background Implement TASK-001
Context: .specs/tasks/TASK-001-subtask.md
Constraints: Only modify files listed in spec
Report: Update task status when complete
```

---

## Option 4: External Orchestration Script

Build a wrapper that orchestrates Cursor CLI (if available) or uses the API:

```bash
#!/bin/bash
# cursor-workflow.sh

SPEC_DIR=".specs"
EPIC_ID=$1

# Phase 1: Research
echo "Starting research phase for $EPIC_ID..."
cursor-cli chat --system "You are in RESEARCH mode. Output to $SPEC_DIR/research/" \
  --prompt "Research the codebase for implementing $EPIC_ID" \
  --output "$SPEC_DIR/research/RES-$EPIC_ID.md"

# Phase 2: Plan (new context)
echo "Starting planning phase..."
cursor-cli chat --system "You are in PLANNING mode." \
  --context "$SPEC_DIR/research/RES-$EPIC_ID.md" \
  --prompt "Create implementation plan with tasks" \
  --output "$SPEC_DIR/epics/$EPIC_ID.md"

# Phase 3: Implement each task (parallel, clean contexts)
for task in $SPEC_DIR/tasks/TASK-$EPIC_ID-*.md; do
  echo "Implementing $task..."
  cursor-cli chat --system "You are in IMPLEMENTATION mode. ONLY implement this task." \
    --context "$task" \
    --prompt "Implement this task per spec" &
done
wait

echo "All tasks complete. Review outputs."
```

---

## Option 5: MCP Server (Most Integrated)

Build a custom MCP (Model Context Protocol) server that:
1. Manages workflow state
2. Exposes tools: `/research`, `/plan`, `/implement`, `/review`
3. Enforces phase transitions
4. Auto-saves specs to Git

```typescript
// Conceptual MCP server
const workflowTools = {
  research: async (epicId: string) => {
    // Set state to RESEARCH
    // Provide research-only context
    // Save output to .specs/research/
  },
  
  plan: async (epicId: string) => {
    // Require research complete
    // Set state to PLANNING
    // Create task files
  },
  
  implement: async (taskId: string) => {
    // Fresh context with ONLY task spec
    // Track progress
    // Update task status
  },
  
  review: async (taskId: string) => {
    // Compare implementation to spec
    // Generate review receipt
  }
};
```

---

## Git Integration

### Pre-commit hook (.git/hooks/pre-commit):
```bash
#!/bin/bash
# Ensure every code change has a spec reference

if ! grep -q "TASK-\|EPIC-" "$1"; then
  echo "ERROR: Commit must reference a spec (TASK-XXX or EPIC-XXX)"
  exit 1
fi
```

### Spec versioning:
- Specs are markdown, Git-tracked
- Changes to specs = PRs, reviewed
- History preserved forever
- Can diff spec vs implementation

---

## Recommended Approach for Your Team

### Start Simple (Option 1 + 2):
1. Create `.specs/` directory structure
2. Add `.cursorrules` with workflow enforcement
3. Train team on the 3-phase convention
4. Use commit hooks for compliance

### Scale Up (Add Option 3 or 4):
1. Leverage Cursor background agents for parallel tasks
2. Build simple orchestration scripts
3. Add CI checks for spec coverage

### Enterprise (Option 5):
1. Build custom MCP server
2. Integrate with your existing project management
3. Add metrics/dashboards for spec coverage

---

## Template Files

### Epic Spec Template (.specs/templates/epic.md):
```markdown
# EPIC-XXX: [Title]

## Status: draft | ready | in-progress | done

## Overview
[High-level description]

## Research Findings
[Link to .specs/research/RES-XXX.md]

## Tasks
- [ ] TASK-XXX-1: [Description]
- [ ] TASK-XXX-2: [Description]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Dependencies
- [List external dependencies]

## Risks
- [List risks and mitigations]

## Notes
[Additional context]
```

### Task Spec Template (.specs/templates/task.md):
```markdown
# TASK-XXX: [Title]

## Status: draft | ready | in-progress | review | done

## Parent Epic: EPIC-XXX

## Description
[What needs to be done]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Tests pass
- [ ] Linter clean

## Files to Modify
- src/path/to/file.ts
- src/path/to/other.ts

## Dependencies
- Requires TASK-YYY complete

## Implementation Notes
[Technical guidance]

## Review Checklist
- [ ] Meets acceptance criteria
- [ ] Code reviewed
- [ ] Tests added
- [ ] Docs updated
```

---

## Comparison: AWS Kiro vs This Approach

| Aspect | AWS Kiro | Cursor + Specs |
|--------|----------|----------------|
| Built-in workflow | ✅ Native | 🔧 Convention-based |
| Spec persistence | ✅ Auto | ✅ Git-tracked |
| Fresh context | ✅ Managed | 🔧 Manual new chat |
| Sub-agents | ✅ Native | ⚠️ Background agents (beta) |
| Team adoption | 🔧 Learn new tool | ✅ Uses existing Cursor |
| Customization | ❌ Limited | ✅ Full control |
| Open source | ❌ No | ✅ Your implementation |

---

## Next Steps

1. Create `.specs/` directory in a pilot project
2. Add `.cursorrules` with workflow rules
3. Train 2-3 developers on the workflow
4. Iterate based on feedback
5. Scale to full team

---

*Created: 2026-01-17*
*For: Karthik's Platform Engineering team*
