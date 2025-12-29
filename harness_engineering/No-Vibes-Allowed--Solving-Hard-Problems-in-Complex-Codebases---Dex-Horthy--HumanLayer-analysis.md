# No Vibes Allowed: Solving Hard Problems in Complex Codebases - Analysis

![Video](https://www.youtube.com/watch?v=rmvDxxNubIg)
---

## Key Insights

- **Context window management = core bottleneck** for coding agents
- **LLMs are stateless** - only current context influences next token, nothing else
- **The "dumb zone"** - performance degrades past ~40% context window usage
- **Compaction = key strategy** - compress context frequently to stay in "smart zone"
- **Sub-agents for context control**, not role anthropomorphization
- **Mental alignment > code review** - plans keep teams aligned as AI ships 2-3x more code
- **Semantic diffusion killed "spec-driven dev"** - term became meaningless
- **Don't outsource thinking** - AI amplifies thinking done/not done, can't replace it
- Bad research line → hundreds of bad code lines

## Lessons

### Context Engineering
- Start new context window when seeing "You're absolutely right" (AI apologizing repeatedly)
- Stay under 40% context usage for complex tasks
- Use intentional compaction even when on track
- Optimize for: correctness, completeness, size, trajectory
- Worst inputs: incorrect info > missing info > noise
- Avoid trajectory where AI learns to fail (human yells → AI fails → repeat)

### Workflow Practices
- **Research-Plan-Implement (RPI)** workflow:
  - Research: understand system, find files, stay objective
  - Plan: exact steps, file names, line numbers, code snippets, test steps
  - Implement: execute with minimal context
- Review plans while they're created, not after
- Include code snippets in plans for reliability
- Progressive disclosure: context files at repo root + subdirectories
- On-demand compressed context > static documentation (less lies)

### Team Dynamics
- Cultural change must come from top
- Staff engineers need to adopt or rift grows
- Junior/mid engineers use AI to fill gaps but create slop
- Senior engineers then hate AI from cleaning slop
- Share plan + AMP thread on PRs for reviewer journey
- Balance plan length: reliability vs readability

### Tool Usage
- Pick one tool, get reps - avoid min-maxing across tools
- Sometimes overkill: button color change = just tell agent
- Scale approach to problem complexity
- Will get wrong repeatedly - takes practice
- MCPs dumping JSON/UUIDs into context = death

## Examples

### Success Cases
- **Boundary ML Rust codebase** (300k lines)
  - One-shot fix using research-plan workflow
  - CTO merged PR, didn't realize it was demo
- **BAML contribution** (7 hours, Saturday)
  - Shipped 35k lines of code
  - Estimated 1-2 weeks work
  - One PR merged week later

### Failure Case
- **Parquet Java** - removing Hadoop dependencies
  - Plans/research insufficient
  - Threw out, went to whiteboard
  - Needed human thinking to map foot guns

### Real Usage
- Team of 3, 8 weeks to rewire workflow
- 2-3x throughput increase
- "Never going back"

## Summary Points

### Core Framework
- Context window has ~168k tokens (Claude)
- Reserve space for output/compaction
- Dumb zone starts ~40% usage
- Sub-agents return succinct messages to parent
- Frequent intentional compaction > occasional compaction

### Anti-Patterns
- Naive: ask → wrong → resteer → repeat → cry/give up
- Too many MCPs = always in dumb zone
- Frontend/backend/QA sub-agents (role-based)
- Not reading plans before execution
- Static onboarding docs (get stale, contain lies)
- Batching completions vs immediate marking

### Practical Steps
1. Research: how system works, relevant files
2. Plan: exact steps, snippets, tests
3. Implement: low context execution
4. Review: plans provide mental alignment
5. Repeat: keep context tight

### Philosophy
- No perfect prompt/silver bullet
- "Harness engineering" = how you integrate with tools
- Future: 99% AI-generated code
- Hardest part: adapting team/SDLC, not the coding
- Complexity ceiling ↑ as compaction effort ↑

### Tooling Mentioned
- Claude Code (primary example)
- Cursor, Codex (alternatives)
- GitHub for context
- MCPs (be careful with)

### Next Evolution
- Coding agents → commodity
- Real challenge: team workflow adaptation at 99% AI code
- Need agentic IDE for teams
- Cultural change hardest part
