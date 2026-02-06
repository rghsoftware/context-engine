# Architecture

Context Engine is a Claude Code plugin that adds structured planning, decision tracking, drift prevention, and per-stack quality checks to multi-stack projects. It ships as a marketplace plugin providing skills, agents, and a project init command.

This document covers how the system works (for users) and how to extend it (for contributors).

## Plugin layout

```
context-engine/                    # Plugin root
├── .claude-plugin/
│   └── marketplace.json           # Plugin marketplace metadata
├── commands/
│   └── init.md                    # /context-engine:init command
├── skills/
│   ├── plan-spec/SKILL.md         # Phase 1: Feature spec
│   ├── plan-research/SKILL.md     # Phase 2: Technical research
│   ├── plan-steps/SKILL.md        # Phase 3: Task breakdown
│   ├── plan-quick/SKILL.md        # Lightweight quick plans
│   ├── impl-start/SKILL.md        # Phase 4: Implementation
│   ├── impl-commit/SKILL.md       # Commit workflow
│   ├── adr-create/SKILL.md        # ADR creation + contradiction detection
│   ├── adr-review/SKILL.md        # ADR consistency + drift review
│   ├── backlog-add/SKILL.md       # Add ideas/bugs to backlog
│   ├── backlog-prioritize/SKILL.md# Prioritize backlog items
│   └── project-init/SKILL.md      # Templates library for init
├── agents/
│   ├── check-task-completion.md   # Milestone validation
│   ├── check-adr-consistency.md   # ADR graph validation
│   ├── qa-python.md               # Python/FastAPI quality
│   ├── qa-vue.md                  # Vue/TypeScript quality
│   ├── qa-rust.md                 # Rust/Tauri quality
│   ├── qa-embedded.md             # Embedded C/C++ quality
│   └── qa-kotlin.md               # Kotlin/KMP quality
└── hooks/                         # Empty; per-project hooks created by init
```

## Per-project artifacts

Running `/context-engine:init` creates these in the target project:

```
project-root/
├── .claude/
│   ├── rules/
│   │   ├── shared-conventions.md     # Git, naming, error messages
│   │   ├── python-fastapi.md         # Path-scoped to detected Python dirs
│   │   ├── vue-typescript.md         # Path-scoped to detected Vue dirs
│   │   └── ...                       # One per detected stack
│   ├── hooks/
│   │   └── post-edit-lint.sh         # Auto-format on save (stack-aware)
│   └── settings.json                 # PostToolUse hook config
├── Context/
│   ├── Features/
│   │   ├── 001-FeatureName/
│   │   │   ├── Spec.md               # Requirements + testable assertions
│   │   │   ├── Tech.md               # Architecture + decisions
│   │   │   └── Steps.md              # Numbered tasks + milestones
│   │   └── 002-QuickTask.md          # Quick plans are single files
│   ├── Decisions/
│   │   ├── 0001-decision-title.md    # ADRs
│   │   └── ...
│   └── Backlog/
│       ├── Ideas.md
│       └── Bugs.md
└── CLAUDE.md                         # Project overview
```

The plugin is global; these artifacts are project-local and committed to the project's repo.

## Core concepts

### 4-phase planning

Features go through four phases, each producing a distinct artifact:

| Phase | Skill | Artifact | Purpose |
|-------|-------|----------|---------|
| 1. Spec | `plan-spec` | `Spec.md` | Requirements, user stories, testable assertions |
| 2. Research | `plan-research` | `Tech.md` | Architecture decisions, stack details, risk assessment |
| 3. Steps | `plan-steps` | `Steps.md` | Numbered tasks (S001-S999) with milestones |
| 4. Implement | `impl-start` | Source code | Build with checkpoint validation at milestones |

Small tasks skip the full workflow and use `plan-quick` to produce a single markdown file.

Each phase reads the outputs of prior phases. `plan-research` reads `Spec.md` and existing ADRs. `plan-steps` reads both and generates paired test tasks (S###-T) and doc tasks (S###-D) alongside implementation tasks. `impl-start` loads all three and validates at each milestone marker.

### Architecture Decision Records (ADRs)

ADRs live in `Context/Decisions/` as numbered markdown files. They capture *why* decisions were made, especially deviations from the original spec.

Key properties:

- **Forward-only**: Old ADRs are never modified. New ADRs supersede them.
- **Mandatory compatibility section**: Every new ADR must acknowledge overlapping accepted ADRs and declare either "compatible because [reason]" or "supersedes ADR-XXXX."
- **Stack-aware overlap detection**: The `adr-create` skill defines "same area" per stack (e.g., Python: same module/endpoint/model; Rust: same crate/command namespace; Embedded: same peripheral/ISR vector).

See `skills/adr-create/SKILL.md` for the full ADR template and overlap detection rules.

### Drift prevention

Drift between spec and implementation is detected at three structural points:

1. **Testable assertions in Spec.md** — Concrete, verifiable statements (e.g., "API returns 429 after 100 req/min per user"). Each has an ID, verification method, and active/superseded status. See `skills/plan-spec/SKILL.md` for the assertions table format.

2. **Milestone checkpoints in Steps.md** — Tasks are grouped by milestones (`MILESTONE` markers). At each milestone, `impl-start` triggers the `check-task-completion` agent, which runs stub detection, test/doc task verification, and surfaces relevant assertions for manual alignment review.

3. **Periodic full review via `adr-review`** — On-demand project-level check that cross-references all assertions against all ADRs to find coverage gaps, stale specs (>30% assertions superseded), and undocumented deviations.

The system does not attempt automated semantic comparison of code against spec prose. It forces structured review at defined intervals.

### Test and doc task generation

`plan-steps` automatically pairs implementation tasks with test and documentation tasks:

- **S###-T** (test tasks): Generated for tasks that create or modify functional code. Placed immediately after their parent task. Include specific scenarios derived from spec assertions and stack conventions.
- **S###-D** (doc tasks): Generated selectively when a task touches public APIs, changes user-facing behavior, or introduces patterns. Placed at milestone boundaries.

The default is "planned unless removed" — developers delete unnecessary paired tasks rather than remembering to add them.

### Quality agents

Five stack-specific QA agents run the relevant linters, type checkers, and pattern checks:

| Agent | Tools | Key checks |
|-------|-------|------------|
| `qa-python` | ruff, mypy, bandit | Type safety, security scanning, FastAPI patterns |
| `qa-vue` | eslint, prettier, vue-tsc | Composition API patterns, accessibility |
| `qa-rust` | clippy, rustfmt, cargo audit | Unsafe blocks, unwrap usage, Tauri security |
| `qa-embedded` | cppcheck, clang-tidy, clang-format | Memory safety, ISR safety (ESP32/RP2350), FreeRTOS |
| `qa-kotlin` | detekt, ktlint | Coroutine safety, Compose patterns, KMP expect/actual |

All QA agents are read-only (Read, Grep, Glob, Bash). They report findings but never modify files.

## Data flow

This shows which artifacts each skill/agent reads and writes:

```
                    ┌─────────────────────────────────┐
                    │         CLAUDE.md + Rules        │
                    │    (project context, always read) │
                    └───────────────┬──────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
   ┌─────────┐              ┌─────────────┐              ┌───────────┐
   │plan-spec│              │  adr-create │              │ plan-quick│
   │→ Spec.md│              │→ ADR file   │              │→ Quick.md │
   └────┬────┘              │→ Spec.md rev│              └───────────┘
        │                   └──────┬──────┘
        ▼                          │
  ┌──────────────┐                 │
  │plan-research │◄────────────────┘ (reads existing ADRs)
  │→ Tech.md     │
  └──────┬───────┘
         │
         ▼
   ┌───────────┐
   │plan-steps │
   │→ Steps.md │ (S###, S###-T, S###-D, milestones)
   └─────┬─────┘
         │
         ▼
   ┌───────────┐     ┌─────────────────────────┐
   │impl-start │────▶│ check-task-completion    │ (at each milestone)
   │→ code     │     │ stub/test/doc/drift check│
   └─────┬─────┘     └─────────────────────────┘
         │
         ▼
   ┌───────────┐     ┌──────────────────────┐
   │impl-commit│     │ qa-* agents           │ (before commit/PR)
   │→ commit   │     │ stack-specific checks │
   └───────────┘     └──────────────────────┘

   Periodic:
   ┌────────────────────────────┐     ┌──────────────────────────┐
   │ adr-review                 │     │ check-adr-consistency    │
   │ assertions vs ADRs vs code │     │ ADR graph validation     │
   └────────────────────────────┘     └──────────────────────────┘
```

## Skill reference

| Skill | Invocation | Reads | Writes |
|-------|-----------|-------|--------|
| `plan-spec` | Describe a feature to build | CLAUDE.md, rules | `Context/Features/NNN-Name/Spec.md` |
| `plan-research` | Automatic after spec | Spec.md, CLAUDE.md, rules, ADRs | `Context/Features/NNN-Name/Tech.md` |
| `plan-steps` | Automatic after research | Spec.md, Tech.md, rules | `Context/Features/NNN-Name/Steps.md` |
| `plan-quick` | "Quick plan: [task]" | CLAUDE.md, rules | `Context/Features/NNN-TaskName.md` |
| `impl-start` | "Implement [feature]" or continue | Spec.md, Tech.md, Steps.md, ADRs, rules | Source code, Steps.md progress |
| `impl-commit` | "Commit these changes" | Steps.md, CLAUDE.md, rules | Git commit |
| `adr-create` | "Create an ADR for [decision]" | All existing ADRs, Spec.md | `Context/Decisions/NNNN-title.md`, Spec.md revision history |
| `adr-review` | "Review ADRs" / "Check for drift" | All ADRs, all Spec.md assertions | Report only (read-only) |
| `backlog-add` | "Add [idea/bug] to backlog" | Backlog files | `Context/Backlog/Ideas.md` or `Bugs.md` |
| `backlog-prioritize` | "Prioritize the backlog" | Both backlog files | Updates to backlog files |

`project-init` is a template library used by the `init` command, not invoked directly.

## Agent reference

| Agent | Trigger | Tools | Phases |
|-------|---------|-------|--------|
| `check-task-completion` | Milestone in `impl-start` | Read, Grep, Glob | Load context, identify scope, stub detection, test task verification, doc task verification, deviation detection, ADR coverage check |
| `check-adr-consistency` | `adr-review` or on demand | Read, Grep, Glob | Load ADRs, build graph, contradiction detection, staleness detection, report |
| `qa-python` | On demand or before PR | Read, Grep, Glob, Bash | ruff, mypy, bandit, pattern checks, FastAPI-specific checks |
| `qa-vue` | On demand or before PR | Read, Grep, Glob, Bash | eslint, prettier, vue-tsc, pattern checks, Composition API, accessibility |
| `qa-rust` | On demand or before PR | Read, Grep, Glob, Bash | clippy, rustfmt, cargo audit, pattern checks, Tauri security, async checks |
| `qa-embedded` | On demand or before PR | Read, Grep, Glob, Bash | cppcheck, clang-tidy, clang-format, memory safety, ISR safety, FreeRTOS, type safety |
| `qa-kotlin` | On demand or before PR | Read, Grep, Glob, Bash | detekt, ktlint, pattern checks, coroutine safety, Compose, KMP, DI checks |

Validation agents (`check-task-completion`, `check-adr-consistency`) are strictly read-only — no Bash, no file writes. QA agents use Bash to run external tools but never modify source files.

## Recommended workflow

### During development

1. PostToolUse hooks run stack-specific formatters automatically (ruff, prettier, rustfmt, clang-format, ktlint)
2. At each milestone marker, `impl-start` triggers `check-task-completion` for stub/test/doc/drift validation

### Before commit

1. Run relevant QA agent(s) for modified stacks
2. `check-task-completion` confirms planned tests and docs were produced
3. `impl-commit` formats and creates conventional commit

### Before PR

1. All QA agents for modified stacks
2. pr-review-toolkit agents in parallel (see integration section below)
3. `adr-review` for project-level drift assessment

### Periodically (sprint boundary)

1. `check-adr-consistency` for full ADR graph validation
2. `adr-review` for project-level drift and staleness detection
3. `backlog-prioritize` to review and triage backlog

## pr-review-toolkit integration

Context Engine and [pr-review-toolkit](https://github.com/anthropics/claude-code-plugins) handle complementary concerns:

| Concern | Context Engine | pr-review-toolkit |
|---------|---------------|-------------------|
| Planning compliance | `check-task-completion` verifies against Steps.md and Spec.md assertions | `code-reviewer` checks CLAUDE.md/rules compliance |
| Spec drift | Milestone checkpoints + `adr-review` | Not addressed |
| Decision tracking | ADR workflow + contradiction detection | Not addressed |
| Code quality | Stack-specific `qa-*` agents (linters, type checkers, security) | `code-reviewer` (guidelines), `code-simplifier` (clarity) |
| Test coverage | Verifies planned test tasks were completed | `pr-test-analyzer` (behavioral coverage gaps) |
| Error handling | Stack-specific pattern checks | `silent-failure-hunter` (catch blocks, fallbacks) |
| Documentation | ADRs + Spec.md revision history | `comment-analyzer` (code comment accuracy) |
| Type design | Not addressed | `type-design-analyzer` (encapsulation, invariants) |

pr-review-toolkit works entirely with local git — no GitHub dependency. Install it as a companion:

```
/plugin install pr-review-toolkit
```

## Adding a new stack

To add support for a 6th language/framework:

1. **Create a rule file**: Add a template to `skills/project-init/SKILL.md` following the existing pattern (path-scoped frontmatter + conventions). The `init` command reads templates from this file.

2. **Create a QA agent**: Add `agents/qa-newstack.md`. Follow the existing agent structure:
   - List available tools and expected tool access (Read, Grep, Glob, Bash)
   - Define check categories (linting, type checking, security, pattern checks, framework-specific)
   - Specify the report format

3. **Update stack detection in init**: The `commands/init.md` command detects stacks by scanning for marker files (e.g., `requirements.txt` for Python, `Cargo.toml` for Rust). Add the new stack's markers.

4. **Add test conventions to plan-steps**: The `skills/plan-steps/SKILL.md` skill references stack-specific test frameworks when generating S###-T tasks. Add the new stack's test framework and conventions.

5. **Add overlap definitions to adr-create**: The `skills/adr-create/SKILL.md` skill defines what "same area" means per stack for ADR contradiction detection. Add definitions for the new stack.

6. **Add a post-edit-lint case**: The hook template in `skills/project-init/SKILL.md` includes a case statement per file extension. Add the new formatter.

## Adding skills and agents

### Skills

Skills live in `skills/<name>/SKILL.md`. Claude Code auto-discovers them.

A skill file contains:
- A description of when the skill should be invoked
- The workflow (numbered steps)
- Templates or references to templates
- Tool access requirements

Skills can read project artifacts (CLAUDE.md, rules, Context/ files) and write to Context/ or source files. Reference existing skills for the pattern.

### Agents

Agents live in `agents/<name>.md`. They're invoked by skills or on demand.

An agent file contains:
- A description and trigger conditions
- Allowed tools (explicitly listed)
- Execution phases
- Report format

Validation agents should be read-only (Read, Grep, Glob only). QA agents can additionally use Bash to run external tools.

## Design decisions

**Why structural checkpoints instead of automated semantic comparison?**
Comparing code against spec prose is unreliable and expensive. The system uses milestones, assertions, and required compatibility sections to force human review at defined intervals. The human makes the judgment call; the system ensures the question gets asked.

**Why forward-only ADRs?**
Editing old ADRs creates confusion about what was decided when. The supersedes pattern preserves history — you can always trace how a decision evolved.

**Why mandatory compatibility sections on ADRs?**
Without them, contradiction detection relies on "hope someone notices." The compatibility section shifts the burden to the ADR author, who must explicitly acknowledge and address overlapping decisions before the ADR is created.

**Why paired test/doc tasks default to "on" instead of "off"?**
Tests and docs get skipped because planning doesn't plan for them. Generating paired tasks by default means the developer actively removes unnecessary ones rather than passively forgetting to add needed ones. One-directional bias toward coverage.

**Why no CI gates or coverage thresholds?**
Drift is a judgment call, not a binary. The system surfaces questions; the developer answers them. Coverage thresholds belong in CI config, not in a planning system.

**Why per-stack QA agents instead of a single generic agent?**
Each stack has different tools (ruff vs. clippy vs. detekt), different critical patterns (ISR safety vs. coroutine cancellation vs. XSS), and different severity calibrations. A generic agent would either miss stack-specific issues or produce noisy false positives.

**Why is project-init a separate SKILL.md template library?**
Templates for rules, CLAUDE.md, hooks, and scaffolds are extensive. Keeping them in a dedicated file avoids bloating the init command and makes templates independently editable.
