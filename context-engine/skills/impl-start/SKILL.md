# Skill: Start Implementation

## Purpose
Begin or resume implementation of a planned feature. Restores full context from planning artifacts and works through Steps.md tasks sequentially. This is Phase 4 of the 4-phase planning workflow.

## When to use
- Starting implementation after planning is complete
- Resuming work in a new Claude Code session
- User says "start working", "implement this", "continue where we left off"

## Prerequisites
- Feature has at minimum a Spec.md (full plan) or a quick plan file
- For full features: Spec.md, Tech.md, and Steps.md all exist

## Workflow

### Step 1: Restore context
1. Read `CLAUDE.md` for project architecture
2. Identify which feature to work on:
   - If user specifies, use that feature
   - If resuming, find the feature with incomplete tasks in Steps.md
   - If ambiguous, ask the user
3. Read the feature's planning artifacts:
   - `Spec.md` — requirements and testable assertions
   - `Tech.md` — architecture decisions and stack details
   - `Steps.md` — task list and progress
4. Read all accepted ADRs in `Context/Decisions/` relevant to this feature
5. Read active `.claude/rules/` for stacks this feature touches

### Step 2: Identify current task
1. Find the first unchecked task in Steps.md
2. Update the Progress section:
   - **Status:** In progress
   - **Current task:** S### — [description]
   - **Last milestone:** [most recent completed milestone, or "—"]
3. Present the current task and ask user to confirm before proceeding

### Step 3: Implement the task
1. Execute the task as described in Steps.md
2. Follow the active stack rules for conventions and patterns
3. If the task is S###-T (test task): write tests using the stack's framework
4. If the task is S###-D (doc task): update the specified documentation
5. After completing the task, mark it as done: `- [x] S###: ...`

### Step 4: Check for milestone
If the completed task is immediately before a `🏁 MILESTONE` marker:

1. **Run stub detection** — search completed files for:
   - `// TODO:` or `# TODO:` without `tracked in Steps.md S###`
   - `// FIXME:` or `# FIXME:`
   - `pass` as sole function body (Python)
   - `todo!()` or `unimplemented!()` (Rust)
   - `throw NotImplementedError` or `throw new Error("not implemented")`
   - `// handled elsewhere` without verified code at the destination
   - Empty method/function bodies

2. **Verify test/doc tasks** — check that all S###-T and S###-D tasks
   before this milestone are completed

3. **Drift checkpoint** — pull the Testable Assertions referenced in the
   milestone marker from Spec.md and present them:
   - "Here are the assertions to verify at this milestone:"
   - List each assertion with its ID
   - Ask: "Does the current implementation align with these? Options:"
     - **Aligned** — proceed to next phase
     - **Diverged — need ADR** — pause implementation, create ADR documenting the deviation
     - **Diverged — spec was wrong** — update Spec.md with revision history entry

4. Report results before continuing to the next task

### Step 5: Continue or complete
- If more tasks remain, return to Step 2
- If all tasks are complete, update Progress status to "Complete"
- Suggest running the full review workflow before committing

## Rules
- Never skip a task unless the user explicitly approves
- Never skip milestone checkpoints — they are not optional
- If a task reveals that the plan is wrong, pause and discuss with the user rather than improvising
- If context window is filling up, save progress to Steps.md and tell the user to start a new session
- Test tasks (S###-T) must produce actual test files, not just assertions in comments
- Doc tasks (S###-D) must modify the specified document, not just note what to change
- If an ADR is needed at a milestone, use the adr-create skill before continuing
