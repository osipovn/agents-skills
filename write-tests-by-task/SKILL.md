---
name: write-tests-by-task
description: "Runs the full cycle of writing Java/Spring backend autotests from a Jira ticket to a Merge Request in GitLab. Use when the user gives a task key in the PROJECT-NUMBER format, says 'write tests for QA-1234', 'take the task', 'autotests for the ticket', 'cover with tests', or attaches a link to a Jira issue. Relies on memory/ and the gitlab, jira and confluence MCP servers."
---

# write-tests-by-task: autotests from a tracker task

This skill implements the end-to-end process: read the Jira ticket → analyze the code →
write autotests following the existing patterns → create an MR in GitLab →
update the task status.

## When to use this skill

- A Jira task key is given and autotests need to be added
- Test coverage for a new API endpoint or a scenario described in the ticket
- Extending an existing test class with new cases from the acceptance criteria in Jira

Do not activate for: refactoring without a task, writing tests without a description
of the requirements, changes outside the autotest directories.

## Preconditions

- `AGENTS.md` and the `memory/` directory exist
- The gitlab, jira and confluence MCP servers are connected
- The user has a Jira task key

If the preconditions are not met, tell the user and offer to run `memory-init`.

## Workflow

After every stage a short summary and an explicit confirmation from the user
before moving to the next one.

### Step 1 — Load the memory

- Read `AGENTS.md` and every file in `memory/`
- Confirm in the chat: "Memory loaded. Files: [list]."
- If some files are missing, stop and offer to initialize the memory

### Step 2 — Get the task

Via MCP Jira, pull:
- summary, description, acceptance criteria
- components, labels, epic
- attached links (Confluence, other tickets)

Via MCP Confluence, read the linked specification pages.

Summarize the task to the user in 5–10 lines:
- What needs to be covered
- Which acceptance criteria were found
- Which related documents were found
- What is missing to write the tests

Wait for confirmation or clarifications.

### Step 3 — Analyze the codebase

- Find relevant existing tests by component/endpoint/domain
- Decide: extend an existing class or create a new one
- Check `memory/test-patterns.md`: which templates apply
- Check `memory/examples.md`: which reference is closest

Draft a plan: the list of test cases with DisplayName and a short description of
the behavior under test. Name the target files. The plan must include negative
cases (invalid data, missing permissions, non-existent entities): without them
only positive scenarios get written.

Wait for the plan to be approved.

Steps 1–3 run in the agent's planning mode (in OpenCode it is the Plan mode,
toggled with Tab; in Claude Code it is plan mode): reading and proposing is allowed,
changing files is not. Switch to build mode only after the plan is approved.

### Step 4 — Branch and implementation

- MCP GitLab: create a branch from the current main/master in the format from
  `memory/workflow.md` (usually `feature/<JIRA-KEY>-<slug>`)
- Write the tests strictly by `memory/test-patterns.md`, copying the structure
  from `memory/examples.md`
- Follow `memory/conventions.md` (naming, style, prohibitions)
- Check `memory/common-pitfalls.md` to avoid the typical mistakes

If a case is not covered by the existing patterns, or a new client, base class or
dependency is needed, stop and ask the user. Make no architectural decisions alone.

### Step 5 — Local validation

- Run only the new tests (for example `mvn test -Dtest=NewTest`
  or `./gradlew test --tests "*.NewTest"`)
- Run the formatter and linter if the project has them
- Paste the command output into the chat
- On failures, analyze the cause and fix it without suppressing tests and without
  weakening assertions: fix test data preparation and waits, not the asserts
- A green run counts only against the real environment. If the service or an
  integration is unreachable and the tests ran against a stub, a mock or a fake,
  do not report "tests pass": state explicitly what was substituted and offer the
  user to sort out the access (URL, TLS, secrets). A failure against the real
  environment is a reason to find the cause (service logs), not to bend the test
  to the response
- Self-check every new test (mutation testing by hand): temporarily break the logic
  or condition under test (comment out a branch, change a formula, flip a boundary
  comparison), make sure the test fails, restore the code. If the test did not fail,
  rewrite the assertion. Report the result for every test in the chat

### Step 6 — Commit and Merge Request

- Commit message in the format `<JIRA-KEY>: <imperative summary>`
- MCP GitLab: create an MR using the repository template
- In the MR description include:
  - Link to the Jira task
  - What is covered (the list of test cases)
  - How to run locally
  - Readiness checklist
- Assign reviewers from CODEOWNERS if the file is present

### Step 7 — Update Jira

- MCP Jira: move the task to "In Review" (or the status the team uses)
- Add a comment with the MR link

## Stop points

Explicit confirmation from the user is required before:
- creating a branch
- creating an MR
- any change outside the autotest directories
- adding a new dependency
- modifying base classes or shared utilities

## Quality control

Before finishing, make sure that:
- Every acceptance criterion from Jira is covered by tests or explicitly deferred with a reason
- The tests follow the patterns from memory/test-patterns.md
- Naming, style and prohibitions from memory/conventions.md are respected
- The tests pass against the real environment; if the run was against a stub, this is stated explicitly
- Every new test fails when the logic under test is broken (self-check done)
- The MR contains all required links and the description
- The Jira task status is updated

## Anti-patterns

- Writing tests without reading memory/: style consistency is lost
- Adding Thread.sleep instead of Awaitility: a conventions violation
- Creating your own helpers when the project already has them
- Hardcoding URLs, tokens or test data in the test code
- Suppressing failing tests with @Disabled without a ticket link
- Creating an MR without running the tests locally
- Making architectural decisions without the user's confirmation
- Ignoring CODEOWNERS when assigning reviewers
