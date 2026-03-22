# Chapter 6: Long-Running Agents — Working Across Context Windows

Every agent architecture we have discussed so far assumes a comforting fiction: that the task at hand can be completed within a single context window. For a surprising number of real-world problems — answering questions, writing a function, debugging a focused issue — this fiction holds. But the moment you turn an agent loose on a genuinely complex task — building a full-stack web application from a specification, migrating a legacy codebase to a new framework, implementing a feature that touches dozens of files across multiple services — you collide with one of the hardest unsolved problems in applied AI engineering: *continuity across context boundaries*.

This chapter examines the problem in depth, walks through the best currently-known solutions drawn from Anthropic's research, and catalogs the failure modes you will encounter when your agent's work outlives its memory.

---

## 6.1 The Problem: Amnesia at Shift Change

Imagine a software project staffed entirely by contract engineers working in eight-hour shifts. Each new engineer arrives at the office with no memory of what happened on the previous shift. They sit down at a desk, open the laptop, and see a codebase in some intermediate state. Files have been added. Tests exist but some are failing. There is a half-written migration script. A branch has been created but not merged. The CI pipeline is red.

This is the situation every long-running agent faces when a new context window begins.

The analogy is more precise than it might first appear. A human engineer arriving at such a desk would do several things instinctively: read the git log, check the CI dashboard, look for a README or a handoff document, maybe ask a colleague what happened. They would orient themselves before writing a single line of code. The key insight of long-running agent design is that *your harness must make this orientation possible* — and, ideally, make it nearly automatic.

### 6.1.1 Why Compaction Is Not Enough

A natural first reaction is to rely on context compaction (Chapter 4). If the agent is running out of context, summarize what has happened so far and inject that summary into the next window. This approach helps, but even with frontier models performing the summarization, it falls short for several reasons:

**Lossy compression of implementation state.** A summary might note "implemented user authentication with JWT tokens," but it will not capture the fact that the refresh token rotation logic is half-implemented, that the middleware is registered in `app.ts` but the route guard has not been wired up in the router, or that the test for token expiry is written but depends on a mock clock utility that does not yet exist. These details are precisely the ones that matter when the next session picks up the work.

**Accumulated drift.** Each compaction pass introduces small inaccuracies. Over five or ten sessions, these compound. The agent's understanding of the codebase diverges from reality in subtle ways — it believes a file exists that was deleted two sessions ago, or it thinks a function signature takes two arguments when it was refactored to take three.

**No ground truth anchor.** A compacted summary is the agent's *belief* about the state of the world. It is not the state of the world itself. The file system, the test suite, the git history — these are ground truth. Any viable long-running agent strategy must re-anchor to ground truth at the start of every session, not rely on the agent's potentially corrupted memory.

### 6.1.2 The Two Canonical Failure Patterns

When teams first attempt long-running agent tasks without explicit harness support, they encounter two failure patterns with near-perfect reliability:

**Failure Pattern 1: The One-Shot Overreach**

The agent attempts to implement everything in a single session. It begins writing code furiously — scaffolding files, creating components, wiring up routes, adding database models — until it exhausts its context window mid-implementation. When the next session starts, the new agent instance finds a codebase littered with half-implemented, undocumented features. Files reference modules that do not exist. Functions are called but never defined. Tests import fixtures that were never created. The codebase is in a state that would challenge even a senior human engineer to untangle.

This failure mode is pernicious because the agent in its first session *feels* productive. It is generating code at high velocity. But velocity without commitment boundaries is just chaos with a high line count.

**Failure Pattern 2: The Premature Completion**

The agent begins a new session, reads the codebase, sees that significant progress has been made — files exist, routes are defined, the application starts without crashing — and concludes that the task is essentially done. It writes a summary declaring victory and exits. In reality, half the features are non-functional, edge cases are unhandled, and the test suite is a mix of passing stubs and genuinely broken assertions.

This failure mode exploits a well-known tendency in language models: they are biased toward declaring success. When an agent sees a codebase that *looks* substantial, it conflates the *presence* of code with the *correctness* of code. Without a rigorous verification mechanism, this bias goes unchecked.

---

## 6.2 The Two-Part Solution: Initializer and Coding Agents

The most effective architecture Anthropic's research has identified for long-running tasks splits the work into two distinct agent roles, each with its own specialized prompt and behavioral contract.

### 6.2.1 The Initializer Agent

The initializer agent runs exactly once, at the very beginning of the project. Its job is not to write application code. Its job is to *create the infrastructure that makes all subsequent sessions productive*. Think of it as the tech lead who sets up the project on day one: scaffolding the repository, writing the contributing guide, defining the test framework, and establishing conventions that every future contributor will follow.

The initializer agent is responsible for the following deliverables:

#### The Bootstrap Script: `init.sh`

This is a shell script that can bring the development environment from zero to a running state. It installs dependencies, starts development servers, seeds databases, and performs any other setup required. The critical property of `init.sh` is *idempotency* — it must be safe to run at the beginning of every session, not just the first one. A coding agent waking up in session 14 should be able to run `./init.sh` and have a working environment within seconds, without understanding the full history of what has been set up before.

```bash
#!/bin/bash
# init.sh — idempotent project bootstrap
set -euo pipefail

# Install dependencies (npm ci is idempotent by design)
npm ci

# Start dev server if not already running
if ! lsof -i :3000 > /dev/null 2>&1; then
    npm run dev &
    sleep 3
fi

# Run database migrations (idempotent if using migration framework)
npx prisma migrate deploy

echo "Environment ready."
```

The initializer agent's prompt should instruct it to write this script early and test it before moving on.

#### The Progress File: `claude-progress.txt`

This is a plain-text file that serves as the handoff document between sessions. Every coding agent session appends to this file before exiting, recording:

- What was accomplished in this session
- What was attempted but not completed
- What blocked progress
- What should be done next

The progress file is the closest analog to the shift-change briefing in our engineering team metaphor. It is deliberately kept as a plain text file rather than a structured format because it serves a narrative purpose: it tells the *story* of the project's evolution, which helps the next agent instance build a mental model quickly.

#### The Initial Git Commit

The initializer agent creates the first commit in the repository with all scaffolding in place. This serves two purposes: it establishes a known-good baseline that any future session can reset to if things go badly wrong, and it begins the git history that subsequent agents will rely on for orientation.

#### The Comprehensive Feature List

This is the single most important artifact the initializer agent creates. It is an exhaustive list of every feature the application must support, expressed as testable requirements, and initially marked as failing.

For a web application, this list might contain 200 or more entries. Here is a representative sample:

```json
{
  "features": [
    {
      "id": "AUTH-001",
      "category": "Authentication",
      "description": "User can register with email and password",
      "status": "failing",
      "test_file": "tests/auth/register.test.ts",
      "priority": 1
    },
    {
      "id": "AUTH-002",
      "category": "Authentication",
      "description": "User cannot register with an already-taken email",
      "status": "failing",
      "test_file": "tests/auth/register-duplicate.test.ts",
      "priority": 1
    },
    {
      "id": "AUTH-003",
      "category": "Authentication",
      "description": "User can log in with valid credentials",
      "status": "failing",
      "test_file": "tests/auth/login.test.ts",
      "priority": 1
    },
    {
      "id": "NAV-001",
      "category": "Navigation",
      "description": "Sidebar shows all top-level sections",
      "status": "failing",
      "test_file": "tests/navigation/sidebar.test.ts",
      "priority": 2
    }
  ]
}
```

Several design choices in this format are deliberate and hard-won:

**JSON, not Markdown.** The feature list is stored as JSON rather than Markdown because language models are significantly less likely to inappropriately modify JSON structures. When the feature list is in Markdown, agents frequently "clean up" the formatting, reword descriptions, consolidate entries, or silently remove features they consider redundant. JSON's rigid syntax creates a natural barrier against casual editing. An agent that wants to change a feature's status must parse the JSON, locate the correct entry, and modify the `status` field — a surgical operation rather than a sweeping rewrite.

**Every feature starts as "failing."** This is a critical design choice. By starting with everything marked as failing, you create an unambiguous definition of "done": the project is complete when every feature's status is "passing." This eliminates the premature completion failure mode because the agent can trivially verify how much work remains by counting failing features.

**Each feature maps to a test file.** The test file path is specified upfront, even though the test does not yet exist. This forces the initializer agent to think through the testing strategy and creates a contract that the coding agent must honor.

**Features have priorities.** Priority ordering prevents the agent from cherry-picking easy features while ignoring hard ones. The coding agent's instructions tell it to work on the highest-priority incomplete feature, ensuring that foundational features (like authentication) are implemented before features that depend on them (like user profiles).

#### The Testing Contract

The initializer agent's prompt includes an explicit, forceful instruction:

> "It is unacceptable to remove or edit tests. Tests define the specification. If a test is failing, the implementation must be fixed to make it pass. Under no circumstances should a test be weakened, deleted, or modified to match incorrect behavior."

This instruction exists because, without it, agents will routinely "fix" failing tests by changing the test's assertions to match whatever the implementation happens to do. This is the software equivalent of moving the goalposts — the test passes, but the feature is broken. The instruction must be unambiguous and emphatic because models will, under context pressure, rationalize test modifications as "updating the test to reflect the current design."

### 6.2.2 The Coding Agent

Every session after the first uses the coding agent prompt. This prompt defines a behavioral contract that is, in some ways, the opposite of how we typically think about maximizing agent productivity. Where instinct says "do as much as possible per session," the coding agent prompt says "do one thing well and leave the codebase in a clean state."

The coding agent's core behavioral rules:

**Work on ONE feature at a time.** Not two. Not "this one plus the easy one next to it." One. This constraint exists because partially implemented features are the primary source of cross-session confusion. A fully implemented feature with passing tests is an asset. A partially implemented feature is a liability — it creates broken imports, failing tests with unclear causes, and code paths that look intentional but are actually scaffolding.

**Commit to git with descriptive messages.** Every meaningful unit of progress gets its own commit. The commit message is not just documentation — it is the primary mechanism by which future agent sessions understand what has been done. A message like "implement user registration endpoint with validation and duplicate-email check" is infinitely more useful than "progress on auth."

**Write progress summaries.** Before exiting, the agent appends to `claude-progress.txt` with a structured summary of the session. This is the handoff briefing.

**Leave the environment in a clean, mergeable state.** No uncommitted changes. No running processes that will fail on the next startup. No half-written files. The codebase at the end of every session should be in a state where a human engineer could check it out, run the tests, and understand what exists.

---

## 6.3 The Getting-Up-to-Speed Protocol

When a coding agent session begins, it follows a precise orientation protocol before writing any code. This protocol is the equivalent of the new-shift engineer's first thirty minutes: read the room, understand the state of play, verify that the environment works, and only then start building.

### Step 1: Run `pwd`

This sounds trivial, but it is the agent's first contact with ground truth. The working directory tells the agent where it is in the file system, which project it is working on, and whether the environment looks as expected. In automated pipelines where agents might be spun up in incorrect directories, this single check prevents an entire class of errors.

### Step 2: Read Git Logs and Progress Files

```bash
git log --oneline -20
cat claude-progress.txt
```

The git log provides a factual record of what has been committed. The progress file provides narrative context. Together, they give the agent a much richer understanding than either alone: the git log says *what* was done; the progress file says *why*, *what was attempted but abandoned*, and *what should come next*.

### Step 3: Read the Feature List and Choose the Next Feature

The agent reads the feature list JSON, filters for features with status "failing," sorts by priority, and selects the highest-priority incomplete feature. This selection is deterministic and auditable — there is no ambiguity about what the agent should work on next.

### Step 4: Run `init.sh` to Start the Dev Server

The bootstrap script brings the environment to a known-good state. Because `init.sh` is idempotent, this step is safe regardless of how the previous session left things.

### Step 5: Run Basic End-to-End Tests

Before implementing anything new, the agent runs the existing test suite to verify that previously implemented features still work. This step catches regressions introduced by the previous session (which might have committed code that passed unit tests but broke integration tests) and establishes a baseline for the current session.

### Step 6: Fix Existing Bugs Before Starting New Work

If the end-to-end test run reveals failures in features that are marked as "passing" in the feature list, the agent fixes those regressions before starting on new features. This is counterintuitive — the agent is "wasting" its session on work that was supposedly already done — but it is essential for maintaining system integrity. A codebase with accumulating regressions becomes exponentially harder to work with over time.

---

## 6.4 Testing and Verification: The Trust Problem

The most insidious failure mode in long-running agent systems is not incorrect code — it is the agent's self-assessment of correctness. Left to its own devices, Claude will tend to mark features as "passing" after implementing them but before rigorously verifying that they work end-to-end. The agent's internal reasoning often goes something like: "I wrote the handler, I wrote the test, the test passes, therefore the feature works." But this reasoning ignores an entire class of integration issues: the handler works in isolation but the route is not registered, the test mocks the database but the real database has a different schema, the frontend component renders but the CSS makes it invisible.

### 6.4.1 Browser Automation for Ground Truth

For web applications, the gold standard for verification is to test the feature the way a human user would: by interacting with it through a browser. The most effective tool for this in agent harnesses is browser automation via a tool like Puppeteer, often exposed to the agent through the Model Context Protocol (MCP).

With Puppeteer MCP, the agent can:

- **Navigate to URLs** in a real browser instance
- **Fill in form fields** and click buttons
- **Read the rendered DOM** to verify that expected content appears
- **Take screenshots** to visually verify layout and styling
- **Execute JavaScript** in the page context to check application state
- **Wait for network requests** to complete before making assertions

This transforms testing from "does my code look correct?" to "does the feature actually work when a user interacts with it?" The difference is profound. An agent that has verified a registration flow by actually filling in the form, clicking the button, seeing the success message, and then logging in with the newly created credentials has *evidence* that the feature works, not just *belief*.

### 6.4.2 The Limitations of Browser Automation

Browser automation through Puppeteer (or similar tools) is not a perfect oracle. There are categories of behavior it cannot verify:

**Browser-native modals.** JavaScript `alert()`, `confirm()`, and `prompt()` dialogs are rendered by the browser chrome, not the DOM. Puppeteer can intercept these dialogs programmatically, but the agent cannot "see" them the way it sees DOM elements. If your application relies on native browser dialogs for important user interactions, the agent will have a blind spot. The practical solution is to replace native dialogs with custom modal components that are part of the DOM — a design choice that happens to also be better for accessibility and testing in general.

**Visual regression.** The agent can take screenshots, but interpreting them requires vision capabilities that may not be available or reliable for pixel-level comparisons. Layout bugs — an element shifted by 2 pixels, a color slightly wrong, text truncated by overflow — are hard for agents to detect from screenshots alone.

**Timing-dependent behavior.** Animations, transitions, debounced inputs, and other time-dependent features are inherently fragile under automation. The agent may need to introduce explicit waits, which introduces its own class of flaky behavior.

**Cross-browser behavior.** Puppeteer typically runs Chromium. Features that behave differently in Safari or Firefox will not be caught.

Despite these limitations, browser automation represents a massive improvement over the alternative, which is the agent simply trusting its own code. In Anthropic's experiments, adding Puppeteer-based verification to the coding agent's workflow reduced false "passing" declarations by a significant margin.

---

## 6.5 Failure Mode Reference Table

The following table catalogs the most common failure modes in long-running agent systems and the specific behaviors in both the initializer and coding agent that prevent them.

| Failure Mode | Root Cause | Initializer Agent Prevention | Coding Agent Prevention |
|---|---|---|---|
| **One-shot overreach** | Agent tries to implement everything in one session | Creates prioritized feature list requiring sequential implementation | "Work on ONE feature at a time" instruction; commit after each feature |
| **Premature completion** | Agent sees existing code and declares done | All features start as "failing"; completion requires all features "passing" | Must verify features via tests before marking "passing" |
| **Silent regression** | New feature breaks a previously working feature | Creates comprehensive test suite from the start | Runs full test suite before starting new work; fixes regressions first |
| **Test weakening** | Agent modifies tests to match broken behavior | "It is unacceptable to remove or edit tests" instruction | Same instruction reinforced in coding agent prompt |
| **Lost context** | New session does not know what happened before | Creates `claude-progress.txt` and establishes commit conventions | Follows getting-up-to-speed protocol; reads logs and progress files |
| **Environment rot** | Dev environment stops working across sessions | Creates idempotent `init.sh` script | Runs `init.sh` at the start of every session |
| **Feature list drift** | Agent modifies the definition of done | Feature list in JSON (resistant to casual editing) | Must not modify feature descriptions, only status |
| **Cherry-picking easy work** | Agent avoids hard features in favor of quick wins | Features have explicit priority ordering | Must work on highest-priority incomplete feature |
| **Uncommitted progress** | Session ends with meaningful work not in git | Establishes commit conventions from the start | "Leave environment in a clean, mergeable state" instruction |
| **Accumulated half-states** | Multiple partially done features across sessions | One feature per session constraint defined in architecture | Commit only when feature is complete and tested |

---

## 6.6 Open Questions and Active Research

The two-part agent architecture described in this chapter represents the current best practice, but it is not a solved problem. Several open questions remain at the frontier of research.

### 6.6.1 Single Agent vs. Multi-Agent Architectures

The architecture presented here uses a single coding agent that handles all tasks: reading the spec, writing code, writing tests, running tests, debugging failures, and committing results. An obvious question is whether this work should be decomposed across specialized agents:

- A **planning agent** that reads the feature list and breaks the next feature into implementation steps
- A **coding agent** that writes the implementation
- A **testing agent** that writes and runs tests independently of the coding agent
- A **QA agent** that performs browser-based verification
- A **cleanup agent** that ensures code quality, removes dead code, and maintains consistency

The theoretical advantage of this decomposition is specialization: each agent can have a focused prompt optimized for its specific task, and the testing agent's incentives are naturally opposed to the coding agent's (the tester wants to find bugs, not ship features). This adversarial dynamic can improve quality.

The practical disadvantages are significant:

**Coordination overhead.** Multi-agent systems require a coordination layer to manage handoffs, resolve conflicts, and handle the case where one agent's output is not in the format another agent expects. This coordination layer is itself a source of bugs and complexity.

**Context fragmentation.** The coding agent understands the implementation deeply because it wrote it. The testing agent must reconstruct this understanding from the code alone. This information loss at each handoff boundary can lead to shallow tests that exercise the happy path but miss the edge cases that the coding agent was actually worried about.

**Debugging difficulty.** When something goes wrong in a multi-agent system, determining which agent made the error and why is significantly harder than debugging a single agent's behavior. The transcript becomes a interleaved log of multiple agents' reasoning, and causal chains cross agent boundaries.

**Cost multiplication.** Each agent session consumes tokens. A four-agent pipeline that processes each feature costs roughly four times as much as a single agent, even if the total work is the same. For long-running tasks with hundreds of features, this cost multiplication is substantial.

The current evidence suggests that a single well-prompted agent with good tooling outperforms naive multi-agent architectures for most tasks. However, this may change as models improve at structured coordination and as frameworks for multi-agent orchestration mature.

### 6.6.2 Generalizing Beyond Web Applications

The initializer/coding agent pattern was developed primarily in the context of building web applications — a domain with several convenient properties:

- **Observable output.** Web applications produce visual output that can be verified through browser automation.
- **Mature testing ecosystems.** JavaScript/TypeScript testing frameworks, HTTP assertion libraries, and browser automation tools are well-established.
- **Incremental buildability.** Web features can often be implemented independently and integrated incrementally.
- **Fast feedback loops.** Dev servers with hot reloading provide near-instant feedback on changes.

Applying the same pattern to other domains requires adapting these properties:

**Systems programming (C, Rust, Go).** The equivalent of browser automation might be integration tests against a running binary, or property-based tests that exercise APIs. The bootstrap script becomes more complex (managing compilation, linking, and system dependencies). Feature lists may need to account for non-functional requirements like memory safety and performance targets.

**Data engineering.** Features are pipeline stages or data transformations. Verification requires comparing output datasets against expected results. The bootstrap script must provision data stores and seed them with test data. Progress tracking must account for the fact that pipeline stages have complex dependencies.

**Infrastructure as code.** Features are infrastructure resources or configurations. Verification requires actually provisioning infrastructure (or using mock providers). The cost of verification is potentially much higher, and rollback on failure is more complex.

**Machine learning.** Features might be model capabilities or performance thresholds. Verification is inherently statistical rather than deterministic. The concept of "passing" and "failing" becomes fuzzy — does a model accuracy of 94.8% count as passing when the target is 95%?

Each of these domains requires domain-specific adaptations of the core pattern, but the fundamental insight — *anchor to ground truth, work incrementally, verify before declaring done, leave a trail for the next session* — transfers universally.

### 6.6.3 The Boundary Between Harness and Model

A recurring theme in long-running agent design is the tension between solving problems through better prompting versus solving them through better tooling. Consider the "don't modify tests" instruction. This can be enforced through:

1. **Prompting:** Tell the agent not to do it, emphatically.
2. **File permissions:** Make test files read-only at the OS level.
3. **Git hooks:** Reject commits that modify files in the `tests/` directory.
4. **Architectural separation:** Run tests in a separate read-only filesystem mount.

Each approach trades off flexibility, reliability, and complexity differently. Prompt-based enforcement is the simplest and works surprisingly well, but it can be overridden by a sufficiently confused or context-pressured model. File permissions are more robust but prevent legitimate test additions. Git hooks strike a reasonable balance but require the agent to understand why its commit was rejected. Architectural separation is the most robust but the most complex to set up.

The general principle emerging from research is: *use prompting for guidance and tooling for guardrails*. Tell the agent what to do through the prompt; prevent catastrophic failures through the harness. The getting-up-to-speed protocol is an example of guidance. The idempotent `init.sh` script is an example of a guardrail. The feature list in JSON is both — it guides the agent's work selection while its rigid format guards against accidental modification.

---

## 6.7 Practical Implementation Checklist

For teams implementing long-running agent systems, the following checklist summarizes the critical infrastructure:

1. **Initializer prompt** is distinct from the coding prompt and runs exactly once.
2. **`init.sh`** is idempotent and brings the environment from zero to running.
3. **Feature list** is in JSON format with status tracking and priority ordering.
4. **`claude-progress.txt`** exists and is appended to at the end of every session.
5. **Git is used as ground truth,** with descriptive commit messages for every unit of progress.
6. **The getting-up-to-speed protocol** is encoded in the coding agent prompt.
7. **End-to-end tests** run before new work begins in each session.
8. **Browser automation** (or equivalent domain-specific verification) is available for feature verification.
9. **Test integrity** is protected through both prompting and tooling.
10. **Session scope** is limited to one feature at a time.

---

## 6.8 Summary

Long-running agents present a fundamental challenge: maintaining coherent progress on complex tasks across context boundaries where the agent has no memory of previous sessions. The solution is not better memory — it is better infrastructure. By splitting the work into an initializer agent that creates the scaffolding for success and a coding agent that follows a disciplined protocol for incremental progress, we can achieve reliable multi-session task completion.

The key insights are:

- **Ground truth lives in the file system, not in the model's context.** Git logs, progress files, test results, and the feature list are the authoritative record of project state.
- **Incremental progress with verification beats ambitious one-shot attempts.** One fully implemented, tested, committed feature per session is better than three half-implemented features.
- **The agent's self-assessment of correctness is unreliable.** External verification through browser automation or comprehensive test suites is essential.
- **The harness does the heavy lifting.** The prompts provide guidance, but the infrastructure — the bootstrap script, the feature list format, the git history, the testing tools — provides the guardrails that make long-running agents practical.

In the next chapter, we will examine how feedback loops and verification systems can be designed to replace human QA entirely — turning the agent from a developer who needs code review into a developer who reviews their own code with the rigor of a hostile critic.
