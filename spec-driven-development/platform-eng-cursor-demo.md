# Platform Engineering: Structured Development Practices with Cursor

**Demo Date:** Thursday, January 22, 2026
**Audience:** 100+ Developers
**Presenter:** Karthik

---

## 🎯 Core Thesis

> Moving from "AI as autocomplete" to "AI as orchestrated engineering partner"

---

## Key Vocabularies

- **Orchestrating Intelligence** — Directing AI capabilities through structured processes, not ad-hoc prompts
- **Institutional Knowledge** — Codified standards, patterns, and decisions that survive individual contributors
- **Mutable Living Specs** - One Spec per feature

---

## The Four Stages

### Stage 1: Foundational Discipline 🏗️

**Theme:** _"Before you can orchestrate AI, you need engineering rigor"_

**Key Concepts:**

- Clean code principles still matter
- Git hygiene, PR discipline, testing culture
- AI amplifies habits — good AND bad
- "Garbage in, garbage out" applies to prompts too

**Demo Ideas:**

- Show AI producing bad code from vague prompts
- Same task with structured prompt → better output
- Lesson: Discipline isn't replaced, it's prerequisite

**Vocabulary:**

- Prompt hygiene
- Context discipline
- Engineering fundamentals

---

### Stage 2: Institutional Knowledge Embedding through Rules 📚

**Theme:** _"Codify tribal knowledge so AI can learn it"_

**Key Concepts:**

- `.cursorrules` / `CLAUDE.md` / `AGENTS.md` as institutional memory
- Rules files = onboarding docs for AI (and humans!)
- Capture: naming conventions, arch patterns, forbidden anti-patterns
- AI becomes "senior engineer who read all the docs"

**Demo Ideas:**

- Project without rules → AI makes random choices
- Same project with `.cursorrules` → consistent, team-aligned output
- Show a real rules file with your team's standards

**Vocabulary:**

- Institutional knowledge embedding
- Rules as documentation
- Codified standards vs tribal knowledge
- AI onboarding

**Example `.cursorrules` structure:**

```
# Project Context
# Architecture Decisions
# Naming Conventions
# Testing Standards
# Forbidden Patterns
# Error Handling Philosophy
```

---

### Stage 3: Spec-Driven Philosophy — Research → Plan → Implement 📋

**Theme:** _"Think first, code second — with AI"_

**Key Concepts:**

- Resist "code immediately" urge
- Research phase: AI explores codebase, asks questions
- Plan phase: Structured spec before any code
- Implement phase: Execute against reviewed plan
- Plans are reviewable artifacts (not lost in chat)

**Demo Ideas:**

- Task without planning → messy, incomplete solution
- Same task with spec → coherent, complete implementation
- Show the spec document as a PR artifact

**Vocabulary:**

- Spec-driven development
- Research → Plan → Implement
- Planning as artifact
- Reviewable AI work

**Flow:**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  RESEARCH   │ ──▶ │    PLAN     │ ──▶ │  IMPLEMENT  │
│             │     │             │     │             │
│ • Explore   │     │ • Spec doc  │     │ • Execute   │
│ • Questions │     │ • Reviewed  │     │ • Test      │
│ • Context   │     │ • Approved  │     │ • Verify    │
└─────────────┘     └─────────────┘     └─────────────┘
```

---

### Stage 4: Autonomous Workflows 🤖

**Theme:** _"Trust but verify — at scale"_

**Key Concepts:**

- AI can work autonomously with guardrails
- Fresh context per task (prevent drift)
- Multi-model review gates (AI reviews AI)
- Receipt-based gating (no proof = no progress)
- Human reviews outcomes, not keystrokes

**Demo Ideas:**

- Show autonomous loop concept (Ralph mode / background agents)
- Quality gates that block bad work
- Morning review of overnight AI work
- Treat AI as "untrusted but capable contractor"

**Vocabulary:**

- Orchestrating intelligence
- Autonomous workflows
- Quality gates
- Cross-model review
- Receipt-based gating
- Re-anchoring (fresh context)

**Architecture:**

```
┌────────────────────────────────────────────┐
│  ORCHESTRATION LAYER (human-defined)       │
│  ┌──────────────────────────────────────┐  │
│  │  while tasks remain:                 │  │
│  │    1. Fresh context (re-anchor)      │  │
│  │    2. Execute task                   │  │
│  │    3. Cross-model review             │  │
│  │    4. Gate: SHIP or REWORK           │  │
│  │    5. Log receipt                    │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

---

## The Journey (Summary Slide)

| Stage                      | Mindset Shift             |
| -------------------------- | ------------------------- |
| 1. Foundational Discipline | "AI needs my rigor"       |
| 2. Institutional Knowledge | "AI learns our way"       |
| 3. Spec-Driven Philosophy  | "AI plans before coding"  |
| 4. Autonomous Workflows    | "AI works while I review" |

---

## Key Takeaways for Audience

1. **AI amplifies your engineering culture** — make sure it's worth amplifying
2. **Rules files are the new onboarding docs** — write them for AI AND humans
3. **Planning is not overhead, it's leverage** — specs are reviewable artifacts
4. **Autonomy requires guardrails** — quality gates, not blind trust
5. **This is platform engineering** — you're building the system that builds software

---

## Demo Flow (Draft)

- [ ] **Intro** (5 min): The evolution from autocomplete to orchestration
- [ ] **Stage 1** (10 min): Show discipline gap, live example
- [ ] **Stage 2** (10 min): `.cursorrules` before/after demo
- [ ] **Stage 3** (15 min): Research → Plan → Implement live
- [ ] **Stage 4** (10 min): Autonomous workflow concept + quality gates
- [ ] **Wrap-up** (5 min): The journey, key takeaways
- [ ] **Q&A** (10 min)

---

## Resources to Reference

- Flow-Next by @gmickel (autonomous workflows, quality gates)
- Anthropic's agent guidelines (re-anchoring, drift prevention)
- Your team's actual `.cursorrules` or `CLAUDE.md`

---

_Last updated: 2026-01-17_
