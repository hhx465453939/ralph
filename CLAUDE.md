# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ralph is an autonomous AI agent loop system that runs [Amp](https://ampcode.com) repeatedly until all PRD items are complete. Each iteration spawns a fresh Amp instance with clean context. Memory persists via git history, `progress.txt`, and `prd.json`.

## Key Architecture Concepts

### Fresh Context Per Iteration
Each Ralph iteration creates a **new Amp instance** with no memory of previous work. The only state that persists between iterations:
- Git history (commits from completed stories)
- `progress.txt` - Append-only log of learnings and discovered patterns
- `prd.json` - User stories with `passes: true/false` status

### Story Sizing is Critical
Each user story MUST be completable in ONE context window. If a story is too large:
- The LLM runs out of context before finishing
- Broken code gets committed
- Future iterations build on broken foundation

**Right-sized stories:** Add a database column, add a UI component, update a server action
**Too big:** "Build the entire dashboard", "Add authentication"

### Dependency Ordering
Stories execute in `priority` order. Earlier stories must NOT depend on later ones.

**Correct order:** Schema changes → Server actions → UI components → Aggregation views

## Common Commands

### Development - Flowchart Visualization
```bash
cd flowchart
npm install
npm run dev      # Start dev server
npm run build    # Build for production
```

The flowchart directory contains an interactive React Flow diagram explaining how Ralph works.

### Running Ralph (in target projects)
```bash
./scripts/ralph/ralph.sh [max_iterations]
```
Default is 10 iterations. The script will:
1. Check out/create the branch specified in `prd.json.branchName`
2. Pick the highest priority story with `passes: false`
3. Implement that single story
4. Run quality checks (typecheck, lint, test)
5. Commit if checks pass
6. Update `prd.json` to mark story as `passes: true`
7. Append learnings to `progress.txt`
8. Repeat until all stories pass or max iterations reached

### Debugging Ralph State
```bash
# See which stories are done
cat prd.json | jq '.userStories[] | {id, title, passes}'

# See learnings from previous iterations
cat progress.txt

# Check git history
git log --oneline -10
```

## Key Files and Their Purposes

| File | Purpose |
|------|---------|
| `ralph.sh` | Bash loop that spawns fresh Amp instances |
| `prompt.md` | Instructions given to each Amp instance |
| `prd.json` | User stories with `passes` status (the task list) |
| `prd.json.example` | Example PRD format for reference |
| `progress.txt` | Append-only learnings for future iterations |
| `skills/prd/` | Skill for generating PRDs from feature descriptions |
| `skills/ralph/` | Skill for converting PRDs to `prd.json` format |
| `flowchart/` | Interactive React Flow visualization |

## Ralph Skills System

### PRD Skill (`skills/prd/SKILL.md`)
- **Purpose:** Generate Product Requirements Documents from feature descriptions
- **Trigger phrases:** "create a prd", "write prd for", "plan this feature"
- **Output:** `tasks/prd-[feature-name].md`
- **Key requirements:**
  - Ask 3-5 clarifying questions with lettered options
  - User stories must be small and verifiable
  - Always include "Typecheck passes" in acceptance criteria
  - For UI stories: include "Verify in browser using dev-browser skill"

### Ralph Skill (`skills/ralph/SKILL.md`)
- **Purpose:** Convert markdown PRDs to `prd.json` format
- **Trigger phrases:** "convert this prd", "turn this into ralph format"
- **Output:** `prd.json`
- **Critical rules:**
  - Each story must be completable in ONE iteration
  - Stories ordered by dependencies (schema → backend → UI)
  - All acceptance criteria must be verifiable
  - UI stories require browser verification

## PRD JSON Structure

```json
{
  "project": "ProjectName",
  "branchName": "ralph/[feature-name-kebab-case]",
  "description": "Feature description",
  "userStories": [
    {
      "id": "US-001",
      "title": "Story title",
      "description": "As a [user], I want [feature] so that [benefit]",
      "acceptanceCriteria": [
        "Verifiable criterion 1",
        "Verifiable criterion 2",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

## Progress Tracking

### progress.txt Format
The progress file has TWO sections:

1. **Codebase Patterns** (at the TOP) - Consolidated reusable patterns:
```markdown
## Codebase Patterns
- Example: Use `sql<number>` template for aggregations
- Example: Always use `IF NOT EXISTS` for migrations
```

2. **Iteration Logs** (appended after each story):
```markdown
## [Date/Time] - [Story ID]
Thread: https://ampcode.com/threads/$AMP_CURRENT_THREAD_ID
- What was implemented
- Files changed
- **Learnings for future iterations:**
  - Patterns discovered
  - Gotchas encountered
  - Useful context
---
```

### AGENTS.md Updates
After each iteration, update `AGENTS.md` files in directories where you made changes:
- API patterns or conventions specific to that module
- Gotchas or non-obvious requirements
- Dependencies between files
- Testing approaches for that area

**Do NOT add:** Story-specific details, temporary debugging notes, information already in progress.txt

## Stop Condition

When ALL stories have `passes: true`, Ralph outputs:
```
<promise>COMPLETE</promise>
```
And the loop exits successfully.

## Auto-Archiving

When starting a new feature (different `branchName`), Ralph automatically archives:
- Previous `prd.json`
- Previous `progress.txt`
- To: `archive/YYYY-MM-DD-feature-name/`

## Amp Configuration (Recommended)

Add to `~/.config/amp/settings.json`:
```json
{
  "amp.experimental.autoHandoff": { "context": 90 }
}
```

This enables automatic handoff when context fills up, allowing Ralph to handle large stories that exceed a single context window.

## Critical Success Factors

1. **Small stories** - Each must complete in one context window
2. **Quality checks** - Typecheck, lint, tests must pass before committing
3. **AGENTS.md updates** - Future iterations benefit from discovered patterns
4. **Browser verification** - Required for all UI stories using dev-browser skill
5. **Clean CI** - Broken code compounds across iterations
