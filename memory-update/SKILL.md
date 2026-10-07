---
name: memory-update
description: "Checks the agent's memory (AGENTS.md and memory/) against the current state of the autotest repository and related documentation, shows the differences and fixes the memory after confirmation. Use after a merged feature, after a library or base class change, or when the agent confidently refers to something that no longer exists in the code."
---

# memory-update: keeping the agent's memory current

Stale memory is more dangerous than no memory: without memory the agent will at least
study the code again; with stale memory it does not doubt what it "remembers" and
writes by yesterday's rules. This skill keeps the memory current by checking only the delta.

## When to use this skill

- After changes merged into the main branch: a new feature, a new controller, a new helper
- After a change of a library version, a base class, or the environment configuration
- On a symptom: the agent refers to a class, method or rule that does not exist
- On a schedule, if changes are small but frequent (once a week is enough)

Do not activate if: there is no AGENTS.md. Run memory-init first.

## Preconditions

- `AGENTS.md` and the `memory/` directory exist
- Access to the repository history (git)
- Ideally the issue tracker and wiki MCP servers are connected: then links to tasks
  and specifications are checked as well

## Workflow

After every step a short summary and an explicit confirmation from the user.

### Step 1 — Determine the delta

- Read `AGENTS.md` and its "Revision history" section: date and commit of the last sync
- Collect the changes since then: `git log --stat` from the last sync to HEAD,
  autotest, config and build files only
- If there is no revision history, treat the whole repository as the delta and warn
  that the check will take a while
- Output: the list of changed files grouped by memory topic
  (architecture, conventions, patterns, stack, process, examples, pitfalls)

### Step 2 — Check every memory file

For each file in `memory/` go through the items and verify by opening the code:

- `project-overview.md` — new modules, environments, external dependencies?
- `architecture.md` — has the base class hierarchy, context initialization, lifecycle
  or containers changed; does every class mentioned exist at the given path
- `conventions.md` — new stable conventions in the code, or violations of old ones
  (if a violation is widespread, the rule is stale, not the code)
- `test-patterns.md` — is there a new test type without a template
- `frameworks-stack.md` — do the versions in the table match `pom.xml` / `build.gradle`,
  the language version, the framework version
- `workflow.md` — have the branch, commit, MR formats or task statuses changed
- `examples.md` — are the reference tests current: the file exists, the code matches
  the repository; if a reference was rewritten, update the copy; if deleted, pick a new one
- `common-pitfalls.md` — add pitfalls found in recent tasks (failures, flakiness,
  rollbacks); remove items that no longer reproduce

If MCP servers are connected: check the task and wiki links in the memory — they exist,
have not moved, are not marked obsolete.

Priority rule: code beats memory. On divergence the memory is fixed.
Exception: if the code violates a rule from `conventions.md` in one or two places,
that is a code defect: report it to the user and do not change the rule.

Output: a table of differences "memory file → item → what the code says → proposed fix".

### Step 3 — Edits after confirmation

- Show the diff for every memory file
- Apply the edits one file at a time, waiting for confirmation
- Do not rewrite a file wholesale: targeted edits keep the memory history readable
- Watch the size: `AGENTS.md` ≤150 lines, the whole memory no more than roughly
  a quarter of the context window; if it grows, trim `examples.md`

### Step 4 — Revision history

Add an entry to the "Revision history" section of `AGENTS.md`:
date, HEAD commit, what changed in the memory, one line per file.
This is the starting point for the next sync.

### Step 5 — Verification and commit

- Re-read `AGENTS.md`: every memory link points to an existing file
- Every class and library mentioned in the memory exists (open and confirm)
- No contradictions or duplicates between files
- Commit the memory as a separate commit: `memory: sync with <HEAD short sha>`

## Quality control

- Every file changed since the last sync was checked, not a sample
- Every memory edit was confirmed by the user
- `examples.md` contains only code that exists
- The "Revision history" entry was added

## Anti-patterns

- Rebuilding the memory from scratch instead of checking the delta: the user's refinements are lost
- Changing the code to match the memory: code beats memory
- "Refreshing everything just in case" without a diff: the user cannot see what changed
- Running from the middle of a working session with a full context: start in a clean session

## How to automate

The next step after running it by hand is a hook after merging into the main branch
(a CI job or a git hook) that runs this skill in a clean session and opens an MR with
the memory changes. A human approves the MR instead of every single edit.
