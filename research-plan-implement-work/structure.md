# RPI Flow Context Structure

**RPI** = Research → Plan → Implement workflow
**Flow Context** = The persistent, file-based state that provides AI agents with structured project knowledge across sessions.

---

## Directory Structure

```
.
└── .flow/                              # Flow Context root (hidden, keeps repo clean)
    │
    ├── research/                       #    Research findings & explorations
    │   │                               #    Created BEFORE planning a feature
    │   │                               #    Documents codebase analysis, patterns, constraints
    │   ├── RES-001-auth-system.md      # Research doc for auth system feature
    │   └── RES-002-order-retrieval.md  # Research doc for order retrieval feature
    │
    ├── features/                       #    Feature specifications (the "Plan" phase)
    │   │                               #    One folder per feature, contains full spec breakdown
    │   │
    │   ├── auth-system/                # Feature: Authentication System
    │   │   ├── requirements.md         #   - What: functional & non-functional requirements
    │   │   ├── design.md               #   - How: technical design, architecture decisions
    │   │   └── tasks.md                #   - Work: atomic task breakdown
    │   │
    │   └── order-retrieval/            # Feature: Order Retrieval
    │       ├── requirements.md         #   - What: functional & non-functional requirements
    │       ├── design.md               #   - How: technical design, architecture decisions
    │       └── tasks.md                #   - Work: atomic task breakdown (~4hrs each)
    │
    ├── changes/                        #    Change records (the "Implement" phase)
    │   │                               #    Each file = one atomic change/iteration
    │   │                               #    Links back to feature, tracks what was actually done
    │   ├── CHG-001-auth-system-init.md           # Initial auth system implementation
    │   ├── CHG-002-auth-system-m2m-support.md    # Added M2M (machine-to-machine) support
    │   ├── CHG-003-order-retrieval-init.md       # Initial order retrieval implementation
    │   └── CHG-004-order-retrieval-refill.md     # Added refill order functionality
    │
    └── memory/                         #    Project Memory (persistent baseline knowledge)
        │                               #    Rarely changes, provides foundational context
        │                               #    AI reads this first to understand the project
        ├── 001-overview.md             # Non-technical product overview (the "why")
        ├── conventions.md              # Coding standards, naming conventions, patterns
        └── decisions.md                # Architecture Decision Records (ADRs)
```

---

## Naming Conventions

| Prefix    | Folder      | Purpose                                                              |
| --------- | ----------- | -------------------------------------------------------------------- |
| `RES-###` | `research/` | Research documents (numbered for ordering)                           |
| `CHG-###` | `changes/`  | Change records (numbered chronologically)                            |
| `###-`    | `memory/`   | Memory docs (numbered for read order, e.g., `001-overview.md` first) |

---

## Workflow: Research → Plan → Implement

```
1. RESEARCH   →  .flow/research/RES-XXX-feature.md
   ↓              (explore codebase, document findings)

2. PLAN       →  .flow/features/feature-name/
   ↓              (requirements.md → design.md → tasks.md)

3. IMPLEMENT  →  .flow/changes/CHG-XXX-feature-description.md
                  (one change record per implementation iteration)
```

---

## Key Principles

1. **File I/O is state** — `.flow/` is the source of truth, not conversation history
2. **Fresh context, always** — AI re-reads specs each session (re-anchoring)
3. **Plan before code** — No implementation without specs in `features/`
4. **Memory is baseline** — `memory/` provides stable project context that rarely changes
5. **Changes are receipts** — `changes/` documents what was actually implemented

---

_This is GitOps for AI agents._ 🚀
