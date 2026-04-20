---
name: 'implement-epic'
description: 'Orchestrate full epic implementation with dependency-aware parallel execution: parse waves, spawn parallel agents per wave, run create → ATDD → develop → code review per story'
---

Your task is to orchestrate the implementation of stories from the epic file:
`_bmad-output/planning-artifacts/epics/$ARGUMENTS`

## Phase 1: Parse Epic & Build Execution Plan

1. Read the epic file completely
2. Extract ALL stories (look for `## Story X.Y:` headers)
3. **Check story statuses** — skip stories marked as **DONE**, **SKIPPED**, or **N/A** in their status annotations
4. For stories marked **PARTIAL**, include them but note which ACs are already done — pass this context to agents
5. **Parse the dependency and parallelization sections:**
   - Look for `## Story Dependencies`, `### Per-Story Dependency Table`, `### Execution Waves`, `### Critical Path`
   - If these sections exist, use them to determine execution order and parallelism
   - If they do NOT exist, fall back to sequential execution (lowest story number first)
6. Display the execution plan to the user:

```
Implementing Epic: {name}

Skipped (done/N/A): {list}
Remaining: {list}

Execution Waves:
  Wave 1 (sequential): {stories}
  Wave 2 (parallel):   {stories}
  Wave 3 (parallel):   {stories}
  ...

Critical path: {story} → {story} → {story}
```

7. Wait for user confirmation before proceeding. If user says to proceed, continue. If user wants to modify the plan (e.g., skip stories, reorder), adjust accordingly.

## Phase 2: Create Task Graph

Use `TaskCreate` and `TaskUpdate` to build a dependency-aware task graph:

1. For each remaining story, create a task:
   - **subject**: `Story {number}: {title}`
   - **description**: Include story number, title, status (partial/new), which ACs are done (if partial), and the list of files mentioned in implementation notes
   - **metadata**: `{ "storyNumber": "{number}", "wave": "{wave_number}" }`

2. Set up dependencies using `TaskUpdate`:
   - For each story's `Depends On` entries, use `addBlockedBy` to link tasks
   - This ensures blocked stories cannot start until their dependencies complete

## Phase 3: Wave-Based Execution

Process stories wave by wave. Within each wave, stories run in parallel where possible.

### Wave Execution Rules

- **Within a wave**: Launch all unblocked stories in parallel using the Agent tool
  - Spawn one **orchestrator agent per story** (named `story-{number}`, e.g., `story-21.6`)
  - Each orchestrator runs the full 4-step pipeline for its story sequentially
  - Use `run_in_background: true` for all but the last story in a wave (to enable parallelism)
- **Between waves**: Wait for ALL stories in the current wave to complete before starting the next wave
- **Within a story pipeline**: The 4 steps (create → ATDD → develop → review) are ALWAYS sequential
- **Partial stories**: Pass context about already-completed ACs to the agents so they don't redo work

### Parallel Safety Rules

- **Wave 1** stories (foundation/infrastructure) should ALWAYS run sequentially — they create shared utilities that later waves depend on
- **Waves 2+** can run stories in parallel IF:
  - Stories touch different modules/files (check the epic's file references)
  - Stories have no `blockedBy` dependencies on each other within the same wave
- **Code Review + Commit (Agent 4)** for parallel stories must run sequentially (one commit at a time) to avoid git conflicts. Use `TaskUpdate` to coordinate: when a story reaches the review stage, check if another review is in progress and wait
- If two parallel stories are likely to touch the same files, run them **sequentially** instead — file conflicts are not worth the parallelism gain

### Wave Completion Check

After each wave:
1. Call `TaskList` to verify all wave tasks are `completed`
2. Run `git log --oneline -N` (where N = number of stories in wave) to capture commit hashes
3. Display wave summary before proceeding to next wave

## Phase 4: Story Pipeline (Per-Story Orchestrator Agent)

Each story orchestrator agent runs 4 sub-agents sequentially. The orchestrator agent prompt:

```
You are a story orchestrator for Story {story_number}: {title}.
Your job is to run the full BMAD implementation pipeline for this story by spawning 4 agents sequentially.

Story context:
- Epic file: _bmad-output/planning-artifacts/epics/{epic_file}
- Status: {new|partial — if partial, list completed ACs}
- Dependencies completed: {list of completed dependency stories}

Run these 4 agents SEQUENTIALLY using the Agent tool (mode: "bypassPermissions"). Each must complete before the next starts.

{Agent 1 definition}
{Agent 2 definition}
{Agent 3 definition}
{Agent 4 definition}

After all 4 complete, update your task status to completed using TaskUpdate.

CRITICAL: Never output large blocks of Russian/Cyrillic text verbatim. Summarize Russian content and reference it by key or variable name. Use Unicode escape sequences for Russian strings in test fixtures. Process Cyrillic-heavy files in small chunks. This prevents API 400 content filtering errors.
```

## Agent Definitions

In each agent prompt below, replace `{story_number}` with the current story number being processed.

### Agent 1 — Create Story

Spawn a `general-purpose` agent with `mode: "bypassPermissions"` and `model: "opus"` and this prompt.
**Use ultrathink (high reasoning effort)** — prepend the agent prompt with: `<system-reminder>The user has requested reasoning effort level: high. Apply this to the current turn.</system-reminder>`

```
You are executing a BMAD workflow autonomously. Enter YOLO mode immediately — skip ALL interactive confirmations, prompts, and "continue?" questions throughout the entire workflow.

Steps:
1. Load and READ the COMPLETE file: _bmad/core/tasks/workflow.xml (no offset/limit)
2. This is the workflow execution engine. Now load the workflow config: _bmad/bmm/workflows/4-implementation/create-story/workflow.yaml
3. Execute the workflow to create story {story_number} — follow all workflow.xml instructions exactly
4. After the story file is created, load and execute the validation checklist from the workflow
5. YOLO mode is active for the ENTIRE workflow — never wait for user input

CRITICAL: Never output large blocks of Russian/Cyrillic text verbatim. Summarize instead.
```

### Agent 2 — ATDD Checklist

Spawn a `general-purpose` agent with `mode: "bypassPermissions"` and this prompt:

```
You are executing a BMAD workflow autonomously. Enter YOLO mode immediately — skip ALL interactive confirmations, prompts, and "continue?" questions throughout the entire workflow.

Steps:
1. Load and READ the COMPLETE file: _bmad/core/tasks/workflow.xml (no offset/limit)
2. This is the workflow execution engine. Now load the workflow config: _bmad/tea/workflows/testarch/atdd/workflow.yaml
3. Execute the workflow to generate the ATDD acceptance test checklist for story {story_number}
4. YOLO mode is active for the ENTIRE workflow — never wait for user input

CRITICAL: Never output large blocks of Russian/Cyrillic text. Use Unicode escape sequences for Russian strings in test fixtures.
```

### Agent 3 — Develop Story

Spawn a `general-purpose` agent with `mode: "bypassPermissions"` and this prompt:

```
You are executing a BMAD workflow autonomously. Enter YOLO mode immediately — skip ALL interactive confirmations, prompts, and "continue?" questions throughout the entire workflow.

Steps:
1. Load and READ the COMPLETE file: _bmad/core/tasks/workflow.xml (no offset/limit)
2. This is the workflow execution engine. Now load the workflow config: _bmad/bmm/workflows/4-implementation/dev-story/workflow.yaml
3. Execute the workflow to develop/implement story {story_number} — follow all workflow.xml instructions exactly
4. Use the ATDD checklist (look for atdd-checklist-*.md in _bmad-output/implementation-artifacts/ or _bmad-output/test-artifacts/) as your implementation guide
5. Do NOT create any git commits at this stage
6. YOLO mode is active for the ENTIRE workflow — never wait for user input

CRITICAL: Never output large blocks of Russian/Cyrillic text verbatim. Summarize instead.
```

### Agent 4 — Code Review

Spawn a `general-purpose` agent with `mode: "bypassPermissions"` and `model: "opus"` and this prompt.
**Use ultrathink (high reasoning effort)** — prepend the agent prompt with: `<system-reminder>The user has requested reasoning effort level: high. Apply this to the current turn.</system-reminder>`

```
You are executing a BMAD workflow autonomously. Enter YOLO mode immediately — skip ALL interactive confirmations, prompts, and "continue?" questions throughout the entire workflow.

Steps:
1. Load and READ the COMPLETE file: _bmad/core/tasks/workflow.xml (no offset/limit)
2. This is the workflow execution engine. Now load the workflow config: _bmad/bmm/workflows/4-implementation/code-review/workflow.yaml
3. Execute the workflow to perform adversarial code review on story {story_number}
4. Auto-fix ALL found issues without asking for confirmation
5. When the story status is set to "done", create a git commit with all changes (per CLAUDE.md: mandatory commit when story status → done)
6. YOLO mode is active for the ENTIRE workflow — never wait for user input

CRITICAL: Never output large blocks of Russian/Cyrillic text verbatim. Summarize instead.
```

## Phase 5: Completion

After all waves are complete:

1. Call `TaskList` to verify all tasks are `completed`
2. Run `git log --oneline` to collect all commit hashes from this epic
3. Display final summary:

```
Epic: {name}
Stories completed: {count}/{total} ({skipped} skipped)
Execution waves: {wave_count}

Wave Results:
  Wave 1: {stories} — {sequential|parallel}
  Wave 2: {stories} — {sequential|parallel}
  ...

Commits:
  {hash} {message}
  {hash} {message}
  ...
```

## Error Handling

- **Agent failure**: If any agent in the pipeline fails, stop that story's pipeline immediately. Mark the task as `in_progress` (not completed). Display the error and ask the user whether to retry, skip, or abort the epic.
- **Build failure**: If Agent 3 (develop) or Agent 4 (review) reports build failures, stop and surface the error. Do NOT proceed to the next story in the same wave — build failures may affect parallel stories.
- **Git conflict**: If Agent 4 (commit) fails due to git conflicts, stop ALL parallel stories in the wave. Surface the conflict for manual resolution.
- **Wave failure**: If any story in a wave fails, do NOT start the next wave. Complete remaining stories in the current wave, then pause for user input.

## Fallback: No Dependency Graph

If the epic file does NOT contain a `## Story Dependencies` or `### Execution Waves` section:

1. Fall back to **fully sequential execution** (original behavior)
2. Process stories in ascending number order
3. Run the 4-agent pipeline for each story one at a time
4. This is the safe default — no parallelism without explicit dependency information
