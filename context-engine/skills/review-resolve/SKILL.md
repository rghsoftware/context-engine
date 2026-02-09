# Skill: Resolve Review Findings

## Purpose
Work through a captured review file, resolve each finding by category, generate
appropriate planning artifacts, and update the review file with resolution status.

## When to use
- At the start of a fresh session after a review was captured
- User says "resolve review", "fix review findings", "work through the review"
- User references a specific review file in Context/Reviews/

## Prerequisites
- A review file must exist in `Context/Reviews/` (created by review-capture)
- This should run in a **fresh session** with full context budget — not in the same
  session that performed the review

## Workflow

### Step 1: Load the review file
1. If the user specified a file, load that one
2. Otherwise, list files in `Context/Reviews/` and find the most recent with
   `Status: 🔴 Unresolved`
3. If no unresolved reviews exist, tell the user — nothing to do
4. Read the full review file
5. Load the associated feature's Spec.md, Steps.md, and Tech.md for context

### Step 2: Work through Fix-Now findings

Process each `[FIX]` finding in priority order (Critical → High → Medium → Low):

1. Open the referenced file and line
2. Apply the fix
3. Verify the fix doesn't break anything obvious (check imports, references)
4. Update the finding in the review file:
   - **Status:** ✅ Fixed
   - **Resolution:** {Brief description of what was changed}

If a fix turns out to be more complex than expected (requires restructuring,
affects other modules, changes behavior), reclassify it:
- Reclassify to [TASK] if it needs tracked work
- Reclassify to [ADR] if it requires an architectural decision
- Update the category label and move the finding to the correct section

### Step 3: Work through Missing Task findings

For each `[TASK]` finding:

1. Determine the next available S### number from Steps.md
2. Add the new task to Steps.md in the appropriate phase:
   - If it's a test → add as S###-T after the related implementation task
   - If it's documentation → add as S###-D after the related implementation task
   - If it's new implementation → add at the end of the relevant phase
3. If the finding relates to a Testable Assertion, verify the assertion exists in
   Spec.md. If not, add it to the Testable Assertions table.
4. Update the finding in the review file:
   - **Status:** ✅ Task created
   - **Resolution:** Added as S### in Steps.md

Do NOT implement the task now — just track it. Implementation follows the normal
impl-start workflow.

### Step 4: Work through Architectural Concern findings

For each `[ADR]` finding:

1. Present the concern to the user with context:
   - What the reviewer found
   - What the current Tech.md says
   - What existing ADRs are relevant
2. Ask the user to decide:
   - **Create ADR** — proceed to use the adr-create skill
   - **Dismiss** — the concern was considered but doesn't warrant a decision change
     (user must provide a reason)
   - **Defer** — mark as deferred for later consideration
3. Update the finding in the review file:
   - If ADR created: **Status:** ✅ ADR created → **Resolution:** ADR-NNNN
   - If dismissed: **Status:** ✅ Dismissed → **Resolution:** {user's reason}
   - If deferred: **Status:** 🟡 Deferred → **Resolution:** Deferred — {reason}

### Step 5: Work through Convention Gap findings

For each `[RULE]` finding:

1. Identify which rule file should be updated (or if a new rule file is needed)
2. Read the current rule file
3. Present the suggested addition to the user
4. If approved:
   - Add the convention to the appropriate rule file
   - If the gap is something the post-edit hook could catch, update the hook too
5. Update the finding in the review file:
   - If applied: **Status:** ✅ Rule updated → **Resolution:** Added to {rule-file}
   - If dismissed: **Status:** ✅ Dismissed → **Resolution:** {reason}

### Step 6: Update review file header

After processing all findings, update the review file:

1. Count resolutions:
   - All resolved → **Status:** 🟢 Resolved
   - Some deferred → **Status:** 🟡 Partially resolved
   - Any unresolved → **Status:** 🔴 Unresolved (should not happen if workflow completes)
2. Update the Resolution Checklist at the bottom (check completed items)
3. Add a Resolution Summary section:

```markdown
## Resolution Summary
**Resolved at:** {YYYY-MM-DD}
**Session:** {brief description}

| Category | Total | Fixed | Tasks Created | ADRs | Rules | Dismissed | Deferred |
|---|---|---|---|---|---|---|---|
| [FIX] | N | N | — | — | — | — | — |
| [TASK] | N | — | N | — | — | — | — |
| [ADR] | N | — | — | N | — | N | N |
| [RULE] | N | — | — | — | N | N | — |
| **Total** | **N** | | | | | | |
```

### Step 7: Commit resolution work

Use the impl-commit skill to commit the fixes. The commit message should reference
the review file:

```
fix(review): resolve PR review findings P{N}

Fixes: [FIX] items resolved inline
Tasks: S### added for [TASK] items
ADR-NNNN created for [ADR] items
Rules updated for [RULE] items

Review: Context/Reviews/PR-{branch}-{date}.md
```

### Step 8: Suggest next steps

Based on what was generated:
- If new tasks were added → "Run impl-start to work through the new tasks"
- If ADRs were created → "Review the new ADRs to make sure they're complete"
- If rules were updated → "The updated rules will apply to future code automatically"
- Always → "Run review-verify to confirm all findings are resolved"

## Rules
- ALWAYS start by loading the review file — don't work from conversation memory
- Process findings in category order: FIX → TASK → ADR → RULE
- Within each category, process in priority order: Critical → High → Medium → Low
- Never skip a finding without explicit user input (dismiss or defer with reason)
- Never auto-dismiss architectural concerns — the user must decide
- If context is getting tight, save progress to the review file immediately
  (update statuses for what's done so far) before continuing
- Keep the review file as the single source of truth — don't track resolutions
  anywhere else
