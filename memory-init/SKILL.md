---
name: memory-init
description: "Initializes the agent's long-term memory for a Java/Spring backend autotest repository: creates AGENTS.md and memory/ based on a deep audit of the code. Use when the user says 'initialize memory', 'set yourself up for this repo', 'first time in this repo', when the agent is connected to a new autotest project for the first time, or when the repository has no AGENTS.md."
---

# memory-init: initializing the agent's memory

This skill runs a structured audit of a Java/Spring autotest repository and builds
a reusable knowledge base for the agent: rules, patterns, reference examples, workflow.

## When to use this skill

- First connection to a Java + Spring autotest repository
- AGENTS.md is missing, or the memory is stale after a refactoring
- The team needs a single standard that is committed to git

Do not activate if: AGENTS.md already exists and is up to date (use memory-update
instead), or the repository is not an autotest project.

## Target memory structure

```
├── AGENTS.md
└── memory/
    ├── project-overview.md
    ├── architecture.md
    ├── conventions.md
    ├── test-patterns.md
    ├── frameworks-stack.md
    ├── workflow.md
    ├── examples.md
    └── common-pitfalls.md
```

AGENTS.md is a lightweight root file (≤150 lines): only links and hard rules.
The agent reads it at the start of every session (OpenCode, Claude Code, Codex CLI
and similar tools pick up AGENTS.md automatically; if your agent expects a different
file name, add a symlink next to it). Details live in the memory/ files it links to.

Two rules on top of the structure:
- Code beats memory. If they diverge, fix the memory, not the code.
- The whole memory must fit into roughly a quarter of your model's context window.
  If it is bigger, trim examples.md, not the rules.

## Workflow

Finish every step with a short summary and an explicit question: "Ready to move on
to step N+1?". Never proceed without the user's confirmation.

### Step 1 — Repository reconnaissance

Goal: gather facts without changing any files.

Actions:
- Build the directory tree 3 levels deep, excluding target, build, .idea, .git
- Read the root configs: pom.xml / build.gradle(.kts), settings.xml,
  application*.yml/properties, README*, Makefile, Dockerfile, docker-compose*,
  .gitlab-ci.yml, junit-platform.properties
- Identify and record the stack:
  - test framework (JUnit 4/5, TestNG)
  - assertion library (AssertJ, Hamcrest, Truth)
  - HTTP client (RestAssured, WebTestClient, Feign, OkHttp)
  - database access (JdbcTemplate, JPA, Testcontainers, Liquibase/Flyway)
  - Spring context (@SpringBootTest, @ContextConfiguration, profiles)
  - mocks/stubs (WireMock, MockServer, Mockito)
  - reporting (Allure, ReportPortal)
  - parameterization and data providers
  - lifecycle hooks (@BeforeAll/@BeforeEach, listeners)
  - test data builders / factories / fixtures
- Find the base classes (AbstractTest, BaseApiTest, BaseDbTest) and describe the hierarchy
- Find 3–5 typical test classes and describe their structure
- Record the package structure: tests, steps, clients, models, config, helpers
- Record naming conventions (*Test/*IT/*Tests, methods, packages)
- Understand environment parameterization and where secrets live

Create no files at this step. If something is not found, say "not found" explicitly.

Output: a structured markdown report in the chat.

### Step 2 — Clarifying questions

After the report, ask 10–15 clarifying questions wherever there is uncertainty.
Make no assumptions. Wait for the answers before moving to step 3.

### Step 3 — Create AGENTS.md

Create `AGENTS.md` strictly in this structure:

- Purpose (one sentence)
- Stack (bullet list)
- Memory map: links to the memory/ files with a one-line description each
- Hard rules (Always): read conventions.md before writing code, check against
  test-patterns.md and examples.md, never change base classes without asking,
  no new dependencies without approval, run the checks before committing,
  read the Jira ticket via MCP before starting a task, branch and commit format
- Task workflow (7 steps, see the write-tests-by-task skill)
- Stop signals: ambiguous acceptance criteria, a new base class, new test data,
  existing tests failing
- MCP servers: gitlab, jira, confluence, with the purpose of each
- Hallucination guard: never write code that references non-existent entities
- Priority rule: code beats memory; on divergence update the memory (memory-update skill)

Limit: ≤150 lines. Move details into memory/.

### Step 4 — Create memory/ one file at a time

Create the files one per iteration, showing the content to the user and waiting
for edits before moving to the next one.

**4.1 project-overview.md** — what is under test, business context, environments,
entry points (local run, CI, tags), module structure, external dependencies and
how they are emulated.

**4.2 architecture.md** — layers (tests → steps → clients → models → config),
base class hierarchy, Spring context initialization, TestConfiguration and
@MockBean, test lifecycle, Testcontainers (reusable, network, healthcheck),
database migrations, parallel execution, logging and Allure attachments. Use
real class names with paths.

**4.3 conventions.md** — naming (classes, methods, DisplayName, packages), code style
(import order, Lombok, final, method length), test conventions (assert vs
SoftAssertions, @Tag, @Disabled with a ticket, test data location), prohibitions
(Thread.sleep → Awaitility, System.out → SLF4J, hardcoded URLs and tokens).
Back every rule with an example and a file path.

**4.4 test-patterns.md** — step-by-step templates for the test types in the repo:
REST positive, REST negative, database test, Kafka/event-driven, WireMock stubs,
parameterized test. For each: when to apply, Arrange/Act/Assert skeleton,
base class, clients, test data, assertions, link to the reference example.

**4.5 frameworks-stack.md** — a table:
| Area | Library | Version | Where used | Notes |
For the key libraries (RestAssured, AssertJ, Testcontainers, WireMock,
Awaitility, Allure, JUnit 5) add an "Idioms in our project" section with examples.
Also: versions of Java, Spring Boot, the build tool and plugins.

**4.6 workflow.md** — the detailed process for working on a task. 7 stages:
receiving the task (MCP Jira + Confluence), analysis, implementation, local
validation, MR via MCP GitLab, Jira update. After every stage a sync point with the user.

**4.7 examples.md** — 3–5 REFERENCE tests from the repo, copied as-is.
For each: path, type, full code, a commentary. This is the main guide when
generating new tests.

**4.8 common-pitfalls.md** — typical mistakes IN THE CONTEXT OF THIS project:
parallel-run problems, sources of flakiness, misuse of base class methods, forgotten
cleanup, Testcontainers gotchas, Spring profile quirks, known flaky tests. Every item:
problem → why → the right way, with an example.

### Step 5 — Self-check

After all files are created:
- Re-read AGENTS.md and every memory/*.md
- Check there are no contradictions between the files
- Check that every link in AGENTS.md points to an existing file
- Check that every class and library mentioned actually exists in the project
  (open the files to confirm)
- Check for duplication (one topic lives in one place)
- Write a report: what was found, what was fixed, what is left as TODO
- Propose 5–7 clarifying questions to the user

### Step 6 — Dry run on a real task

Without writing any code into the repo: take a real or hypothetical Jira ticket and
walk through the full workflow from memory/workflow.md, narrating every action and
referring to specific items in the memory. The user judges how well the rules are followed.

### Step 7 — Final edits

Based on the dry run:
- Add the places where the agent went wrong to common-pitfalls.md
- Refine examples.md if the style became clearer
- Extend workflow.md with the steps the user spelled out
- Add a "Revision history" section to AGENTS.md: the memory changelog
- Show the diff and commit

## Quality control

Before finishing, make sure that:
- AGENTS.md is ≤150 lines and contains only links and hard rules
- Every memory/ file refers to concrete paths in the repo
- examples.md contains ≥3 real tests from the repo
- There are no contradictions between the files
- Every class and library mentioned really exists
- The user has confirmed every step

## Anti-patterns

- Inventing class or library names: open the file and verify
- One monolithic memory file: agents lose their way in large documents
- Duplicating rules across files: one topic lives in one place
- Writing code into the repo during initialization: this skill only creates memory
- Skipping sync points with the user: every step requires confirmation
- Time-sensitive wording in rules: describe invariants, not dates
