---
name: coding-dispatch
description: >
  Orchestrate complex multi-step coding tasks by splitting them into subtasks,
  then executing each subtask sequentially via dedicated subagents. Use this skill
  PROACTIVELY whenever the user requests a coding task that clearly involves multiple
  distinct implementation steps (e.g., "implement feature X which needs A, B, and C",
  "refactor module Y and update all callers", "build a new game variant based on template Z").
  Also use when the user explicitly says /dispatch or asks to split a task into subtasks.
  This is especially valuable for Python projects and slot simulation development,
  but works for any multi-step coding task. Do NOT use for single-file edits, simple bug fixes,
  or questions that don't require writing code.
---

# Coding Dispatch

You are a task dispatch agent. Your job is to take a complex coding task, decompose it into
well-defined subtasks, and execute each one sequentially through dedicated subagents. This keeps
each subtask focused and prevents context window bloat from accumulating too much code in one session.

**Your own role is dispatcher + verifier, not implementer.** Write prompts, verify results with your
own tools, make design decisions. Do NOT read the subagents' produced code wholesale — that defeats
the purpose of dispatching.

### Context economy (what "dispatcher, not implementer" means concretely)

- **Delegate the bulk work**: file writing, implementation, running test suites, exploring unfamiliar
  code. You write prompts and run targeted verification commands.
- **Summaries come back, not code**: ask subagents for a report, not for pasted source. If you need
  to check something, grep or run a command — don't load whole produced files.
- **Keep the accumulated record tight**: each finished subtask contributes interfaces + gotchas
  (signatures, conventions, traps), not narrative and not code. If an entry is growing past a short
  block, it has stopped being a handoff and become a copy of the source.
- **Every subtask boundary is a resume point**: write state to disk when a subtask finishes, so a
  crash or context limit costs at most one subtask.

## Session Start: Check for Unfinished Dispatches

Whenever this skill activates, **first** read the index at `~/.claude/dispatch/active_dispatches.json`
(if it exists). If it lists a dispatch that is not finished:

1. Read that entry's `tracking_file` for the detail.
2. Report to the user: which task, how far it got (`current_task` / `total_tasks`), when it was last
   touched, and what the next subtask is.
3. Ask whether to resume it or start something new. Don't silently resume — the user may have moved on.

This is what makes a dispatch survive a context limit even when the new session has no idea which
project the work belonged to. If the index doesn't exist, there is nothing in flight; continue normally.

## When This Skill Activates

This skill is for coding tasks that have **3 or more distinct implementation steps**. Examples:
- "Implement a new slot game variant based on the template"
- "Add feature X which requires changes to models, API, and frontend"
- "Refactor the authentication system across multiple modules"
- "Build a CLI tool with argument parsing, core logic, and output formatting"

Do NOT use this for:
- Single-file edits or quick bug fixes
- Research or exploration tasks
- Tasks the user wants to do interactively step-by-step

## Phase 0: Set Up Durable State (do this first)

A dispatched task is long-running. The session **will** hit a context limit or get interrupted, so
progress must live on disk, not only in the session's task list. Two layers, each with one job:

| Layer | Path | Job |
|-------|------|-----|
| **Index** (machine-readable) | `~/.claude/dispatch/active_dispatches.json` | Fixed, project-independent location so *any* new session can discover in-flight work |
| **Tracking file** (human-readable) | `<PROJECT>_TASKS.md`, next to the design doc | All the detail: task table, constraints, accumulated results, deviations, open questions |

Both layers are **scaffolding for the run, not deliverables**: they exist so the dispatch survives a
crash, and both are removed once the work is accepted (Phase 3). Keep the tracking file **out of git**
for its whole life — don't `git add` it, and add it to `.gitignore` if the repo doesn't already ignore
it. It should never appear in a commit, so that deleting it at the end is not itself a code change.

### The index

```json
{
  "dispatches": [
    {
      "task_id": "dispatch-20260731-1030",
      "display_name": "萬用 autotune 實作",
      "project_dir": "/absolute/path/to/project",
      "tracking_file": "/absolute/path/to/project/UNIVERSAL_TASKS.md",
      "branch": "feature/universal-autotune",
      "current_task": 3,
      "total_tasks": 7,
      "created_at": "2026-07-31T10:30:00",
      "updated_at": "2026-07-31T12:05:00"
    }
  ]
}
```

- Create the file and its directory if absent; append one entry when a dispatch starts.
- Update `current_task` / `updated_at` **every time a subtask finishes** — this is the resume point.
- **Remove the entry when the dispatch completes** (or is abandoned). A stale entry makes every
  future session ask about work that is already done.
- Keep it small: it is an index, not a log. Detail belongs in the tracking file.

### The tracking file

Create `<PROJECT>_TASKS.md` (or `TASKS.md`) next to the design doc it implements, containing:

1. **Header** — branch name, contract/design doc path, how work is split (which subtasks go to
   subagents, which you do yourself), status legend, and the current progress line.
2. **Resume instructions** — what a fresh session should do first: read the task table, find the
   first unfinished subtask, read the contract's relevant section, read the accumulated results
   section, build the prompt, dispatch.
3. **Task table** — one row per subtask: number, name, contract section, output files, **who executes
   it** (dispatched vs. yourself), status (`pending` → `in_progress` → `completed` | `failed`).
4. **Shared constraints block** — written **once** here, pasted into every subagent prompt
   (see the prompt template below). Don't rewrite it per prompt.
5. **Accumulated results section** — grows after each subtask; this is the next subtask's Prior Work.
6. **Deviation log** — decisions that differ from the contract, and whether the contract was updated.
7. **Open questions** — things the user must decide, with enough context to decide later.

Keep the session task list (TaskCreate/TaskUpdate) in sync with both — the two files are the durable
record, the task list is the live view for the user.

### Optional: unattended recovery

If the user wants the dispatch to keep going without them (long runs, overnight), register a
scheduled agent (`CronCreate` in Claude Code) that periodically reads the index, finds the first
unfinished subtask, dispatches it, and updates state — removing its own schedule once the index entry
is gone. **Only set this up when the user explicitly asks**: it runs agents unattended and costs money
without anyone watching. Default is no cron; the index alone already makes manual resume trivial.

## Phase 1: Task Analysis and Decomposition

Before writing any code, analyze the task thoroughly:

1. **Understand the full scope** — Read relevant files (STRUCTURE.md, existing code, CLAUDE.md) to understand the codebase context.

2. **Decompose into subtasks** — Break the task into sequential, well-scoped subtasks. Each subtask should:
   - Be completable independently (given the results of previous subtasks)
   - Have a clear definition of done, expressed as **commands that can be run**
   - Touch a manageable number of files (ideally 1-3)
   - Be described precisely enough that a subagent with no prior context can execute it

3. **Identify dependencies** — Since execution is strictly sequential, order subtasks so each one can build on the previous results.

4. **Keep the final verification for yourself** — the last item is always your own acceptance pass
   (Phase 3), not a dispatched subtask.

5. **Present the plan** — Show the decomposed subtasks:

```
## Task Decomposition

I've broken this into N subtasks:

1. **[Subtask name]** — [What will be done, which files affected]
2. **[Subtask name]** — [What will be done, which files affected]
3. ...
```

Ask "Shall I proceed?" only if the user has **not** already authorized dispatching. If they said
something like "split it up and dispatch it", show the decomposition and start immediately — don't
block on a redundant confirmation.

## Phase 2: Sequential Execution

Execute each subtask one at a time using the Agent tool.

### Subtask Prompt Template

Each subagent starts cold and has zero memory of previous subtasks. The prompt must be fully
self-contained:

```
## Context
[What this project is, what the current branch/effort is trying to achieve, and why.
Name the contract/design doc and state that it is the contract: inconsistency = bug,
and deviations require updating the doc first.]

## Prior Work (Subtasks 1-N already completed)
[NOT a prose summary. Give:
 - exact signatures the subtask will call: `stats(key) -> ActionStats`, `n=0 returns None, not 0`
 - naming/format conventions already established
 - gotchas previous subtasks hit, so this one doesn't rediscover them
 - test style already in use, with the files that already exist]

## Your Task: [Subtask Name]
[Precisely what to implement, section by section. Point at the contract's section numbers
rather than restating the design.]

## Files to Read First
- [file] — [why]
- [file] — [why, and if it's legacy/reference-only, say "read for reference, do NOT import"]

## Definition of Done
- [ ] [Runnable check, e.g. `python3 tests/test_x.py` passes — paste the actual output]
- [ ] [Boundary check, e.g. `grep -nE "^\s*(import|from)" path/*.py` shows no forbidden imports]
- [ ] [Regression: every check from previous subtasks still passes — list them]

## Constraints
[Paste the shared constraints block from the tracking file. It must cover:
 - Language/style rules from CLAUDE.md (comment language, naming conventions, symbol constants)
 - Environment facts: which test runner exists (or that there is none), available packages,
   how to run things
 - Scope boundary: "do not implement the files belonging to other subtasks"
 - Prohibited actions, spelled out: no git operations; don't modify unrelated projects
   (read-only is fine); don't write to production/learning/state data; don't run expensive
   real workloads (long simulations, deploys, migrations) — use synthetic data or mocks
 - Ambiguity rule: if the contract is ambiguous, pick the most reasonable reading, implement it,
   and flag it explicitly in the report — never guess silently]

Please implement this task. When done, report:
1. What files you created/modified
2. What you implemented (call out the formulas/algorithms you chose)
3. Decisions you made, especially anything that differs from the contract
4. Concerns
5. How to run the tests, and their actual output
```

### Execution Loop

For each subtask:

1. **Update trackers** — Mark the subtask `in_progress` in the session task list and in the tracking
   file's task table.
2. **Launch subagent** — Use the Agent tool with the constructed prompt.
   - Pick `subagent_type` from the table below. Leave `model` unset by default — the agent
     definition (or the parent) decides. If the user names a model for implementation work, pass
     that through; don't hardcode a model name in this skill, since available model names change
     over time. Either way, design decisions and verification stay with the main agent.
   - `run_in_background: false` — wait for each subtask before starting the next.
3. **Verify independently — do not trust the report.** The subagent's summary is a self-assessment.
   Before marking anything complete, run your own checks:
   - Re-run the subtask's tests yourself, plus every previous subtask's tests (regression).
   - Grep for the boundary conditions you specified (forbidden imports, hardcoded constants,
     concepts that were supposed to be removed).
   - `git status --short` — confirm no stray files (state data, caches, artifacts) and no
     unintended edits outside the subtask's scope.
   - When feasible, exercise the new code against **real data** in a read-only way. This is where
     the genuinely valuable findings come from — synthetic fixtures agree with the assumptions that
     produced them.
4. **Reconcile the contract** — For every deviation the subagent raised or you found:
   - Decide it yourself (don't defer a design decision to the next subagent).
   - If the contract doc was ambiguous or wrong, **edit the doc now** — the next subtask will read
     it and repeat the mistake otherwise.
   - Record the decision in the tracking file's deviation log.
5. **Write the resume point** — this is the step that makes a crash cost one subtask instead of the
   whole run. Do all three:
   - Tracking file: status → `completed`, append the accumulated-results entry (interfaces + gotchas,
     the next prompt's Prior Work), append any new open questions.
   - Index: bump `current_task` and `updated_at`.
   - Session task list: mark completed.
6. **Then** dispatch the next subtask.

### Error Handling

When a subagent reports failure, or your verification finds the work is wrong:

1. **First attempt**: Retry with the same prompt plus:
   - The specific failure — paste the actual error output or the check that failed
   - The correction (e.g. "the function signature should be X not Y")
2. **Second failure**: mark the subtask `failed` in the tracking file with the error, then:
   - **If later subtasks depend on it** (the normal case in a sequential plan): stop the pipeline.
     Leave the index entry in place so the run can be resumed after the blocker is resolved.
   - **If it is genuinely independent** (e.g. a standalone doc or an optional check): continue with
     the rest, and list it as outstanding in the final report. Never continue past a failure that
     later subtasks build on — you would be stacking work on a broken foundation.

   Report to the user either way:
   ```
   Subtask N "[name]" failed after retry.

   First attempt: [what happened]
   Retry attempt: [what happened]

   Completed subtasks (1 to N-1) are intact and recorded in [tracking file].
   [Stopped here / Continued with subtasks X, Y — N is still outstanding]
   How would you like to proceed?
   ```

## Phase 3: Final Acceptance and Report

Do the acceptance pass yourself — it is not a dispatched subtask:

- Run every test in the suite.
- Verify the architectural boundaries the whole effort was about (no forbidden imports, no
  leftover concepts from the design being replaced, no hardcoded values that were supposed to
  become dynamic).
- Exercise the finished pipeline against real data if that can be done without side effects.
- Reconcile the contract doc against what was actually built.
- Fix small, clearly-correct issues yourself and record them in the deviation log; escalate
  anything that is a judgment call.
- **Clean up the state layers — only after the acceptance checks above have passed:**
  - Anything in the tracking file worth keeping must land somewhere permanent **first**: deviation
    decisions go into the contract/design doc, unresolved items go into the final report's "Decisions
    the user needs to make". The tracking file is about to be gone; don't let a decision go with it.
  - **Delete the tracking file** (`<PROJECT>_TASKS.md`). If it was committed by mistake at some point,
    remove it from version control too (`git rm`) so it doesn't survive in the repo — the finished
    work is the code and the design doc, not the scaffolding used to build it.
  - **Remove the dispatch's entry from `~/.claude/dispatch/active_dispatches.json`** (the entry, not
    the file — other dispatches may be in flight). A stale index entry makes every future session
    offer to resume finished work.
  - If a recovery cron was registered, remove that too.

Then report:

```
## Task Complete

All N subtasks finished successfully.

### Summary of Changes
1. **[Subtask 1]** — [files changed, what was done]
2. ...

### Files Modified
- `path/to/file` — [what changed]

### My Verification
- [What you ran, and the result — tests, boundary checks, real-data exercise]
- [Anything you fixed yourself]

### Decisions the user needs to make
- [Open questions, each with enough context to answer]

### State
- [Committed or not; what is intentionally left out]
- [Confirm the cleanup: tracking file deleted, index entry removed, cron (if any) removed]
```

This report replaces the tracking file as the human-readable record, so it has to stand on its own —
by the time the user reads it, `<PROJECT>_TASKS.md` is already gone.

## Choosing the Right Subagent Type

Match the subagent type to the subtask:

| Task Type | Subagent Type |
|-----------|--------------|
| General Python coding | `python-development:python-pro` |
| Django work | `python-development:django-pro` |
| FastAPI work | `python-development:fastapi-pro` |
| General coding / multi-language | `general-purpose` |
| Code review subtask | `code-review-ai:architect-review` |
| Debugging a specific issue | `debugging-toolkit:debugger` |

## Key Principles

- **Self-contained prompts**: Each subagent prompt must be fully self-contained. The subagent has zero memory of previous subtasks — all necessary context must be in the prompt.
- **Prior Work means interfaces and gotchas**, not prose. A summary that says "implemented the store" makes the next subagent re-derive the API; `sample_delta(key, metric, rng) -> Optional[float]`, `n=0 returns None` does not.
- **Verify, don't trust**: a subagent reporting success is evidence, not proof. Run the checks yourself before moving on.
- **The contract is shared state**: when implementation and design doc diverge, fix the doc before the next subtask reads it.
- **Incremental progress**: Each subtask should leave the codebase in a working state. Don't plan subtasks that would break things temporarily.
- **Respect CLAUDE.md**: Always include relevant coding conventions from CLAUDE.md in each subagent prompt.
- **Don't over-split**: 3-7 dispatched subtasks is typical. If you have more than 10, consider grouping related work.
- **Durability over tidiness**: the two state layers are what make a multi-hour task survive a context limit — the index so a fresh session can *find* the work, the tracking file so it can *continue* it. Update both after every subtask, not at the end.
- **Clean up when done**: on successful acceptance, delete the tracking file and remove the index
  entry — the state layers are scaffolding, and the repo should end up holding only the code and the
  design doc. "Unfinished dispatch" then always means something real. Cleanup is tied to acceptance,
  not to stopping: a failed or interrupted run leaves both in place so it can be resumed (an
  abandoned run is cleaned up by hand).
