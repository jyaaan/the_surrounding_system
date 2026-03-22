# Chapter 7: Feedback Loops That Replace Human QA

In every engineering organization that has adopted AI-assisted coding at scale, the same bottleneck shift occurs. At first, the constraint is *writing* code — the agent accelerates this dramatically. Within weeks, the constraint moves to *validating* code. Engineers find themselves spending more time reviewing agent-generated pull requests than they previously spent writing the code themselves. The agent is prolific but not self-correcting. It produces output at machine speed, but that output still requires human-speed review.

This chapter addresses the question: can we design feedback loops tight enough that the agent validates its own work to a standard that makes human QA unnecessary — or at least reduces it to a spot-check rather than a line-by-line review?

The answer, it turns out, is a qualified yes. But getting there requires making the application *legible* to the agent in ways that most codebases are not, building evaluator-optimizer loops that catch the errors humans would catch, and designing evaluation systems rigorous enough to give you confidence that the whole apparatus actually works.

---

## 7.1 Agent Legibility: Making the Application Visible

The core problem with an agent that writes code but cannot validate it is not intelligence — it is *perception*. The agent can reason about code, but it cannot see the running application. It writes a CSS rule but cannot see whether the button is rendered correctly. It adds an API endpoint but cannot observe whether the response reaches the client. It fixes a race condition but cannot watch the system under load to verify the fix.

Making the application *legible* to the agent means giving it the perceptual capabilities to observe the running system the way a human developer would — through a browser, through logs, through metrics dashboards, through the debugger.

### 7.1.1 Chrome DevTools Protocol as Agent Eyes

The most powerful approach for web applications is wiring the Chrome DevTools Protocol (CDP) directly into the agent's runtime. CDP is the protocol that Chrome's developer tools use to communicate with the browser. It supports everything from DOM inspection to network monitoring to JavaScript evaluation to screenshot capture. By exposing CDP capabilities to the agent through its tool interface, you give the agent a direct window into the running application.

The architecture works as follows:

**Launch an isolated app instance per git worktree.** Each agent session — or, in more sophisticated setups, each feature branch — gets its own running instance of the application, connected to its own database and served on its own port. This isolation prevents agents working on different features from interfering with each other and ensures that the agent's observations reflect the state of *its* code, not some other session's.

```
worktree/feature-auth/     → app on :3001, db auth_dev
worktree/feature-nav/      → app on :3002, db nav_dev
worktree/feature-search/   → app on :3003, db search_dev
```

**Snapshot the DOM before and after UI interactions.** When the agent clicks a button, submits a form, or navigates to a page, the harness automatically captures a serialized snapshot of the DOM before and after the interaction. The agent can then diff these snapshots to verify that the expected changes occurred. Did the success message appear? Did the error state render? Did the navigation update? These questions are answered by DOM evidence, not by inspecting source code.

```
BEFORE click("#submit-btn"):
  <form id="register-form">
    <input name="email" value="test@example.com" />
    <button id="submit-btn">Register</button>
  </form>

AFTER click("#submit-btn"):
  <div class="alert alert-success">
    Registration successful! Check your email.
  </div>
```

**Capture screenshots for visual regression.** The agent takes a screenshot after each significant interaction and compares it (or has it compared by a vision model) against a reference image or against general expectations. This catches visual bugs that DOM inspection alone misses: overlapping elements, incorrect colors, missing icons, broken layouts at specific viewport widths.

**Query runtime logs via LogQL and metrics via PromQL.** For backend verification, the agent needs access to the application's logs and metrics. By exposing a log aggregation system (like Loki, queried via LogQL) and a metrics system (like Prometheus, queried via PromQL) as agent tools, you enable the agent to verify backend behavior empirically:

```
# Did the registration endpoint log a successful user creation?
logql: {app="myapp"} |= "user_created" | json | email="test@example.com"

# Did the response time stay under 200ms?
promql: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{handler="/api/register"}[5m]))
```

**The fix-restart-revalidate loop.** With all of these observation capabilities in place, the agent can operate in a tight loop:

1. Make a change to the code
2. Restart the affected service (or wait for hot reload)
3. Interact with the application through the browser
4. Observe the results through DOM snapshots, screenshots, logs, and metrics
5. If the results do not match expectations, diagnose the issue and return to step 1
6. If the results match, commit and move on

This loop is remarkably effective. In Anthropic's internal deployments, individual Codex runs using this architecture have worked productively for **six hours straight**, often running while engineers slept. The engineers would return in the morning to find completed features with passing tests, clean commits, and progress logs documenting what was done. The key enabler was not model intelligence — it was the feedback loop that let the model *see* whether its changes worked.

### 7.1.2 Why Most Codebases Are Not Legible

If wiring up CDP and log queries is so effective, why isn't everyone doing it? Because most codebases were not designed with agent legibility in mind. They assume a human developer who can:

- Open a browser and navigate by sight
- Read and interpret visual layouts intuitively
- Correlate log entries across services in their head
- Switch between terminal, browser, and IDE fluidly
- Know which metrics to check and where to find them

None of these assumptions hold for an agent. The agent needs explicit tool access to every observation channel, structured output formats it can parse, and clear success/failure criteria it can evaluate programmatically. Making a codebase agent-legible is an investment — but it is an investment that pays dividends not just for agents but for CI/CD, automated testing, and developer onboarding as well.

---

## 7.2 The Seven Legibility Metrics

Anthropic's research has identified seven dimensions along which a codebase's agent-legibility can be measured. These metrics provide a practical framework for evaluating how well your repository supports autonomous agent work and where to invest in improvements.

### 7.2.1 Bootstrap Self-Sufficiency

**Question:** Can the repo be set up from scratch without external knowledge?

A codebase scores high on bootstrap self-sufficiency if an agent (or a new human engineer) can clone the repository and get a working development environment by following instructions contained entirely within the repo. This means:

- All dependencies are declared in lockfiles (`package-lock.json`, `Cargo.lock`, `go.sum`)
- Environment setup is scripted, not documented in a wiki that requires credentials to access
- External service dependencies (databases, caches, message queues) can be provisioned locally via Docker Compose or similar
- API keys for development are either mocked or available through a documented process that does not require human intervention
- The setup script produces clear error messages when prerequisites are missing

**Anti-patterns that kill bootstrap self-sufficiency:**

- "Ask Sarah for the dev API key"
- Setup instructions that reference a Confluence page behind SSO
- Dependencies on internal package registries that require VPN access
- Environment variables documented in a Slack message from 2023
- A README that says "Follow the setup guide" with a broken link

The ideal is a single command — `make setup` or `./init.sh` — that produces a running application from a clean clone. Every step that requires human judgment or external knowledge is a point where an agent will get stuck.

### 7.2.2 Task Entry Points

**Question:** Can the agent easily find and run build, test, and lint commands?

This metric measures how discoverable the project's key operations are. An agent landing in a new repository needs to answer three questions almost immediately: How do I build this? How do I test this? How do I check code quality?

**High legibility looks like:**

- A `Makefile`, `package.json` scripts section, or `justfile` with clearly named targets: `build`, `test`, `lint`, `dev`, `clean`
- Consistent naming across projects in the organization
- Commands that work without arguments for the common case: `npm test` runs all tests, `npm run lint` lints everything
- Commands that accept arguments for the specific case: `npm test -- --filter auth` runs only auth tests

**Low legibility looks like:**

- Build instructions spread across three different CI config files
- Tests that require a specific environment variable to be set but this is only documented in the CI pipeline definition
- Linting that requires a global tool installation not managed by the project
- Different commands for different parts of the monorepo with no unified entry point

A useful heuristic: if a new engineer's first pull request requires them to ask "how do I run the tests?" in Slack, your task entry points need work.

### 7.2.3 Validation Harness

**Question:** Can the agent check whether its changes work?

This is the most critical legibility metric for autonomous agents. A validation harness is the set of tools and processes that allow the agent to verify the correctness of its changes without human review. It includes:

- **Unit tests** that cover core logic and can be run in isolation
- **Integration tests** that verify component interactions
- **End-to-end tests** that exercise user-facing workflows through the actual UI
- **Type checking** that catches structural errors at compile time
- **Contract tests** that verify API compatibility between services

The validation harness must be *fast enough* that the agent can run it within its fix-restart-revalidate loop without excessive context consumption on waiting. A test suite that takes 45 minutes to run is useless for agent feedback loops. Either invest in making it faster or provide a way to run a targeted subset.

The harness must also be *reliable*. Flaky tests are a minor annoyance for human developers who can re-run and use judgment. For agents, a flaky test is indistinguishable from a real failure. The agent will spend its context budget trying to "fix" a test that fails randomly, potentially introducing real bugs in the process. Eliminating test flakiness is a prerequisite for effective agent-driven development.

### 7.2.4 Linting and Formatting

**Question:** Does the codebase enforce style and catch common errors automatically?

This is, in the words of Anthropic's research team, "maybe the biggest low-hanging fruit" for agent legibility.

Linting and formatting tools serve as a *cheap, fast self-check* that the agent can run after every change. They catch entire categories of errors — unused imports, unreachable code, type mismatches, inconsistent formatting, missing return statements — without the cost of running the full test suite. For an agent operating in a tight feedback loop, the ability to run `eslint --fix` and `prettier --write` after every code change is enormously valuable. It catches mistakes in seconds rather than minutes and frees the agent's reasoning capacity for higher-order concerns.

**The cost-benefit is exceptional:**

- **Setup cost:** A few hours to configure ESLint, Prettier, Ruff, rustfmt, or the equivalent for your language
- **Runtime cost:** Milliseconds to seconds per invocation
- **Error coverage:** Catches 30-50% of the trivial mistakes agents make (missing semicolons, wrong import paths, unused variables, formatting inconsistencies)
- **False positive rate:** Near zero when properly configured

**Configuration recommendations for agent-friendly linting:**

- Enable auto-fix where possible so the agent does not need to manually correct formatting issues
- Use strict rulesets — the agent does not have opinions about code style and will happily conform to whatever rules you set
- Include the lint command in the standard task entry points (`npm run lint`, `make lint`)
- Run linting in CI as a hard gate so the agent learns that lint failures must be fixed

The compound effect is significant. An agent that runs linting after every change produces code that is consistently formatted, free of obvious errors, and structurally sound. This dramatically reduces the surface area that human reviewers need to examine.

### 7.2.5 Codebase Map

**Question:** Is there a high-level guide showing where things are?

Large codebases have a geography that takes human engineers weeks to learn. An agent encounters this geography fresh in every session. A codebase map is a document — typically in the repository root — that provides a high-level overview of the project's structure:

```
# Codebase Map

## Directory Structure
src/
  api/          - REST API handlers, one file per resource
  services/     - Business logic layer, called by API handlers
  models/       - Database models (Prisma schema + generated types)
  middleware/   - Express middleware (auth, logging, error handling)
  utils/        - Shared utilities (validation, formatting, crypto)
  config/       - Environment-specific configuration
tests/
  unit/         - Unit tests, mirror src/ structure
  integration/  - Integration tests, one per user workflow
  fixtures/     - Shared test data and mocks

## Key Files
src/api/router.ts         - All route definitions, modify to add new endpoints
src/middleware/auth.ts     - JWT verification, role-based access control
src/config/database.ts     - Database connection and migration runner
prisma/schema.prisma       - Database schema, source of truth for models

## Conventions
- API handlers return { data, error, status } objects
- Services throw typed errors (NotFoundError, ValidationError, etc.)
- All dates stored as UTC, formatted to local time only in API responses
- Feature flags managed via environment variables prefixed with FF_
```

This document does not need to be exhaustive. It needs to answer the question: "If I need to add/modify X, where do I look?" A good codebase map reduces the agent's exploration time from minutes of file system traversal to seconds of document reading.

### 7.2.6 Doc Structure

**Question:** Is documentation organized so the agent can find what it needs?

This metric is distinct from the codebase map. Where the codebase map provides a *spatial* overview (where things are), doc structure provides a *conceptual* overview (how things work). Good doc structure for agent legibility includes:

- **API documentation** that specifies request/response formats, authentication requirements, and error codes
- **Architecture decision records** (see Section 7.2.7) explaining why the system is built the way it is
- **Runbooks** for common operations (deploying, rolling back, adding a new service)
- **Schema documentation** for databases, message formats, and configuration

The key organizing principle is *locality*: documentation about a component should be near that component. A `README.md` in the `src/api/` directory that explains the API conventions is more useful to an agent than a comprehensive wiki page, because the agent discovers it naturally while exploring the code it needs to modify.

**Anti-patterns:**

- All documentation in a separate docs repo that the agent does not have access to
- Documentation that is accurate for the codebase as it existed six months ago
- Documentation that contradicts the code (the agent will trust the documentation and write incorrect code)
- Documentation written for end users rather than developers

### 7.2.7 Decision Records

**Question:** Are past architectural decisions written in the repo?

Architecture Decision Records (ADRs) are short documents that capture the context, decision, and consequences of significant architectural choices. They are valuable for human engineers ("why did we choose Postgres over MongoDB?"), but they are *essential* for agents because agents have no institutional memory.

Without ADRs, an agent encountering a codebase that uses an unusual pattern — say, event sourcing for state management, or a custom ORM instead of a standard one — will attempt to "fix" the unusual pattern by refactoring toward what it considers more standard practices. An ADR that explains "We use event sourcing because our audit requirements demand a complete history of all state changes" prevents this harmful normalization.

**Effective ADR format:**

```markdown
# ADR-007: Use Event Sourcing for Order State Management

## Status: Accepted (2025-03-15)

## Context
Regulatory requirements mandate a complete, immutable audit trail of all
order state transitions. Our previous CRUD-based approach required complex
audit logging that was error-prone and incomplete.

## Decision
Order state is managed through an append-only event log. Current state is
derived by replaying events. The OrderProjection service materializes
current state for queries.

## Consequences
- All order mutations go through the EventStore, not direct DB updates
- OrderProjection must be rebuilt when event schema changes
- Query performance depends on projection freshness (currently ~100ms lag)
- New developers must understand event sourcing before modifying order logic
```

The critical section for agent legibility is **Consequences**, which tells the agent what constraints it must respect. An agent reading ADR-007 will understand that it must not add a direct database UPDATE to the orders table, even if that would be the "obvious" approach.

---

## 7.3 The Evaluator-Optimizer Pattern in Practice

The evaluator-optimizer pattern is one of the most powerful compound architectures for improving agent output quality. Its structure is simple: one LLM instance generates a candidate output, and a second LLM instance evaluates that output against defined criteria. If the evaluation identifies deficiencies, the generator receives the feedback and produces an improved version. This cycle repeats until the evaluator is satisfied or a maximum iteration count is reached.

### 7.3.1 When This Pattern Fits

The evaluator-optimizer pattern is not universally applicable. It fits well when two conditions are met:

**Condition 1: Clear evaluation criteria exist.** The evaluator needs to know what "good" looks like. For code generation, this might mean: "The code compiles, passes all tests, follows the project's style guide, and handles the specified edge cases." For text generation, it might mean: "The summary captures all key points from the source material, is factually accurate, and is under 200 words." When evaluation criteria are vague or subjective ("make the code more elegant"), the evaluator cannot provide actionable feedback, and the loop degenerates into meaningless iteration.

**Condition 2: Iterative refinement provides measurable value.** Some tasks get better with iteration; others do not. Refining a function's error handling through multiple rounds of review genuinely improves it — each round catches edge cases the previous round missed. But refining a simple getter method through multiple rounds adds cost without improvement. The pattern works best for tasks with *depth* — tasks where there are multiple dimensions of quality that are hard to satisfy simultaneously on the first attempt.

### 7.3.2 Two Signs of Good Fit

Anthropic's research identifies two practical heuristics for determining whether the evaluator-optimizer pattern will be effective for a given use case:

**Sign 1: Human feedback demonstrably improves the response.** Before building an automated evaluator, test the concept manually. Take an LLM-generated output, write feedback on it as a human reviewer, feed that feedback back to the LLM, and compare the revised output to the original. If the revised output is meaningfully better, the pattern has potential. If the original and revised outputs are roughly equivalent, iteration is adding cost without value.

**Sign 2: An LLM can provide feedback of similar quality to a human.** The evaluator does not need to be perfect — it needs to be good enough to identify the most impactful improvements. Test this by comparing LLM-generated evaluations to human evaluations on the same outputs. If the LLM consistently identifies the same top issues that humans identify (even if it misses some subtler ones), it is a viable evaluator. If the LLM's evaluations are orthogonal to human evaluations — praising things humans criticize and vice versa — the pattern will not work.

### 7.3.3 Architecture and Implementation Considerations

**Evaluator prompt design.** The evaluator's prompt is as important as the generator's, perhaps more so. It should include:

- Explicit evaluation criteria, ranked by importance
- Examples of good and bad outputs with explanations
- Instructions to be *specific* in feedback — "The function does not handle the case where the input array is empty" is actionable; "The function needs improvement" is not
- A structured output format that the generator can parse

**Iteration budgets.** Without a maximum iteration count, the loop can cycle indefinitely — the generator changes something, the evaluator finds a new issue, the generator fixes it but introduces a different issue, and so on. Three to five iterations is typically the sweet spot: enough to catch major issues, not so many that you hit diminishing returns or oscillation.

**Different models for different roles.** The generator and evaluator do not need to be the same model, and in some cases they should not be. A larger, more capable model as the evaluator paired with a smaller, faster model as the generator can be cost-effective — the generator does the bulk of the token-intensive work, while the evaluator needs fewer tokens but higher judgment quality.

**Evaluation specificity.** The most common failure mode of evaluator-optimizer loops is *vague evaluation*. When the evaluator says "The code could be improved," the generator makes an arbitrary change that may or may not be an improvement. When the evaluator says "Line 15: the `catch` block swallows the error without logging it; add `logger.error(err)` before the re-throw," the generator makes a targeted fix. Specificity in evaluation feedback is the difference between productive iteration and random walk.

---

## 7.4 Evaluation Systems (Evals)

If feedback loops are how agents validate individual changes, *evals* are how you validate the entire system. An eval is a systematic process for measuring how well your agent performs across a representative set of tasks. Evals are to agent systems what test suites are to traditional software: the mechanism by which you know whether a change improved things, broke things, or had no effect.

Building a good eval system is one of the highest-leverage investments you can make in agent engineering. It is also one of the most commonly underinvested areas, because the payoff is indirect — evals do not ship features; they tell you whether your features work. This section provides a comprehensive guide to building evals that actually work.

### 7.4.1 Core Concepts and Terminology

Before diving into eval design, we need a precise vocabulary. These terms are used throughout the eval literature and in this chapter:

**Task.** A specific, well-defined problem that the agent must solve. A task includes a description (what the agent is asked to do), any input data, and criteria for what constitutes a correct solution. Example: "Given the repository at commit `abc123`, fix the bug described in issue #456 so that the test in `tests/issue_456.test.ts` passes."

**Trial.** A single attempt by the agent to complete a task. Because LLM behavior is non-deterministic, the same task may produce different results across trials. Eval systems typically run multiple trials per task to account for this variance.

**Grader.** The mechanism that evaluates a trial's output and assigns a score. Graders are the most critical component of an eval system — an eval is only as good as its grader. We will examine grader types in detail in Section 7.4.2.

**Transcript.** The complete record of a trial: the agent's reasoning, tool calls, tool outputs, and final result. Transcripts are essential for debugging failed trials and for understanding *why* the agent succeeded or failed, not just *whether* it did.

**Outcome.** The grader's assessment of a trial. At minimum, an outcome is a binary pass/fail. More sophisticated systems include partial credit scores, specific failure reasons, and metadata about which aspects of the task were handled correctly.

### 7.4.2 The Three Grader Types

Every grader falls into one of three categories, each with distinct tradeoffs:

#### Code-Based Graders

Code-based graders evaluate trial outcomes using deterministic logic: running tests, checking file contents, comparing outputs to expected values, verifying that specific strings appear in specific files.

**Strengths:**

- **Fast:** Execution is nearly instantaneous
- **Cheap:** No API calls or model inference required
- **Deterministic:** The same output always gets the same grade
- **Debuggable:** When a grader gives a wrong score, you can trace through the logic and fix it

**Weaknesses:**

- **Brittle:** Code-based graders break when the solution format changes. A grader that checks for `"status": "success"` in the response will fail if the agent returns `"status": "ok"` — even if the solution is correct.
- **Narrow:** They can only check what they are programmed to check. Aspects of quality not anticipated when the grader was written go unexamined.
- **Expensive to maintain:** As the agent's capabilities evolve, graders must be updated to account for new solution strategies that the grader's author did not anticipate.

**Best used for:** Tasks with clearly defined, machine-verifiable outcomes. "Does the test pass?" "Does the function return the correct output for these inputs?" "Does the generated file contain the required configuration?"

#### Model-Based Graders

Model-based graders use an LLM to evaluate the agent's output. The grader LLM receives the task description, the agent's output, and evaluation criteria, and produces a score with a justification.

**Strengths:**

- **Flexible:** Can evaluate subjective qualities like code readability, documentation clarity, or UI design appropriateness
- **Adaptive:** Naturally handles solution variations — if the agent solves the problem in an unexpected but correct way, a model grader can recognize this
- **Low maintenance:** The same grader prompt works across many tasks without modification

**Weaknesses:**

- **Non-deterministic:** The same output may receive different scores across evaluations. This makes it harder to track improvements precisely and can create noise in eval results.
- **Expensive:** Each grading operation requires an LLM inference call, which at scale adds meaningful cost
- **Susceptible to biases:** Model-based graders share the biases of the models they use — they may favor verbose solutions over concise ones, or conventional approaches over novel ones
- **Circular reasoning risk:** If the same model (or a closely related one) generates and evaluates, it may rate its own idiosyncratic outputs higher than equally valid alternatives

**Best used for:** Tasks where correctness is not purely mechanical — code review quality, documentation completeness, design appropriateness, explanation clarity.

#### Human Graders

Human graders are domain experts who evaluate the agent's output manually.

**Strengths:**

- **Gold standard accuracy:** A skilled human evaluator catches issues that neither code nor model graders would identify — subtle bugs, misleading variable names, architecturally questionable decisions, edge cases that the task description did not explicitly mention
- **Contextual understanding:** Humans bring organizational context, domain expertise, and implicit quality standards that are difficult to encode in grader prompts
- **Calibration:** Human grades serve as the reference against which code-based and model-based graders can be calibrated

**Weaknesses:**

- **Expensive:** Human evaluation does not scale. At $50/hour for a senior engineer, grading 1,000 trials at 5 minutes each costs over $4,000.
- **Slow:** A human grader processes orders of magnitude fewer trials per hour than automated graders
- **Inconsistent:** Different humans grade differently, and the same human grades differently on Monday morning versus Friday afternoon. Inter-rater reliability must be measured and managed.
- **Not automatable:** Human grading cannot be embedded in CI/CD pipelines or triggered automatically on code changes

**Best used for:** Calibrating automated graders, evaluating novel task types where automated criteria have not yet been established, and final validation of high-stakes eval results.

### 7.4.3 Capability Evals vs. Regression Evals

Eval suites serve two fundamentally different purposes, and conflating them leads to confusion and wasted effort.

**Capability evals** measure what the agent *can* do. They push the boundary of difficulty to find the agent's limits. A capability eval for a coding agent might include tasks ranging from trivial (rename a variable) to extremely hard (implement a distributed consensus algorithm from a paper). The purpose is to map the agent's competence frontier: at what difficulty level does performance degrade? What categories of tasks does it handle well versus poorly?

Capability evals are *expected* to have low pass rates on hard tasks. A capability eval where the agent passes everything is not challenging enough to be informative — it is like a strength test where the heaviest weight is 10 pounds.

**Regression evals** measure that the agent *still* does what it used to do. When you change the agent's prompt, update its tools, switch to a new model version, or modify the harness in any way, regression evals verify that previously working capabilities have not been broken. The tasks in a regression suite are things the agent has demonstrated it can do — they are the "test suite" for the agent itself.

Regression evals should have *high* pass rates. A regression eval failure is a bug — something that worked before is now broken. It demands investigation and resolution, just like a failing test in a software project.

**The relationship between the two:** Tasks migrate from capability evals to regression evals. When a capability eval reveals that the agent can reliably handle a new category of task (say, pass rate exceeds 90%), representative tasks from that category move into the regression suite, where they serve as guardrails against future regressions.

### 7.4.4 pass@k vs. pass^k

Two aggregate metrics dominate eval reporting, and understanding their semantics is essential for interpreting results correctly:

**pass@k** (pass-at-k): Run the task k times. The task is considered solved if *at least one* of the k trials succeeds. Formally:

> pass@k = 1 - (number of ways to choose k failures from n trials) / (number of ways to choose k trials from n)

In practice, for small k, this is often approximated as: run k trials, report success if any trial passes.

pass@k measures *capability* — can the agent solve this problem at all, given multiple attempts? A pass@5 of 80% means that if you run the task five times, there is an 80% chance that at least one run will produce a correct solution. This metric is useful for scenarios where you can run multiple attempts and select the best one (e.g., generating code completions where a human will choose among candidates).

**pass^k** (pass-to-the-k, or "consistent pass"): Run the task k times. The task is considered solved only if *all k trials succeed*.

pass^k measures *reliability* — does the agent solve this problem consistently? A pass^5 of 60% means that if you run the task five times, there is a 60% chance that every single run will produce a correct solution. This metric is the right one for autonomous deployments where the agent runs unsupervised and you need confidence that it will succeed on any given attempt.

**The gap between pass@k and pass^k reveals non-determinism.** If pass@5 is 95% but pass^5 is 40%, the agent *can* solve the task but does so unreliably. This gap tells you that the task is within the agent's capabilities but that something — prompt sensitivity, sampling randomness, or fragile reasoning chains — makes the outcome inconsistent. Closing this gap is often a matter of prompt engineering, better tool design, or more constrained solution paths, rather than fundamental model improvement.

### 7.4.5 The Roadmap to Great Evals

Building an eval system is an iterative process. The following roadmap provides a sequence of steps, ordered by increasing sophistication, for developing evals that genuinely measure and improve your agent's performance.

#### Step 1: Start with 20-50 Simple Tasks from Real Failures

Do not begin by designing a comprehensive eval framework. Begin by collecting real failures. Every time your agent produces incorrect output in production — a bug, a misunderstood requirement, a broken build — capture the task as an eval case. These real-world failures are more valuable than synthetic tasks because they represent the actual distribution of problems your agent encounters.

For each failure, record:

- The exact prompt or task description the agent received
- The context it had access to (files, documentation, tool outputs)
- What it produced (the incorrect output)
- What it should have produced (the correct output, or at least the criteria for correctness)
- Why it failed (if you can determine this from the transcript)

Twenty to fifty such cases give you a meaningful starting eval suite. You do not need hundreds of tasks to begin — you need representative tasks from your actual failure distribution.

#### Step 2: Convert Manual Checks to Automated Tasks

If you are currently validating agent output by manually reviewing pull requests, convert your review process into automated checks. For every category of issue you catch in review, ask: can this be checked programmatically?

- "The agent forgot to update the test" → Check that modified source files have corresponding test modifications
- "The agent used a deprecated API" → Lint rule or grep for deprecated patterns
- "The agent's solution works but is O(n^2) when O(n) is possible" → Benchmark test with performance threshold
- "The agent didn't handle the null case" → Add null-input test cases to the eval

Not every manual check can be automated, but a surprising number can. Each automated check reduces your review burden and makes the eval suite more comprehensive.

#### Step 3: Write Unambiguous Tasks with Reference Solutions

Ambiguous tasks produce noisy eval results. If the task says "improve the search function," any change to the search function could be argued as an improvement. Instead:

**Ambiguous:** "Fix the performance issue in the user dashboard."

**Unambiguous:** "The `/api/dashboard` endpoint currently takes >2 seconds to respond when the user has more than 1,000 records. Modify the implementation so that it responds in under 200ms for users with up to 10,000 records, as verified by the existing benchmark test in `tests/perf/dashboard.bench.ts`."

Reference solutions serve two purposes: they verify that the task is solvable (if you cannot write a correct solution, the task is ill-defined), and they provide calibration material for model-based graders (the grader can compare the agent's solution to the reference).

#### Step 4: Build Balanced Problem Sets

An eval suite dominated by one category of task will give you a distorted picture of agent performance. If 40 of your 50 tasks are "fix a bug in a Python function," you know a lot about the agent's Python debugging ability and very little about anything else.

Balance across multiple dimensions:

- **Language/framework:** Distribute tasks across the languages and frameworks your agent works with
- **Task type:** Bug fixes, feature implementation, refactoring, documentation, configuration changes
- **Difficulty:** Easy tasks that should always pass (regression), medium tasks that test core competence, hard tasks that probe the frontier (capability)
- **Codebase size:** Small isolated files, medium modules, large multi-file changes
- **Domain:** Frontend, backend, data, infrastructure, testing

A balanced suite of 50 tasks is more informative than an imbalanced suite of 500.

#### Step 5: Build a Robust Eval Harness with Clean Environments

Each eval trial must start from a known, clean state. If trial 1 modifies a file and trial 2 starts with that modification still in place, your results are contaminated. The eval harness must:

- **Provision a fresh environment per trial.** This typically means a fresh git checkout, a clean Docker container, or a fresh virtual machine. The overhead is worth it — shared state between trials is the most common source of unreliable eval results.
- **Enforce time and resource limits.** An agent that enters an infinite loop should not block the entire eval run. Set timeouts per trial (e.g., 10 minutes for simple tasks, 30 minutes for complex ones) and kill trials that exceed them.
- **Capture complete transcripts.** Every tool call, every response, every intermediate result. You will need these for debugging.
- **Handle non-determinism gracefully.** Run multiple trials per task and report statistical results rather than single-run outcomes.

#### Step 6: Grade Outcomes, Not Paths

A critical principle of eval design: grade the *result*, not the *process*. If the task is to fix a bug, the grader should check whether the bug is fixed — not whether the agent used the same approach as the reference solution.

**Wrong approach:** Check that the agent's diff matches the expected diff.

**Right approach:** Run the test suite and check that the previously failing test now passes while no previously passing tests have broken.

This principle extends to partial credit. A binary pass/fail grader loses information. A grader that assigns partial credit — "the agent fixed the primary bug (0.7) but introduced a minor regression in error message formatting (−0.1), net score 0.6" — provides much richer signal for understanding agent behavior and measuring improvement over time.

Design partial credit schemes that reflect the relative importance of different correctness dimensions:

- Core functionality works: 0.5
- Edge cases handled: 0.2
- No regressions introduced: 0.15
- Code quality (style, documentation): 0.1
- Performance within bounds: 0.05

#### Step 7: Check Transcripts — Verify the Eval Measures What Matters

After running your first batch of evals, manually review the transcripts for a sample of both passing and failing trials. You are looking for:

- **False positives:** The grader marked the trial as passing, but the agent's solution is actually incorrect. This means your grader is too lenient.
- **False negatives:** The grader marked the trial as failing, but the agent's solution is actually correct (just in a way the grader did not anticipate). This means your grader is too brittle.
- **Right answer, wrong reason:** The agent passed the eval but got lucky — it made compensating errors, or it hardcoded the expected output, or it solved a different problem that happened to produce the right answer. These cases indicate that your task or grader needs to be more rigorous.
- **Wrong answer, right process:** The agent followed a reasonable approach but made a small mistake that cascaded into a failure. These cases suggest the task difficulty is near the agent's capability boundary — worth keeping for capability evals but perhaps too hard for regression suites.

Transcript review is labor-intensive but irreplaceable. It is the only way to verify that your eval system is actually measuring what you think it is measuring.

#### Step 8: Monitor for Capability Eval Saturation

As your agent improves (through better prompts, better tools, or better models), capability evals that were once challenging become easy. When a capability eval's pass rate exceeds 90-95% consistently across multiple runs, it has saturated — it is no longer providing useful signal about the agent's frontier.

When a capability eval saturates:

1. Move representative tasks from it into the regression suite
2. Design harder variants of the saturated tasks for the new capability eval
3. Explore new task categories that the agent has not been tested on

Saturation is a *good sign* — it means your agent is genuinely improving. But an eval suite composed entirely of saturated tasks is useless for guiding further improvement.

#### Step 9: Keep Suites Healthy Long-Term

Eval suites, like test suites, require ongoing maintenance:

- **Remove tasks that no longer reflect real usage.** If your agent no longer works with Python 2, remove Python 2 tasks.
- **Update reference solutions when APIs change.** A reference solution that uses a deprecated library function will produce false negatives.
- **Recalibrate model-based graders periodically.** As grader models are updated, their scoring behavior may shift. Re-validate against human grades quarterly.
- **Track eval suite metrics over time.** Not just agent performance on evals, but eval suite health: how many tasks are saturated? How many have unstable graders? What is the false positive/negative rate?
- **Add new tasks as new failure modes are discovered.** The eval suite should be a living document of everything you have learned about how your agent fails.

---

## 7.5 Putting It All Together: The Verification Stack

The concepts in this chapter form a layered verification stack, where each layer catches errors that the layer below it misses:

```
Layer 5: Human review (spot-check, high-stakes decisions)
   |
Layer 4: Eval suite (systematic measurement across many tasks)
   |
Layer 3: Evaluator-optimizer loop (iterative refinement per task)
   |
Layer 2: End-to-end verification (browser automation, log queries)
   |
Layer 1: Fast automated checks (linting, type checking, formatting)
   |
Layer 0: The code itself
```

**Layer 1** catches syntactic and structural errors in milliseconds. It is the cheapest and fastest check. Run it after every change.

**Layer 2** catches functional errors — the code compiles and passes lint, but does it actually work when a user interacts with it? This requires a running application and observation tools. Run it after implementing each feature.

**Layer 3** catches quality and completeness issues — the feature works, but does it handle edge cases? Is the error messaging helpful? Is the code maintainable? Run it for complex or high-impact changes.

**Layer 4** catches systemic issues — the agent works well on this task, but has it regressed on tasks it used to handle? Is it improving over time or degrading? Run it on every prompt, tool, or model change.

**Layer 5** catches everything else — the subtle, contextual, judgment-dependent issues that automated systems miss. Use it sparingly and strategically, focusing human attention where it has the highest marginal value.

The goal is not to eliminate human review entirely — it is to make human review *efficient* by ensuring that the agent's output has already passed through four layers of automated verification before a human ever sees it. When the human reviewer opens a pull request that has passed all four layers, they can focus on architecture, strategy, and subtle correctness issues rather than catching typos, lint violations, and broken tests.

---

## 7.6 Summary

Replacing human QA with automated feedback loops is not a single technique — it is a *system* of interlocking mechanisms:

- **Agent legibility** gives the agent the perceptual capabilities to observe the running application, turning it from a blind code generator into a developer with eyes on the product.
- **The seven legibility metrics** provide a practical framework for evaluating and improving your codebase's readiness for autonomous agent work.
- **The evaluator-optimizer pattern** creates internal quality pressure through adversarial evaluation loops, catching issues that single-pass generation misses.
- **Eval systems** provide the meta-level verification that the entire apparatus works — measuring agent performance systematically, catching regressions, and guiding improvement.

The common thread is *closing loops*. Every gap between "agent writes code" and "agent verifies code works" is a gap where bugs hide. Every observation channel you wire into the agent's runtime — DOM snapshots, screenshots, log queries, metrics, linting output, test results — closes one of these gaps. The tighter the loops, the more the agent can be trusted to work autonomously, and the more human attention can be redirected from routine verification to the high-judgment work that humans still do best.

In the next chapter, we will examine how these verification systems interact with deployment pipelines, exploring the patterns for safely shipping agent-generated code to production.
