# Skill: Generate Implementation Steps

## Purpose
Break a feature into numbered, ordered implementation tasks with paired test and documentation tasks. This is Phase 3 of the 4-phase planning workflow. Produces Steps.md.

## When to use
- After Tech.md is approved (Phase 2 complete)
- User says "break this down", "create tasks", "plan the steps"

## Prerequisites
- `Context/Features/NNN-FeatureName/Spec.md` exists (approved)
- `Context/Features/NNN-FeatureName/Tech.md` exists (approved)

## Workflow

### Step 1: Load context
1. Read the feature's `Spec.md` — Requirements and Testable Assertions
2. Read the feature's `Tech.md` — Architecture decisions, stack details, testing strategy
3. Read active `.claude/rules/` for stacks this feature touches
4. Read accepted ADRs referenced in Tech.md

### Step 2: Generate implementation tasks
1. Break the feature into sequential implementation tasks (S001–S999)
2. Group tasks into logical phases with milestone markers
3. Mark parallelizable tasks with `[P]`
4. Generate paired test tasks (S###-T) and documentation tasks (S###-D)

### Step 3: Write Steps.md
Write `Context/Features/NNN-FeatureName/Steps.md` using the rules below.

### Step 4: Review with user
1. Present the task breakdown
2. Confirm milestone placement makes sense
3. User may delete -T or -D tasks they deem unnecessary
4. Iterate until approved

## Task Numbering

- **S001–S999**: Implementation tasks
- **S###-T**: Test task paired to S### (placed immediately after S###)
- **S###-D**: Documentation task (placed at milestone boundaries)
- **[P]**: Parallelizable — can run concurrently with adjacent [P] tasks
- **🏁 MILESTONE**: Checkpoint — triggers task completion verification and drift check

## Test Task Generation Rules (S###-T)

**Always generate for:**
- New API endpoints, routes, or IPC command handlers
- New business logic, algorithms, or data transformations
- New data models, schemas, or migrations
- State management changes (stores, caches, session handling)
- Interrupt handlers, peripheral drivers, hardware abstraction layers
- Anything that modifies persistent state

**Skip for:**
- Configuration file changes
- Import reorganization or module restructuring
- Renaming (files, variables, functions) without behavior change
- CSS/styling-only changes
- Dependency version bumps
- Documentation-only changes

**Test task content:**
- Include 3-5 specific test scenarios in parentheses
- Derive scenarios from Spec.md Testable Assertions where they map to this task
- Include edge cases and error conditions relevant to the stack
- Reference the stack's test framework (from the active rule file)

**Stack test conventions:**

| Stack | Framework | Convention |
|---|---|---|
| Python/FastAPI | pytest + httpx AsyncClient | Async tests, fixture-based setup, parametrized edge cases |
| Vue.js/TypeScript | Vitest + Vue Test Utils | Component mount tests, Pinia store tests, composable unit tests |
| Rust/Tauri | cargo test + Tauri mock runtime | #[test] in module, integration tests in tests/ dir |
| Embedded C/C++ | Unity test framework | HAL-abstracted tests, mock peripherals, ISR-safe assertions |
| Kotlin/KMP | kotlin.test + JUnit5 | commonTest for shared logic, platform tests for expect/actual |

## Documentation Task Generation Rules (S###-D)

**Generate when a task:**
- Adds or changes a public API endpoint (→ update API docs)
- Adds a new user-facing feature or capability (→ update README or user docs)
- Changes configuration, environment variables, or setup requirements (→ update setup docs)
- Introduces a new architectural pattern (→ ADR or CLAUDE.md update)
- Changes build or deployment process (→ update build docs)
- Adds a dependency with non-obvious setup steps (→ update install docs)

**Skip when a task:**
- Is internal refactoring with no API or behavior change
- Fixes a bug without changing expected behavior
- Is a test-only change
- Modifies already-documented behavior (unless docs become incorrect)

**Doc task content:**
- Specify which document to update (README.md, API.md, CLAUDE.md, etc.)
- Describe what section or content needs updating
- Place at milestone boundaries, not after every implementation task

## Milestone Placement Rules

Place a `🏁 MILESTONE` marker:
- After completing a logical unit of work (e.g., "data layer done", "API complete")
- When integration between components should be verified
- Before switching stacks (e.g., after backend tasks, before frontend tasks)
- At natural review points where drift checking is valuable

Each milestone marker includes:
- A short description of what was completed
- References to Spec.md Testable Assertions to verify at this checkpoint

## Steps.md Template

```markdown
# Implementation Steps: [Feature Name]

**Spec:** Context/Features/NNN-FeatureName/Spec.md
**Tech:** Context/Features/NNN-FeatureName/Tech.md

## Progress
- **Status:** Not started
- **Current task:** —
- **Last milestone:** —

## Tasks

### Phase 1: [Phase description]
- [ ] S001: [Implementation task description]
- [ ] S001-T: Test [what to test] ([scenario 1, scenario 2, scenario 3])
- [ ] S002: [Implementation task description] [P]
- [ ] S003: [Implementation task description] [P]
- [ ] S003-T: Test [what to test] ([scenario 1, scenario 2, error case])

🏁 MILESTONE: [Phase complete] — verify against [A-001, A-002]

### Phase 2: [Phase description]
- [ ] S004: [Implementation task description]
- [ ] S004-T: Test [what to test] ([scenarios])
- [ ] S005: [Implementation task description]
- [ ] S005-T: Test [what to test] ([scenarios])
- [ ] S005-D: Update [document] — [what to add/change]

🏁 MILESTONE: [Phase complete] — verify against [A-003, A-004]

### Phase 3: Integration & Polish
- [ ] S006: [Integration task]
- [ ] S006-T: Test [integration scenarios]
- [ ] S007-D: Update README — [final documentation updates]

🏁 MILESTONE: Feature complete — verify all assertions, full drift check
```

## Rules
- Tasks must be small enough to complete in one Claude Code session
- Each task must be independently testable or verifiable
- Test tasks follow their implementation task, never batched at the end
- The final milestone always includes "verify all assertions, full drift check"
- Mark Progress section as "Not started" — impl-start skill updates this
- If a task depends on an unresolved Open Question from Spec.md, note it
