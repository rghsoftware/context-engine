---
description: Initialize context engineering system in the current project. Detects stacks, creates rules, scaffolds directories, generates CLAUDE.md.
---

# Initialize Context Engineering

Set up this project for the context engineering workflow. Creates per-project files
that the plugin's skills, agents, and hooks need.

**Read the project-init skill first** — it contains all the rule templates and
scaffold content needed for this command. Look in the plugin's
`skills/project-init/` directory.

## Workflow

### Step 1: Detect stacks

Scan the project for these indicators:

| Stack | Look for |
|---|---|
| Python/FastAPI | `requirements.txt`, `pyproject.toml`, `Pipfile`, `*.py` in source dirs, `fastapi` in deps |
| Vue.js/TypeScript | `package.json` with `vue`, `*.vue` files, `tsconfig.json`, `vite.config.ts` |
| Rust/Tauri | `Cargo.toml`, `*.rs`, `tauri.conf.json`, `src-tauri/` |
| Embedded C/C++ | `CMakeLists.txt` with ESP-IDF/Pico, `sdkconfig`, `platformio.ini`, `*.c`/`*.h` in firmware dirs |
| Kotlin/KMP | `build.gradle.kts` with Kotlin plugin, `*.kt`, multiplatform in settings.gradle |

Record which stacks were detected and which directories they live in.

### Step 2: Confirm with user

Present detected stacks and their directories. Let the user correct misdetections,
add stacks that weren't auto-detected, or adjust directory paths. Wait for
confirmation before proceeding.

### Step 3: Create rules

Create `.claude/rules/` with:
- `shared-conventions.md` — always (no paths frontmatter)
- One rule file per confirmed stack with `paths:` pointing to **actual detected directories**

Use the templates from the project-init skill. Substitute real directory paths
into the `paths:` frontmatter — never use hardcoded defaults.

**Never overwrite existing rule files.** If a rule already exists, skip it and
tell the user.

### Step 4: Create Context/ scaffolding

```
Context/
├── Features/.gitkeep
├── Decisions/.gitkeep
└── Backlog/
    ├── Ideas.md
    └── Bugs.md
```

Use the scaffold content from the project-init skill templates.

### Step 5: Create or update CLAUDE.md

If CLAUDE.md doesn't exist, create from the template in project-init skill.
If it exists, append the Development Workflow section without overwriting.

Fill in the Tech Stack table with detected stacks. Ask the user to fill in
project name, overview, architecture, and setup instructions.

### Step 6: Create project hooks

Create `.claude/settings.json` with PostToolUse hook for auto-formatting.
Create `.claude/hooks/post-edit-lint.sh` (executable) with cases only for
detected stacks.

**If `.claude/settings.json` exists, merge the hooks — don't replace.**

### Step 7: Summary

Present what was created and next steps:
1. Customize CLAUDE.md
2. Review rules in `.claude/rules/`
3. Consider installing pr-review-toolkit (`/plugin`)
4. Start planning a feature
