# Chapter 10: Prompt Engineering for Agent Harnesses

Every engineer who has worked with LLMs has opinions about prompting. Most of those opinions were formed in the context of single-turn interactions: "write me a poem," "summarize this document," "explain quantum computing." Agent harnesses operate in a fundamentally different regime. The prompts you write are not one-off instructions — they are the *operating system* of an autonomous worker that will execute dozens or hundreds of steps, make judgment calls without your input, and encounter situations you never anticipated.

This chapter is not a generic prompting guide. It is a focused treatment of the prompt engineering techniques that make or break agent systems. We will cover system prompt design, multi-turn agentic workflows, subagent orchestration, state management, the most damaging anti-patterns, and the practical art of writing an effective `AGENTS.md` file.

---

## System Prompt Design

The system prompt is the single most important artifact in your harness. It sets the agent's identity, capabilities, constraints, and behavioral defaults. A poorly written system prompt does not just degrade output quality — it creates *systemic* failures that compound across every step of the agent loop.

### Be Clear and Direct

Here is a litmus test that will save you hours of debugging: **show your system prompt to a colleague who has minimal context on your project. If they would be confused by any instruction, the agent will be too.**

LLMs are sophisticated pattern matchers, but they are not mind readers. Ambiguous instructions produce ambiguous behavior. Consider the difference:

```
Bad:  "Be careful with the database."
Good: "Never execute DELETE or DROP statements against the production database.
       When you need to modify production data, generate a migration script
       and present it for review instead of executing it directly."
```

The first version requires the agent to guess what "careful" means in your context. The second version is unambiguous and actionable. Notice that the good version does three things: it states a prohibition, it provides an alternative action, and it makes the boundary concrete.

**Specificity scales.** When your prompt is vague, every tool call the agent makes carries a small probability of misinterpretation. Over a 50-step task, those small probabilities compound into near-certain failure. When your prompt is specific, each step is more likely to succeed, and the compound effect is reliable task completion.

### Explain Why, Not Just What

Agents encounter novel situations constantly. When they do, they need to *reason about intent*, not just pattern-match against rules. Explaining *why* a behavior matters gives the agent the information it needs to generalize correctly.

```
Weak:   "Always run tests before committing."
Strong: "Always run tests before committing. A broken commit blocks the
         entire team because our CI pipeline runs on every push and a red
         build prevents all merges. The cost of a broken commit is much
         higher than the cost of running tests locally."
```

With the weak version, an agent might skip tests when it "knows" its change is trivial — because the instruction feels like a suggestion. With the strong version, the agent understands that the consequences are severe and asymmetric, making it far less likely to take shortcuts.

This principle extends to architectural constraints as well. Do not just say "the frontend must not import from the backend." Say *why*:

```
"The frontend must not import directly from the backend. These are deployed
as separate services with independent release cycles. A direct import would
create a build-time dependency that prevents independent deployment and would
break our CI/CD pipeline, which builds each service in isolation."
```

Now the agent can reason correctly even about edge cases you did not anticipate, such as whether it is acceptable to share a types file (answer: yes, if it is in a shared package that both services depend on).

### Use XML Tags to Structure Prompts

As system prompts grow beyond a few hundred words, structure becomes essential. XML tags are the most reliable structuring mechanism for LLM prompts because they are unambiguous, nestable, and familiar from the model's training data.

```xml
<system>
  <role>
    You are a senior backend engineer working on a Python FastAPI application.
    You have deep expertise in async programming, PostgreSQL, and REST API design.
  </role>

  <architecture>
    The application follows a layered architecture:
    - Routers (API layer) in src/api/
    - Services (business logic) in src/services/
    - Repositories (data access) in src/repositories/
    - Models (SQLAlchemy) in src/models/

    Dependencies flow downward only: routers -> services -> repositories -> models.
    No layer may import from a layer above it.
  </architecture>

  <constraints>
    - Never modify migration files that have already been applied
    - Always use async database operations
    - All new endpoints must have OpenAPI documentation
    - Test coverage must not decrease
  </constraints>

  <verification>
    After making changes:
    1. Run `make lint` to check formatting and style
    2. Run `make test` to run the test suite
    3. Run `make typecheck` to verify type annotations
    If any step fails, fix the issues before proceeding.
  </verification>
</system>
```

Each content type gets its own tag. This is not just cosmetic — it materially improves the agent's ability to retrieve and apply the right information at the right time. When the agent is deciding whether a particular import is valid, it can "look at" the `<architecture>` section. When it finishes a task, it can "look at" the `<verification>` section. Tags create navigable structure within the prompt.

**Nesting matters.** Use nested tags for hierarchical information:

```xml
<tool name="database_query">
  <purpose>Execute read-only SQL queries against the analytics database</purpose>
  <parameters>
    <param name="query" type="string" required="true">
      The SQL query to execute. Must be a SELECT statement.
    </param>
    <param name="timeout" type="integer" required="false">
      Query timeout in seconds. Default: 30. Maximum: 300.
    </param>
  </parameters>
  <safety>
    This tool connects to the production analytics replica. It is read-only
    by design (the database user has SELECT-only permissions), but be mindful
    of query performance. Avoid full table scans on tables with more than
    10 million rows.
  </safety>
</tool>
```

### Role-Setting Focuses Behavior

The role you assign in the system prompt is not decorative. It calibrates the model's entire distribution of likely behaviors. "You are a senior backend engineer" activates different knowledge, different communication patterns, and different risk tolerances than "You are a helpful assistant."

Effective role-setting has three components:

1. **Expertise domain**: What the agent knows deeply
2. **Behavioral disposition**: How the agent approaches problems
3. **Organizational context**: Where the agent sits in a team

```xml
<role>
You are a senior infrastructure engineer at a fintech company. You have deep
expertise in Kubernetes, Terraform, and AWS. You are methodical and cautious —
you always check the current state of infrastructure before proposing changes,
and you prefer incremental rollouts over big-bang deployments. You work on a
platform team that serves 12 product engineering teams, so your changes affect
many downstream consumers.
</role>
```

This role description does more than set expertise. The phrase "methodical and cautious" primes the agent to check before acting. "Incremental rollouts" biases toward safer deployment strategies. "12 product engineering teams" establishes that blast radius matters.

### Long Context: Structure and Grounding

When your system prompt includes large amounts of reference material — API documentation, style guides, architecture decision records — two techniques become essential.

**Put longform data at the top.** The model's attention patterns favor information at the beginning and end of context. Place large reference documents at the start of the system prompt, before your instructions. This is counterintuitive — most people put instructions first — but it ensures the reference material is available when the instructions refer to it.

```xml
<reference>
  <api_documentation>
    [... hundreds of lines of API docs ...]
  </api_documentation>
</reference>

<instructions>
  When implementing new endpoints, follow the patterns documented in the
  API documentation above. Specifically, all responses must use the
  standard envelope format described in the "Response Format" section.
</instructions>
```

**Ground responses in quotes.** When the agent needs to work with reference material, instruct it to quote relevant passages before reasoning about them. This technique, sometimes called *grounding*, reduces hallucination by forcing the agent to anchor its reasoning in actual source material rather than its parametric memory.

```
When answering questions about the codebase, first quote the relevant code
or documentation, then reason about it. Use the format:

> [quoted passage]

Based on this, [your reasoning].
```

Grounding is especially important when the context window contains multiple documents that might conflict. By quoting, the agent makes its information source explicit, which makes errors easier to catch and correct.

---

## Prompting for Agentic Systems

Single-turn prompting is well understood. Agentic prompting — where the model operates across many turns, maintains state, and makes autonomous decisions — requires different techniques.

### Long-Horizon Reasoning

An agent working on a complex task might operate for dozens of turns before completing. The key insight is that **the first prompt in a context window serves a fundamentally different purpose than subsequent prompts.**

The **initializer prompt** (first message in a context window) sets the stage. It needs to:

- Establish the full context of the task
- Load all relevant state from previous sessions
- Define the agent's plan of action
- Set expectations for verification and completion criteria

```xml
<initializer>
You are continuing work on the user authentication refactor. Here is the
current state:

<progress>
{contents of progress.txt}
</progress>

<remaining_tasks>
{contents of remaining-tasks.json}
</remaining_tasks>

<git_log>
{recent git log showing completed work}
</git_log>

Review the progress file and remaining tasks. Identify the next task to work
on. Before starting, verify that all previously completed work still passes
tests by running `make test`.
</initializer>
```

The **continuation prompt** (used in subsequent turns within the same context window) is leaner. The agent already has context, so the prompt focuses on steering:

```
Good progress on the OAuth integration. Before moving to the next task,
please run the full test suite to verify nothing is broken. If tests pass,
move on to implementing the session management changes described in task 3
of remaining-tasks.json.
```

The initializer prompt is heavy with context because the agent has no memory of previous sessions. The continuation prompt is light because the agent has conversational context from earlier turns.

### Multi-Context Window Workflows

Real-world agent tasks often span multiple context windows. The agent works until its context fills up, then a new session starts with a fresh context window. This creates a *handoff problem*: how does the new session know what the previous session accomplished?

Three mechanisms solve this, each serving a different purpose:

**progress.txt** — An unstructured text file where the agent records what it has done, what it tried that did not work, and any observations that might be useful for future sessions. Think of it as a lab notebook.

```
## Session 3 - 2026-03-22

Completed:
- Implemented OAuth token refresh logic in src/auth/refresh.py
- Added unit tests for token refresh (all passing)
- Fixed a bug where expired tokens were cached

Attempted but abandoned:
- Tried to use the redis cache for token storage, but the TTL semantics
  don't match our refresh token lifecycle. Using PostgreSQL instead.

Notes for next session:
- The session management task (task 3) depends on the token refresh work
  completed in this session. Start by verifying refresh tests still pass.
- There's a flaky test in test_auth_middleware.py::test_concurrent_refresh
  that occasionally fails due to a race condition. Not related to our
  changes — pre-existing issue.
```

**tests.json** or **remaining-tasks.json** — A structured file that tracks what needs to be done. JSON is preferred over Markdown for reasons we will discuss in the State Management section.

```json
{
  "tasks": [
    {
      "id": 1,
      "description": "Implement OAuth provider abstraction",
      "status": "complete",
      "completed_in_session": 1
    },
    {
      "id": 2,
      "description": "Implement token refresh logic",
      "status": "complete",
      "completed_in_session": 3
    },
    {
      "id": 3,
      "description": "Implement session management",
      "status": "pending",
      "depends_on": [2],
      "notes": "See design doc in docs/design-docs/session-management.md"
    }
  ]
}
```

**git log** — The source of truth for what code actually changed. The initializer prompt should include a recent git log so the agent can see the actual commits, not just what the progress file claims happened.

```bash
git log --oneline --since="2 days ago"
```

These three mechanisms are complementary. The progress file provides narrative context ("we tried X and it did not work"). The task file provides structured tracking ("task 3 is next"). The git log provides ground truth ("these files actually changed").

### Run Integration Tests Before New Features

This is a specific prompting instruction that deserves its own callout because it prevents one of the most common failure modes in long-running agent work: **building on a broken foundation.**

```
Before starting any new feature work, run the integration test suite:
  make integration-test

If any tests fail, diagnose and fix them before proceeding. Do not start
new work on top of a broken test suite — the failures will compound and
become much harder to debug later.
```

This instruction seems obvious, but without it, agents frequently charge ahead with new feature work without verifying that previous work is still intact. The result is a cascade of failures that wastes the entire context window.

### Provide Verification Tools

Agents are more reliable when they can verify their own work. The most powerful verification tools are:

- **Linters and formatters**: Catch style and structural issues immediately
- **Type checkers**: Catch type errors before runtime
- **Test runners**: Verify functional correctness
- **Visual verification tools**: For UI work, tools like Playwright MCP or screenshot-comparison tools let the agent see what it built

The key prompting technique is to **make verification a mandatory step, not an optional one**:

```
After implementing any UI change:
1. Run `npm run build` to verify the build succeeds
2. Use the Playwright MCP to take a screenshot of the affected page
3. Compare the screenshot against the design spec in docs/design-docs/
4. If the implementation does not match the spec, iterate until it does

Do not consider a UI task complete until you have visually verified it.
```

Without explicit instructions to verify, agents tend to write code, see no errors in the output, and assume everything is fine. Visual verification catches the class of bugs where the code is technically correct but the result does not match expectations.

### Balancing Autonomy and Safety

The fundamental tension in agent prompting is between autonomy (letting the agent work without interruption) and safety (preventing the agent from causing damage). The resolution is a simple framework: **consider reversibility and potential impact.**

```xml
<safety_framework>
Before taking any action, assess it on two dimensions:

1. REVERSIBILITY: Can this action be easily undone?
   - Reversible: creating a file, making a git commit, writing to a log
   - Irreversible: deleting a production database, sending an email,
     publishing a package

2. IMPACT: What is the blast radius if this goes wrong?
   - Low impact: affects only the local development environment
   - High impact: affects production, other team members, or external users

Actions that are REVERSIBLE and LOW IMPACT: proceed without confirmation.
Actions that are IRREVERSIBLE or HIGH IMPACT: describe what you plan to do
and ask for confirmation before proceeding.

Examples:
- Creating a new file in the repo: reversible + low impact -> proceed
- Running `git push --force`: irreversible + high impact -> ask first
- Modifying a CI configuration: reversible + medium impact -> proceed but
  note the change prominently
- Deleting a database table: irreversible + high impact -> ask first and
  provide a rollback plan
</safety_framework>
```

This framework gives the agent a decision procedure rather than a list of rules. A list of rules will always have gaps — there is always some action you did not think to prohibit. A decision procedure lets the agent reason about novel situations correctly.

**Confirm before destructive operations.** This deserves special emphasis. Instruct the agent to always confirm before:

- Deleting files or directories
- Force-pushing to shared branches
- Modifying production configurations
- Sending communications to external parties
- Executing database migrations

The cost of asking for confirmation is a few seconds of delay. The cost of a destructive operation performed in error can be catastrophic.

---

## Subagent Orchestration

Modern agent systems can delegate subtasks to subagents — separate agent instances that run in their own context windows. Claude 4.6 is particularly good at recognizing when a task would benefit from delegation, but effective orchestration still requires careful prompting.

### When Delegation Helps

Delegation is valuable when:

- A subtask is **self-contained** and can be fully specified without extensive context from the parent task
- The subtask is **parallelizable** — it does not depend on the outcome of other concurrent work
- The subtask would consume significant context window space that the parent agent needs for other reasoning
- The subtask requires **different expertise** than the parent task (e.g., the parent is doing backend work but needs a CSS fix)

### When to Do Work Inline

Keep work inline when:

- The task is **small** (fewer than 5-10 tool calls)
- The task requires **deep context** from the current conversation that would be expensive to transfer
- The task's output will be **immediately used** in the next step of the parent's reasoning
- **Coordination overhead** would exceed the time saved by parallelism

### Watching for Overuse

A common failure mode is **subagent proliferation**: the parent agent spawns subagents for trivially small tasks, spending more time on delegation overhead than the tasks themselves would take.

```
When considering whether to delegate to a subagent, apply this heuristic:
if the task would take you fewer than 5 tool calls to complete, do it
yourself. Delegation has overhead — writing the task description, waiting
for the subagent, reviewing its output — that is only worthwhile for
substantial tasks.

When you do delegate, provide the subagent with:
1. A clear, self-contained task description
2. All necessary context (file paths, relevant code snippets, constraints)
3. Explicit success criteria
4. Instructions to commit its work to a branch when done

Do not delegate tasks that require judgment calls about the overall
architecture or design direction — those decisions should stay with you.
```

The key heuristic is: **delegation should save net time.** If the overhead of specifying, launching, and reviewing a subagent's work exceeds the time you would spend doing the work yourself, delegation is a net loss.

---

## State Management

Agent systems need to maintain state across turns and across sessions. The choice of state management format is not cosmetic — it materially affects agent reliability.

### Structured Formats (JSON) for State Data

Use JSON for any data that needs to be read, updated, and written back programmatically. Task lists, feature tracking, configuration state, and test results all belong in JSON.

```json
{
  "feature": "user-authentication",
  "status": "in_progress",
  "tasks": [
    {"id": 1, "name": "OAuth provider abstraction", "status": "complete"},
    {"id": 2, "name": "Token refresh logic", "status": "complete"},
    {"id": 3, "name": "Session management", "status": "in_progress"},
    {"id": 4, "name": "Password reset flow", "status": "pending"}
  ],
  "blockers": [],
  "last_updated": "2026-03-22T14:30:00Z"
}
```

### Why JSON Over Markdown for Feature Tracking

This is a point where practical experience diverges from intuition. Many engineers instinctively reach for Markdown because it is human-readable and because LLMs generate fluent Markdown. But for state data that agents read and write, **JSON is significantly more reliable than Markdown.**

The reason is behavioral, not technical. When an agent reads a Markdown file, it treats the content as *text to be edited*. The model's training on text-editing tasks makes it likely to "improve" the Markdown — rewording descriptions, reorganizing sections, adding formatting. These well-intentioned edits corrupt state data.

JSON, by contrast, triggers a different behavioral mode. The model treats JSON as *data to be updated*. It is far less likely to reword a JSON value, reorder fields for aesthetics, or add commentary. The structural rigidity of JSON acts as a behavioral guardrail.

```markdown
<!-- DON'T: Markdown feature tracking -->
## Features
- [x] OAuth provider abstraction - Complete
- [x] Token refresh logic - Done, including edge cases for expired tokens
- [ ] Session management - Working on this now, started with the database schema
- [ ] Password reset flow

<!-- The agent will "helpfully" rewrite descriptions, merge items, add notes -->
```

```json
// DO: JSON feature tracking
{
  "tasks": [
    {"id": 1, "name": "OAuth provider abstraction", "status": "complete"},
    {"id": 2, "name": "Token refresh logic", "status": "complete"},
    {"id": 3, "name": "Session management", "status": "in_progress"},
    {"id": 4, "name": "Password reset flow", "status": "pending"}
  ]
}
// The agent will update status fields and leave everything else alone
```

This difference is subtle but consequential. In long-running agent workflows spanning many sessions, Markdown-based state files tend to drift — descriptions get reworded, completed items get merged or summarized, and information is quietly lost. JSON-based state files tend to be updated cleanly because the format discourages creative editing.

### Unstructured Text for Progress Notes

Progress notes are the opposite case. Here, you *want* the agent to write freely. Observations, debugging narratives, hypotheses about why something failed — these are best captured as unstructured text.

```
progress.txt is your working notebook. After each significant action, append
a brief note describing what you did and why. Include:
- What you attempted
- Whether it succeeded or failed
- If it failed, what you learned from the failure
- Any observations that might be useful for future sessions

Do not edit or delete previous entries. Always append to the end of the file.
```

The instruction to "never edit previous entries" is important. Without it, agents sometimes rewrite the progress file to be "cleaner," destroying the historical record of attempts and failures that is invaluable for debugging.

### Git for Checkpoints

Git commits serve as checkpoints in long-running agent work. Each commit captures a known-good state that can be returned to if subsequent work goes wrong.

```
After completing each task:
1. Run all tests to verify correctness
2. Stage all relevant changes (but not temporary files, logs, or state files
   that are tracked separately)
3. Create a commit with a descriptive message following the conventional
   commits format: type(scope): description
4. If tests fail after a commit, you can `git stash` your current changes,
   verify the committed state is clean, then `git stash pop` to continue
   debugging

Never amend a commit that has already been pushed. Create a new commit instead.
```

The combination of JSON state files, unstructured progress notes, and git commits gives you three layers of state management, each serving a different purpose: structured tracking, narrative context, and code snapshots.

---

## Anti-Patterns

Every experienced harness engineer has encountered these failure modes. They are pernicious because they are not bugs — the agent appears to be working, and only careful observation reveals that its behavior is subtly wrong.

### Overthinking (Over-Prompting)

**Symptom**: Your system prompt is thousands of words long, filled with detailed instructions for every conceivable situation. The agent's behavior is inconsistent because it is trying to follow too many rules simultaneously.

**Root cause**: Treating the system prompt like a legal contract instead of a set of principles. When you add a rule for every edge case, the rules inevitably conflict, and the agent's behavior becomes unpredictable.

**Fix**: Replace blanket defaults with targeted instructions. Instead of a 50-rule system prompt, use a 10-rule system prompt with the most important principles, and add specific instructions only when needed in task-level prompts.

```
Before (over-prompted):
"Always use TypeScript. Always use strict mode. Always use const instead
of let. Always use arrow functions. Always use template literals instead
of string concatenation. Always use async/await instead of promises.
Always use destructuring. Always use the optional chaining operator.
Always..."

After (targeted):
"Follow the TypeScript style guide in docs/STYLE.md. If you are unsure
about a style question, check the existing code in the same directory
for precedent."
```

The "after" version is not just shorter — it is more *robust*. It handles cases the "before" version did not enumerate, and it will not conflict with itself.

**How to diagnose**: If you find yourself adding prompting rules in response to specific agent failures, step back and ask: "Is this failure caused by a missing rule, or by too many rules creating confusion?" More often than you expect, the answer is the latter.

### Overeagerness / Overengineering

**Symptom**: The agent builds elaborate abstractions, adds features that were not requested, creates helper classes "in case they are needed later," or refactors working code that it was not asked to touch.

**Root cause**: The model's training on high-quality code repositories creates a bias toward "doing things properly," which manifests as adding unnecessary complexity.

**Fix**: Add explicit guidance to keep solutions minimal.

```
Implement the minimum viable solution that satisfies the requirements.
Do not:
- Add features that were not requested
- Create abstractions "for future use"
- Refactor code that is not related to the current task
- Add utility functions that are not immediately needed

If you think additional work would be valuable, note it as a suggestion
in the progress file rather than implementing it. Let the user decide
whether to pursue it.
```

**The keyword is "minimal."** Agents respond well to this word because it gives them a clear optimization target. Without it, the implicit optimization target is "impressive" or "thorough," which leads to overengineering.

### Focusing on Passing Tests (Not Solving Problems)

**Symptom**: The agent writes code that makes failing tests pass but does not actually solve the underlying problem. Common manifestations include hard-coding expected values, adding special cases for test inputs, or catching and silently swallowing exceptions.

**Root cause**: The agent treats test passage as the terminal goal rather than as a signal of correctness. When tests fail, the shortest path to making them pass is often to game the test rather than fix the code.

**Fix**: Instruct the agent to write general-purpose solutions.

```
When fixing failing tests:
1. First understand WHY the test is failing. Read the test code to
   understand what behavior it is verifying.
2. Fix the underlying code to handle the general case correctly. The fix
   should work for any valid input, not just the specific inputs in the
   test.
3. Never hard-code expected values from test cases into the implementation.
4. Never add special-case handling for specific test inputs.
5. If a test seems wrong (testing for incorrect behavior), raise that
   concern rather than contorting the code to match it.
```

**A useful heuristic to include in your prompt**: "If someone changed the test's input values, would your implementation still produce correct results?" This forces the agent to think about generality rather than test-specific correctness.

### Hallucinations

**Symptom**: The agent confidently describes code that does not exist, invents API endpoints, references documentation that was never written, or claims that a function behaves in a way that contradicts its actual implementation.

**Root cause**: The model's parametric memory contains patterns from its training data. When asked about code it has not read, it generates plausible-sounding but fictional descriptions based on these patterns.

**Fix**: Instruct the agent to investigate before answering and never speculate about code it has not opened.

```
CRITICAL: Never make claims about code you have not read in this session.
If asked about a file, function, or module:
1. Open the file and read the relevant code
2. Only then describe what it does, based on the actual source code
3. If you cannot find the file or function, say so explicitly rather
   than guessing what it might contain

Do not speculate about:
- What a function "probably" does based on its name
- What arguments an API "likely" accepts
- How a module "typically" behaves in codebases like this

When in doubt, read the code. When you have read the code, quote it.
```

The phrase "in this session" is important. It tells the agent that even its knowledge of this codebase from training data is not trustworthy — only what it has directly read in the current context window counts as reliable information.

---

## The AGENTS.md / CLAUDE.md File

The `AGENTS.md` file (or `CLAUDE.md`, depending on your tooling) is the entry point for any agent working in your repository. It is not a comprehensive manual — it is a **table of contents** that tells the agent where to find the information it needs.

### Keep It Around 100 Lines

This constraint is intentional and important. A 100-line file is short enough to be loaded into every context window without consuming significant space, but long enough to provide meaningful guidance. Think of it as the `README.md` of your agent harness — it should orient, not overwhelm.

If your `AGENTS.md` is growing beyond 100 lines, that is a signal that you need to move content to referenced documents.

### What to Include

The `AGENTS.md` file should contain:

1. **Project overview** (5-10 lines): What this project is, what technology stack it uses, and any high-level constraints.

2. **Repository structure** (10-15 lines): Where the important directories are and what they contain. Not an exhaustive directory listing — just the key landmarks.

3. **Key commands** (10-15 lines): How to build, test, lint, and deploy. The exact commands, not "see the Makefile."

4. **Architecture rules** (10-15 lines): The most important architectural constraints — dependency rules, naming conventions, forbidden patterns. Reference `ARCHITECTURE.md` for the full picture.

5. **Common tasks** (15-20 lines): Step-by-step instructions for the tasks agents most commonly perform: adding a new API endpoint, creating a new component, writing a migration.

6. **Pointers to detailed docs** (5-10 lines): Links to design docs, style guides, security guidelines, and other reference material in the `docs/` directory.

7. **Safety constraints** (5-10 lines): What the agent must never do. Explicit prohibitions.

Here is a realistic example:

```markdown
# AGENTS.md

## Project Overview
TaskFlow is a project management API built with Python 3.12, FastAPI, and
PostgreSQL. It serves the TaskFlow web and mobile clients. All code is in
the `src/` directory. Tests are in `tests/`.

## Key Commands
- `make dev` — Start the development server with hot reload
- `make test` — Run the full test suite (pytest)
- `make test-unit` — Run unit tests only
- `make test-integration` — Run integration tests (requires running database)
- `make lint` — Run ruff linter and formatter
- `make typecheck` — Run mypy type checker
- `make migrate` — Apply pending database migrations
- `make migration NAME="description"` — Generate a new migration

## Architecture
See ARCHITECTURE.md for the full architecture guide. Key rules:
- Layered architecture: routers -> services -> repositories -> models
- Dependencies flow downward only
- All database access goes through repositories, never direct SQL in services
- All endpoints require authentication unless explicitly marked public

## Common Tasks
### Adding a new endpoint
1. Create or update the router in `src/api/`
2. Create or update the service in `src/services/`
3. Create or update the repository in `src/repositories/`
4. Add tests in `tests/` mirroring the source structure
5. Run `make lint && make typecheck && make test`

### Creating a database migration
1. Modify the model in `src/models/`
2. Run `make migration NAME="description_of_change"`
3. Review the generated migration in `migrations/versions/`
4. Run `make migrate` to apply it
5. Run `make test-integration` to verify

## Documentation
- `docs/DESIGN.md` — Design principles and patterns
- `docs/FRONTEND.md` — Frontend integration guide
- `docs/SECURITY.md` — Security requirements and authentication
- `docs/design-docs/` — Feature-specific design documents
- `docs/references/` — External API docs and specifications

## Constraints
- Never modify migrations that have been applied to production
- Never bypass authentication checks
- Never log sensitive data (passwords, tokens, PII)
- Always run the full test suite before committing
```

### What to Put in Referenced Documents

The `AGENTS.md` file points to detailed documents in the `docs/` directory. Each of these documents should be comprehensive and self-contained:

- **`ARCHITECTURE.md`**: Full architecture description, dependency diagrams, module responsibilities, data flow, deployment architecture. This can be hundreds of lines.

- **`docs/DESIGN.md`**: Design principles, patterns used in the codebase, rationale for key decisions. Reference this when agents need to make design choices.

- **`docs/SECURITY.md`**: Authentication and authorization model, data handling policies, security requirements for new features.

- **`docs/design-docs/`**: One document per major feature or system. These provide deep context for agents working on specific areas.

- **`docs/exec-plans/`**: Execution plans for multi-step projects. These help agents understand the sequencing and dependencies of large initiatives.

- **`docs/references/`**: External documentation — API specs for third-party services, protocol documentation, etc.

### Structuring the docs/ Directory

```
docs/
  DESIGN.md              # Design principles and patterns
  FRONTEND.md            # Frontend integration guide
  SECURITY.md            # Security requirements
  design-docs/
    user-authentication.md
    notification-system.md
    payment-processing.md
  exec-plans/
    q1-auth-refactor.md
    q2-performance.md
  product-specs/
    user-stories.md
    api-requirements.md
  references/
    stripe-api.md
    sendgrid-integration.md
```

Each document should have a clear scope. A design doc covers one feature. An exec plan covers one project. A reference doc covers one external system. This granularity means agents only need to load the specific documents relevant to their current task, keeping context windows efficient.

### Maintenance: Preventing Staleness

An `AGENTS.md` file that is out of date is worse than no file at all — it actively misleads agents. Maintenance requires a deliberate process:

**Who updates it**: The engineer who makes changes that affect agent workflows. If you change a key command, update `AGENTS.md`. If you restructure a directory, update `AGENTS.md`. This should be part of the same PR as the change itself.

**How often**: Continuously, as part of normal development. Do not batch `AGENTS.md` updates — they will never happen. Instead, make updating the agent docs a checklist item in your PR template:

```markdown
## PR Checklist
- [ ] Tests pass
- [ ] Linter passes
- [ ] AGENTS.md updated (if workflow changes)
- [ ] Architecture docs updated (if structural changes)
```

**How to prevent staleness**: The most effective technique is to make the agent *use* the documentation constantly. When agents encounter outdated instructions, they fail in visible ways — broken commands, missing files, wrong directory paths. These failures surface staleness faster than any review process.

You can also add a periodic verification task: once a month, have an agent read the `AGENTS.md` file and try to execute every command listed in it. Any failures reveal stale documentation.

**A pragmatic approach to versioning**: Some teams timestamp their `AGENTS.md` with a `Last verified:` date at the top. This creates social pressure to keep it current — a "Last verified: 6 months ago" date is a clear signal that the file needs attention.

```markdown
# AGENTS.md
<!-- Last verified: 2026-03-15 -->
<!-- Maintainers: @alice, @bob -->
```

---

## Putting It All Together

Effective prompt engineering for agent harnesses is not about clever tricks or magical incantations. It is about clear communication, structured information, and thoughtful constraints. The principles in this chapter can be summarized in five rules:

1. **Be specific and explain why.** Vague instructions produce vague behavior. Explained instructions generalize to novel situations.

2. **Structure with XML tags.** Large prompts need navigable structure. Tags provide it without ambiguity.

3. **Use the right format for the right data.** JSON for state, unstructured text for narratives, git for code snapshots.

4. **Balance autonomy and safety through reversibility.** Give agents a decision framework, not an exhaustive rule list.

5. **Keep the entry point concise and the references comprehensive.** The `AGENTS.md` file is a table of contents, not an encyclopedia.

These principles compound. A well-structured system prompt with clear constraints produces an agent that makes better tool calls, which produces better intermediate results, which reduces the need for human intervention, which makes the entire system more efficient. The upfront investment in prompt engineering pays dividends across every agent run.
