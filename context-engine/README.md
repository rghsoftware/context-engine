# Context Engine

A Claude Code plugin for structured feature development across multiple tech stacks.

## What it does

**Planning workflow** — 4-phase feature development: Spec → Tech Research → Steps → Implementation. Small tasks get a lightweight quick plan instead.

**ADR tracking** — Architecture Decision Records with automatic contradiction detection against existing decisions. Forward-only supersedes pattern.

**Drift prevention** — Testable assertions in specs, milestone checkpoints during implementation, periodic spec-vs-reality reviews.

**Per-stack quality agents** — Specialized QA for Python/FastAPI, Vue/TypeScript, Rust/Tauri, Embedded C/C++ (ESP32/RP2350), and Kotlin Multiplatform.

**Test & doc task generation** — Implementation steps automatically include paired test tasks (S###-T) and documentation tasks (S###-D).

## Install

### From GitHub

```bash
# In Claude Code:
/plugin marketplace add github.com/rghsoftware/context-engine
/plugin install context-engine@context-engine-marketplace
```

### From local directory

```bash
# Clone the repo, then in Claude Code:
git clone https://github.com/rghsoftware/context-engine.git
/plugin marketplace add /path/to/context-engine
/plugin install context-engine@context-engine-marketplace
```

## Setup per project

After installing the plugin, initialize each project:

```
/context-engine:init
```

This detects your tech stacks, creates path-scoped rules, scaffolds the `Context/`
directory, generates a starter `CLAUDE.md`, and sets up auto-formatting hooks.

**What init creates (project-local):**
- `.claude/rules/` — Conventions for each detected stack
- `.claude/settings.json` — Hook configuration
- `.claude/hooks/post-edit-lint.sh` — Auto-format on save
- `Context/Features/` — Feature planning artifacts
- `Context/Decisions/` — Architecture Decision Records
- `Context/Backlog/` — Ideas and bug tracking
- `CLAUDE.md` — Project overview

**What the plugin provides (global):**
- 10 skills (planning, implementation, ADR, backlog)
- 7 agents (task completion, ADR consistency, 5× stack QA)
- The init command

## Usage

### Plan a new feature

Describe what you want to build. Claude uses the planning skills automatically:

```
I want to add user authentication with OAuth2
```

Claude will walk through:
1. **Spec** — Requirements and testable assertions
2. **Tech** — Architecture decisions and stack details
3. **Steps** — Numbered tasks with test and doc tasks
4. **Implement** — Build with milestone checkpoints

### Quick task

```
Quick plan: fix the login button race condition
```

### Create an ADR

```
We decided to switch from REST to GraphQL — create an ADR
```

Claude loads all existing ADRs, checks for contradictions, and requires explicit
compatibility acknowledgment for overlapping decisions.

### Review ADRs and drift

```
Review ADRs and check for drift
```

### Run quality checks

```
Run the Python quality check on the backend changes
```

### Commit

```
Commit these changes
```

Claude formats code, builds a conventional commit message with task/ADR references.

## What's included

### Skills (auto-invoked based on context)

| Skill | Purpose |
|---|---|
| plan-spec | Phase 1: Feature specification with testable assertions |
| plan-research | Phase 2: Technical architecture and decisions |
| plan-steps | Phase 3: Task breakdown with test/doc tasks and milestones |
| plan-quick | Quick plan for small tasks |
| impl-start | Phase 4: Implementation with milestone checkpoints |
| impl-commit | Commit workflow with formatting and conventional messages |
| adr-create | Create ADR with contradiction detection |
| adr-review | Full ADR consistency and spec drift review |
| backlog-add | Add ideas or bugs to backlog |
| backlog-prioritize | Review and prioritize backlog |

### Agents (invoked for validation and quality)

| Agent | Purpose |
|---|---|
| check-task-completion | Milestone validation: stubs, tests, docs, drift |
| check-adr-consistency | ADR contradiction and staleness detection |
| qa-python | Python/FastAPI: ruff, mypy, bandit, pattern checks |
| qa-vue | Vue/TypeScript: eslint, prettier, vue-tsc, a11y |
| qa-rust | Rust/Tauri: clippy, rustfmt, cargo audit, safety |
| qa-embedded | Embedded C: cppcheck, clang-tidy, ISR/memory safety |
| qa-kotlin | Kotlin/KMP: detekt, ktlint, coroutine/Compose checks |

### Commands

| Command | Purpose |
|---|---|
| /context-engine:init | One-time project setup |

## Companion plugin

For comprehensive code review, also install **pr-review-toolkit**:

```
/plugin install pr-review-toolkit
```

It provides 6 additional review agents (comment-analyzer, pr-test-analyzer,
silent-failure-hunter, type-design-analyzer, code-reviewer, code-simplifier)
that complement this system. Works with GitLab — zero GitHub dependency.

## Adding a new stack

After init, add a stack by creating two files in your project:

1. `.claude/rules/new-stack.md` — with `paths:` frontmatter and conventions
2. Copy a QA agent template — customize checks for the new stack's tools

Add a case to `.claude/hooks/post-edit-lint.sh` for the file extension.

## Customization

- **Stack rules**: Edit `.claude/rules/*.md` in your project directly
- **Planning workflow**: The skills read whatever rules are active — no config needed
- **QA agents**: Edit agent files to add/remove checks for your tools
- **Hooks**: Modify `.claude/hooks/post-edit-lint.sh` for your formatters

## Directory structure

```
context-engine/                    ← This plugin
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── init.md                    ← /context-engine:init
├── skills/
│   ├── plan-spec/SKILL.md
│   ├── plan-research/SKILL.md
│   ├── plan-steps/SKILL.md
│   ├── plan-quick/SKILL.md
│   ├── impl-start/SKILL.md
│   ├── impl-commit/SKILL.md
│   ├── adr-create/SKILL.md
│   ├── adr-review/SKILL.md
│   ├── backlog-add/SKILL.md
│   ├── backlog-prioritize/SKILL.md
│   └── project-init/SKILL.md     ← Templates for init command
└── agents/
    ├── check-task-completion.md
    ├── check-adr-consistency.md
    ├── qa-python.md
    ├── qa-vue.md
    ├── qa-rust.md
    ├── qa-embedded.md
    └── qa-kotlin.md
```
