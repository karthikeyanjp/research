# Cursor Workflow: Options 4 & 5 Implementation

## Option 4: External Orchestration Script

### Concept
A script OUTSIDE Cursor that:
1. Manages workflow state
2. Invokes Cursor/Claude with specific prompts per phase
3. Captures outputs to spec files
4. Ensures fresh context per task

### The Key Insight
Cursor doesn't have a CLI like `claude -p`, but we have alternatives:
- **cursor-cli** (if available in your Cursor version)
- **Anthropic API directly** (via `claude` CLI or API calls)
- **Open Router / other API** for model access

### Implementation A: Using Claude CLI + Cursor for Editing

```bash
#!/bin/bash
# workflow.sh - Research/Plan/Implement orchestrator

set -e

PROJECT_ROOT=$(pwd)
SPECS_DIR="$PROJECT_ROOT/.specs"
WORKFLOW_STATE="$PROJECT_ROOT/.workflow/state.json"

# Initialize workflow state
init_workflow() {
    mkdir -p "$SPECS_DIR"/{research,epics,tasks,reviews}
    mkdir -p "$PROJECT_ROOT/.workflow"
    echo '{"phase": "init", "epic": null, "tasks": []}' > "$WORKFLOW_STATE"
}

# Phase 1: Research
research() {
    local epic_id=$1
    local description=$2
    
    echo "🔍 Starting research for $epic_id..."
    
    # Update state
    jq --arg epic "$epic_id" '.phase = "research" | .epic = $epic' "$WORKFLOW_STATE" > tmp.json && mv tmp.json "$WORKFLOW_STATE"
    
    # Build context: include relevant source files
    local context_files=$(find src -name "*.ts" -o -name "*.tsx" | head -20 | xargs cat 2>/dev/null || echo "")
    
    # Call Claude CLI for research (fresh process = fresh context)
    claude -p "You are researching a codebase for a new feature.

EPIC: $epic_id
DESCRIPTION: $description

CODEBASE CONTEXT:
$context_files

TASK:
1. Analyze the current architecture relevant to this feature
2. Identify existing patterns to follow
3. List dependencies and constraints
4. Note any risks or open questions

OUTPUT FORMAT:
Write a structured research document.
" --output-file "$SPECS_DIR/research/RES-$epic_id.md"

    echo "✅ Research complete: $SPECS_DIR/research/RES-$epic_id.md"
}

# Phase 2: Plan
plan() {
    local epic_id=$1
    
    echo "📋 Starting planning for $epic_id..."
    
    # Read research findings
    local research=$(cat "$SPECS_DIR/research/RES-$epic_id.md")
    
    # Update state
    jq '.phase = "planning"' "$WORKFLOW_STATE" > tmp.json && mv tmp.json "$WORKFLOW_STATE"
    
    # Call Claude CLI for planning (fresh process = fresh context)
    claude -p "You are creating an implementation plan.

EPIC: $epic_id

RESEARCH FINDINGS:
$research

TASK:
1. Create a high-level epic spec
2. Break into discrete tasks (max 4 hours each)
3. Each task needs clear acceptance criteria
4. Identify task dependencies

OUTPUT:
First, output the epic spec, then output each task spec separately.
Use markers: --- EPIC START ---, --- EPIC END ---, --- TASK START [id] ---, --- TASK END ---
" --output-file "/tmp/plan-output.md"

    # Parse output into separate files
    parse_plan_output "$epic_id" "/tmp/plan-output.md"
    
    echo "✅ Planning complete"
}

# Phase 3: Implement (per task)
implement_task() {
    local task_id=$1
    
    echo "🔨 Implementing $task_id..."
    
    # Read task spec
    local task_spec=$(cat "$SPECS_DIR/tasks/$task_id.md")
    
    # Read parent epic for context
    local epic_id=$(grep "Parent Epic:" "$SPECS_DIR/tasks/$task_id.md" | cut -d: -f2 | tr -d ' ')
    local epic_spec=$(cat "$SPECS_DIR/epics/$epic_id.md" 2>/dev/null || echo "")
    
    # Fresh Claude call for implementation
    claude -p "You are implementing a specific task.

TASK SPEC:
$task_spec

EPIC CONTEXT:
$epic_spec

RULES:
1. ONLY implement what's in the task spec
2. Do NOT scope creep into other tasks
3. Follow acceptance criteria exactly
4. Output the code changes as a unified diff

OUTPUT:
Provide implementation as a diff that can be applied with 'git apply'
" --output-file "/tmp/impl-$task_id.diff"

    # Apply the diff (or review first)
    echo "📄 Implementation diff: /tmp/impl-$task_id.diff"
    echo "Review and apply with: git apply /tmp/impl-$task_id.diff"
    
    # Update task status
    sed -i '' 's/Status: ready/Status: review/' "$SPECS_DIR/tasks/$task_id.md"
}

# Run all tasks in parallel (with fresh contexts)
implement_all() {
    local epic_id=$1
    
    for task_file in "$SPECS_DIR/tasks/TASK-$epic_id-"*.md; do
        local task_id=$(basename "$task_file" .md)
        implement_task "$task_id" &
    done
    wait
    
    echo "✅ All tasks implemented"
}

# Parse plan output into separate files
parse_plan_output() {
    local epic_id=$1
    local output_file=$2
    
    # Use awk/sed to split the output (simplified)
    # In practice, use a proper parser
    
    awk '/--- EPIC START ---/,/--- EPIC END ---/' "$output_file" | \
        grep -v "^---" > "$SPECS_DIR/epics/$epic_id.md"
    
    # Extract each task
    local task_num=1
    while IFS= read -r task_block; do
        echo "$task_block" > "$SPECS_DIR/tasks/TASK-$epic_id-$(printf "%03d" $task_num).md"
        ((task_num++))
    done < <(awk '/--- TASK START/,/--- TASK END ---/' "$output_file" | grep -v "^---")
}

# Main CLI
case "$1" in
    init)
        init_workflow
        ;;
    research)
        research "$2" "$3"
        ;;
    plan)
        plan "$2"
        ;;
    implement)
        implement_task "$2"
        ;;
    implement-all)
        implement_all "$2"
        ;;
    *)
        echo "Usage: workflow.sh {init|research|plan|implement|implement-all} [args]"
        exit 1
        ;;
esac
```

### Implementation B: Node.js Version (More Robust)

```typescript
// workflow.ts
import Anthropic from "@anthropic-ai/sdk";
import * as fs from "fs/promises";
import * as path from "path";

const anthropic = new Anthropic();

interface WorkflowState {
  phase: "init" | "research" | "planning" | "implementing" | "review";
  epicId: string | null;
  tasks: TaskState[];
}

interface TaskState {
  id: string;
  status: "pending" | "in-progress" | "review" | "done";
}

class SpecDrivenWorkflow {
  private specsDir: string;
  private state: WorkflowState;

  constructor(projectRoot: string) {
    this.specsDir = path.join(projectRoot, ".specs");
    this.state = { phase: "init", epicId: null, tasks: [] };
  }

  async init() {
    await fs.mkdir(path.join(this.specsDir, "research"), { recursive: true });
    await fs.mkdir(path.join(this.specsDir, "epics"), { recursive: true });
    await fs.mkdir(path.join(this.specsDir, "tasks"), { recursive: true });
    await fs.mkdir(path.join(this.specsDir, "reviews"), { recursive: true });
    console.log("✅ Workflow initialized");
  }

  async research(epicId: string, description: string) {
    console.log(`🔍 Researching ${epicId}...`);
    
    // Gather codebase context
    const context = await this.gatherContext();

    // Fresh API call = fresh context
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      messages: [
        {
          role: "user",
          content: `You are researching a codebase for a new feature.

EPIC: ${epicId}
DESCRIPTION: ${description}

CODEBASE CONTEXT:
${context}

Analyze and output a structured research document covering:
1. Current architecture
2. Patterns to follow
3. Dependencies/constraints
4. Risks and open questions`,
        },
      ],
    });

    const output = response.content[0].type === "text" ? response.content[0].text : "";
    await fs.writeFile(
      path.join(this.specsDir, "research", `RES-${epicId}.md`),
      output
    );

    console.log(`✅ Research saved`);
    return output;
  }

  async plan(epicId: string) {
    console.log(`📋 Planning ${epicId}...`);

    // Read research (passed as context, but NEW API call = fresh context window)
    const research = await fs.readFile(
      path.join(this.specsDir, "research", `RES-${epicId}.md`),
      "utf-8"
    );

    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 8192,
      messages: [
        {
          role: "user",
          content: `You are creating an implementation plan.

RESEARCH FINDINGS:
${research}

Create:
1. Epic spec with overview
2. Task breakdown (max 4 hours each)
3. Acceptance criteria per task

Output as JSON:
{
  "epic": { "id": "${epicId}", "title": "...", "overview": "...", "acceptance_criteria": [...] },
  "tasks": [
    { "id": "TASK-${epicId}-001", "title": "...", "description": "...", "acceptance_criteria": [...], "files_to_modify": [...] }
  ]
}`,
        },
      ],
    });

    const output = response.content[0].type === "text" ? response.content[0].text : "";
    const plan = JSON.parse(output);

    // Write epic spec
    await fs.writeFile(
      path.join(this.specsDir, "epics", `${epicId}.md`),
      this.formatEpicMarkdown(plan.epic)
    );

    // Write each task spec
    for (const task of plan.tasks) {
      await fs.writeFile(
        path.join(this.specsDir, "tasks", `${task.id}.md`),
        this.formatTaskMarkdown(task, epicId)
      );
      this.state.tasks.push({ id: task.id, status: "pending" });
    }

    console.log(`✅ Created ${plan.tasks.length} tasks`);
    return plan;
  }

  async implementTask(taskId: string) {
    console.log(`🔨 Implementing ${taskId}...`);

    // Read ONLY this task's spec (minimal context)
    const taskSpec = await fs.readFile(
      path.join(this.specsDir, "tasks", `${taskId}.md`),
      "utf-8"
    );

    // Fresh API call with minimal context
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 8192,
      messages: [
        {
          role: "user",
          content: `Implement this task. Output ONLY the code changes as a unified diff.

TASK SPEC:
${taskSpec}

RULES:
- ONLY implement what's in the spec
- NO scope creep
- Follow acceptance criteria exactly`,
        },
      ],
    });

    const diff = response.content[0].type === "text" ? response.content[0].text : "";
    await fs.writeFile(`/tmp/impl-${taskId}.diff`, diff);

    console.log(`✅ Diff saved: /tmp/impl-${taskId}.diff`);
    return diff;
  }

  async implementAllParallel(epicId: string) {
    const tasks = this.state.tasks.filter((t) => t.id.includes(epicId));
    
    // Parallel execution = each gets fresh context automatically
    await Promise.all(tasks.map((t) => this.implementTask(t.id)));
    
    console.log(`✅ All tasks implemented`);
  }

  private async gatherContext(): Promise<string> {
    // Simplified: read key files
    // In practice: use tree-sitter, embeddings, or smart selection
    return "// ... codebase context ...";
  }

  private formatEpicMarkdown(epic: any): string {
    return `# ${epic.id}: ${epic.title}

## Status: ready

## Overview
${epic.overview}

## Acceptance Criteria
${epic.acceptance_criteria.map((c: string) => `- [ ] ${c}`).join("\n")}
`;
  }

  private formatTaskMarkdown(task: any, epicId: string): string {
    return `# ${task.id}: ${task.title}

## Status: pending
## Parent Epic: ${epicId}

## Description
${task.description}

## Acceptance Criteria
${task.acceptance_criteria.map((c: string) => `- [ ] ${c}`).join("\n")}

## Files to Modify
${task.files_to_modify.map((f: string) => `- ${f}`).join("\n")}
`;
  }
}

// CLI
const workflow = new SpecDrivenWorkflow(process.cwd());
const [, , command, ...args] = process.argv;

(async () => {
  switch (command) {
    case "init":
      await workflow.init();
      break;
    case "research":
      await workflow.research(args[0], args[1]);
      break;
    case "plan":
      await workflow.plan(args[0]);
      break;
    case "implement":
      await workflow.implementTask(args[0]);
      break;
    case "implement-all":
      await workflow.implementAllParallel(args[0]);
      break;
    default:
      console.log("Usage: npx ts-node workflow.ts {init|research|plan|implement|implement-all}");
  }
})();
```

---

## Option 5: Custom MCP Server

### Concept
MCP (Model Context Protocol) lets you extend Cursor/Claude with custom tools. The AI can call your tools, and your tools control the workflow.

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│                      CURSOR                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Claude (in Cursor)                              │    │
│  │  "I need to implement a feature..."              │    │
│  │                                                  │    │
│  │  → calls: workflow.research("EPIC-001", "...")  │    │
│  │  → calls: workflow.plan("EPIC-001")             │    │
│  │  → calls: workflow.implement("TASK-001")        │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         │ MCP Protocol                   │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │  YOUR MCP SERVER                                 │    │
│  │  - Manages workflow state                        │    │
│  │  - Enforces phase transitions                    │    │
│  │  - Persists specs to Git                         │    │
│  │  - Tracks receipts                               │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Implementation

```typescript
// mcp-workflow-server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs/promises";
import * as path from "path";
import { execSync } from "child_process";

// Workflow state
interface WorkflowState {
  currentPhase: "idle" | "research" | "planning" | "implementing" | "review";
  currentEpic: string | null;
  completedPhases: Record<string, string[]>; // epicId -> completed phases
  taskStatus: Record<string, "pending" | "in-progress" | "review" | "done">;
}

let state: WorkflowState = {
  currentPhase: "idle",
  currentEpic: null,
  completedPhases: {},
  taskStatus: {},
};

const SPECS_DIR = path.join(process.cwd(), ".specs");

// Create MCP server
const server = new Server(
  { name: "workflow-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Define available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "workflow_status",
      description: "Get current workflow status and phase",
      inputSchema: { type: "object", properties: {} },
    },
    {
      name: "workflow_research",
      description:
        "Start research phase for an epic. MUST be called before planning. Outputs research findings.",
      inputSchema: {
        type: "object",
        properties: {
          epicId: { type: "string", description: "Epic identifier (e.g., EPIC-001)" },
          description: { type: "string", description: "What to research" },
        },
        required: ["epicId", "description"],
      },
    },
    {
      name: "workflow_plan",
      description:
        "Create implementation plan. REQUIRES research phase complete. Outputs epic and task specs.",
      inputSchema: {
        type: "object",
        properties: {
          epicId: { type: "string", description: "Epic identifier" },
        },
        required: ["epicId"],
      },
    },
    {
      name: "workflow_get_task",
      description: "Get a specific task spec for implementation",
      inputSchema: {
        type: "object",
        properties: {
          taskId: { type: "string", description: "Task identifier" },
        },
        required: ["taskId"],
      },
    },
    {
      name: "workflow_complete_task",
      description: "Mark a task as complete and save implementation notes",
      inputSchema: {
        type: "object",
        properties: {
          taskId: { type: "string", description: "Task identifier" },
          notes: { type: "string", description: "Implementation notes" },
        },
        required: ["taskId"],
      },
    },
    {
      name: "workflow_save_spec",
      description: "Save a spec file (research, epic, or task)",
      inputSchema: {
        type: "object",
        properties: {
          type: { type: "string", enum: ["research", "epic", "task"] },
          id: { type: "string" },
          content: { type: "string" },
        },
        required: ["type", "id", "content"],
      },
    },
  ],
}));

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "workflow_status":
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(state, null, 2),
          },
        ],
      };

    case "workflow_research":
      // Enforce: can't research if already in another phase
      if (state.currentPhase !== "idle" && state.currentEpic !== args.epicId) {
        return {
          content: [
            {
              type: "text",
              text: `ERROR: Already working on ${state.currentEpic}. Complete it first or call workflow_reset.`,
            },
          ],
          isError: true,
        };
      }

      state.currentPhase = "research";
      state.currentEpic = args.epicId as string;

      // Create research file placeholder
      await fs.mkdir(path.join(SPECS_DIR, "research"), { recursive: true });

      return {
        content: [
          {
            type: "text",
            text: `Research phase started for ${args.epicId}.
            
Your task: Research the codebase for "${args.description}"

When done, call workflow_save_spec with type="research" to save your findings.
Then call workflow_plan to proceed to planning.`,
          },
        ],
      };

    case "workflow_plan":
      // Enforce: must complete research first
      const researchFile = path.join(SPECS_DIR, "research", `RES-${args.epicId}.md`);
      try {
        await fs.access(researchFile);
      } catch {
        return {
          content: [
            {
              type: "text",
              text: `ERROR: Research not complete for ${args.epicId}. Call workflow_research first.`,
            },
          ],
          isError: true,
        };
      }

      state.currentPhase = "planning";
      const research = await fs.readFile(researchFile, "utf-8");

      return {
        content: [
          {
            type: "text",
            text: `Planning phase started for ${args.epicId}.

RESEARCH FINDINGS:
${research}

Your task: Create an implementation plan with:
1. Epic overview
2. Task breakdown (use workflow_save_spec for each)
3. Acceptance criteria

Save the epic with workflow_save_spec(type="epic", id="${args.epicId}", content=...)
Save each task with workflow_save_spec(type="task", id="TASK-${args.epicId}-001", content=...)`,
          },
        ],
      };

    case "workflow_get_task":
      const taskFile = path.join(SPECS_DIR, "tasks", `${args.taskId}.md`);
      try {
        const taskContent = await fs.readFile(taskFile, "utf-8");
        state.taskStatus[args.taskId as string] = "in-progress";
        return {
          content: [{ type: "text", text: taskContent }],
        };
      } catch {
        return {
          content: [{ type: "text", text: `ERROR: Task ${args.taskId} not found.` }],
          isError: true,
        };
      }

    case "workflow_complete_task":
      state.taskStatus[args.taskId as string] = "done";
      
      // Update task file
      const taskPath = path.join(SPECS_DIR, "tasks", `${args.taskId}.md`);
      let content = await fs.readFile(taskPath, "utf-8");
      content = content.replace("Status: in-progress", "Status: done");
      content += `\n\n## Implementation Notes\n${args.notes}\n`;
      await fs.writeFile(taskPath, content);

      // Git commit the spec update
      try {
        execSync(`git add ${taskPath} && git commit -m "[${args.taskId}] Task completed"`, {
          cwd: process.cwd(),
        });
      } catch {
        // Git commit failed, that's ok
      }

      return {
        content: [{ type: "text", text: `Task ${args.taskId} marked complete.` }],
      };

    case "workflow_save_spec":
      const { type, id, content: specContent } = args as {
        type: string;
        id: string;
        content: string;
      };
      
      const dir = type === "research" ? "research" : type === "epic" ? "epics" : "tasks";
      const filename = type === "research" ? `RES-${id}.md` : `${id}.md`;
      const filePath = path.join(SPECS_DIR, dir, filename);

      await fs.mkdir(path.join(SPECS_DIR, dir), { recursive: true });
      await fs.writeFile(filePath, specContent);

      // Track task status
      if (type === "task") {
        state.taskStatus[id] = "pending";
      }

      // Git add the spec
      try {
        execSync(`git add ${filePath}`, { cwd: process.cwd() });
      } catch {
        // Git add failed, that's ok
      }

      return {
        content: [{ type: "text", text: `Saved ${type} spec: ${filePath}` }],
      };

    default:
      return {
        content: [{ type: "text", text: `Unknown tool: ${name}` }],
        isError: true,
      };
  }
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Workflow MCP server running");
}

main().catch(console.error);
```

### Cursor MCP Config

Add to your Cursor settings (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "workflow": {
      "command": "npx",
      "args": ["ts-node", "/path/to/mcp-workflow-server.ts"],
      "env": {}
    }
  }
}
```

### Usage in Cursor

Once configured, Claude in Cursor can call your tools:

```
User: "I need to implement user authentication"

Claude: Let me start the workflow properly.
[calls workflow_research("EPIC-001", "user authentication")]

"I'll research the codebase first..."
[does research, then calls workflow_save_spec(type="research", ...)]

"Now let me create a plan..."
[calls workflow_plan("EPIC-001")]

"Here are the tasks..."
[calls workflow_save_spec(type="task", ...) for each task]

"Ready to implement TASK-001..."
[calls workflow_get_task("TASK-001")]
[implements]
[calls workflow_complete_task("TASK-001", "implemented auth middleware")]
```

---

## Comparison

| Aspect | Option 4 (Script) | Option 5 (MCP) |
|--------|-------------------|----------------|
| **Integration** | External to Cursor | Inside Cursor |
| **Fresh context** | ✅ New process per phase | ⚠️ Same session, but tools enforce phases |
| **User experience** | CLI commands | Natural conversation |
| **Enforcement** | Hard (script controls) | Soft (AI can ignore) |
| **Complexity** | Medium | High |
| **Team adoption** | Learn new CLI | Invisible to users |

---

## Recommendation

**Start with Option 4** (script) because:
1. True fresh context per phase
2. Easier to debug
3. Can integrate with CI/CD
4. Doesn't require MCP knowledge

**Graduate to Option 5** when:
1. Team wants seamless UX
2. You need tighter integration
3. You want AI to self-orchestrate

---

*Created: 2026-01-17*
