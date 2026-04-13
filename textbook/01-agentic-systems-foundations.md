# Chapter 1: Foundations — Agentic Systems

> "The purpose of abstraction is not to be vague, but to create a new semantic level in which one can be absolutely precise."
> — Edsger W. Dijkstra

Before you can design a harness, you must understand the thing being harnessed. A civil engineer who designs bridges without understanding the properties of steel and concrete will produce structures that collapse. A harness engineer who designs constraints and feedback loops without understanding how agentic systems work — their architecture, their patterns of composition, their failure modes — will produce harnesses that are either too rigid to be useful or too loose to be safe.

This chapter provides a rigorous foundation in agentic systems: what they are, how they are structured, the canonical patterns for composing them, and — critically — *when to use which pattern*. By the end, you will have a precise taxonomy that ranges from a single LLM call to a fully autonomous agent, and you will understand the tradeoffs at every point on the spectrum.

We will be precise about terminology, because imprecise terminology leads to imprecise thinking, which leads to poorly designed harnesses. When we say "agent," we mean something specific. When we say "workflow," we mean something different. The distinction is not pedantic. It has concrete implications for how you design your harness.

---

## Agents vs. Workflows: The Fundamental Distinction

The most important conceptual distinction in agentic systems — and the one most frequently confused — is the distinction between **agents** and **workflows**. Anthropic draws this line clearly, and we will adopt their formulation throughout this book:

- **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**. The developer determines the control flow at design time. The LLM operates within rails that are fixed before any input is processed.

- **Agents** are systems where LLMs **dynamically direct their own processes and tool usage**. The LLM determines the control flow at runtime. The sequence of operations emerges from the LLM's reasoning about the specific input, the results of its actions, and the state of the environment.

This distinction maps onto a concept from computer science that you already know: the difference between a **finite state machine** and a **Turing machine**. A workflow, like a finite state machine, has a fixed set of states and transitions defined in advance. An agent, like a Turing machine, can enter states and follow transitions that were not anticipated by the designer.

Let us make this concrete.

### A Workflow: Code Review Pipeline

Consider an automated code review system implemented as a workflow:

```
Diff → [Step 1: Summarize changes] → [Step 2: Check for security issues]
     → [Step 3: Check for performance issues] → [Step 4: Generate review comment]
```

Every diff follows this exact sequence. Step 1 always runs. Step 2 always runs after Step 1. The developer who built this system decided at design time that these four steps, in this order, constitute a code review. The LLM at each step has a specific job, receives specific input, and produces specific output. The control flow is *extrinsic* to the LLM — it lives in the orchestration code.

This is a workflow. It is predictable, debuggable, and testable. You know exactly how many LLM calls it will make (four). You know exactly what each call will cost. You can write integration tests that verify the pipeline's behavior end-to-end.

### An Agent: Autonomous Bug Fixer

Now consider an autonomous bug-fixing system:

```
Bug report → [Agent decides what to do next] → [Executes action] → [Observes result]
                    ↑_______________________________________________|
                    (loops until bug is fixed or agent gives up)
```

The agent receives a bug report. What happens next depends entirely on the bug:

- It might read the relevant source files.
- It might search the codebase for related code.
- It might write a reproduction test.
- It might examine log files.
- It might modify code and run tests.
- It might discover that the bug is actually in a dependency and decide to work around it rather than fix it.

No two bug reports will produce the same sequence of actions. The control flow is *intrinsic* to the LLM — it emerges from the LLM's reasoning. The developer who built this system did not specify the sequence of operations. They specified the *capabilities* (tools the agent can use) and the *goal* (fix the bug), and the agent determines the path.

This is an agent. It is flexible, powerful, and capable of handling novel situations — but it is also unpredictable, expensive (you do not know how many LLM calls it will make), and harder to debug (when it fails, you must trace through its entire decision history to understand why).

### Why the Distinction Matters for Harness Design

The distinction between workflows and agents has direct, practical implications for harness design:

| Dimension | Workflows | Agents |
|---|---|---|
| **Predictability** | High. You know the steps in advance. | Low. The steps emerge at runtime. |
| **Cost control** | Easy. Fixed number of LLM calls. | Hard. Variable and unbounded without limits. |
| **Testing** | Straightforward. Test each step and the pipeline. | Difficult. Must test decision-making in diverse scenarios. |
| **Debugging** | Simple. Check each step's output. | Complex. Must trace the full decision history. |
| **Guardrails needed** | Fewer. The fixed structure is itself a guardrail. | More. The agent's freedom requires more constraints. |
| **Documentation needed** | Less. Each step has narrow, well-defined context needs. | More. The agent must have broad context to make good decisions. |
| **Failure modes** | Predictable. Each step can fail in known ways. | Emergent. Novel failure modes can appear in novel situations. |

This table should inform every harness design decision you make. If you are building a harness for a workflow, you can focus your effort on the quality of each step's prompt and the gates between steps. If you are building a harness for an agent, you need to invest heavily in guardrails, comprehensive documentation, robust feedback loops, and cost controls.

### This Is Not a Value Judgment

Workflows are not inferior to agents. In many cases, workflows are the *correct* choice — they are more predictable, more debuggable, more efficient, and more reliable. The agent pattern is warranted only when the task genuinely requires dynamic decision-making that cannot be anticipated at design time.

The seductive error is to reach for agents because they feel more "advanced" or "intelligent." This is the **complexity bias** — the tendency to prefer complex solutions because they seem more impressive. Resist it. The OpenAI experiment succeeded not because they used the most sophisticated agent architecture available, but because they used the *right level of sophistication for each task* — and consistently erred on the side of simplicity.

**Start simple. Add complexity only when it demonstrably improves outcomes.** This is not a platitude. It is a design principle with teeth. Every time you reach for a more complex pattern, you should be able to articulate the specific failure mode of the simpler pattern that the more complex pattern addresses. If you cannot, you are over-engineering.

---

## The Augmented LLM: The Basic Building Block

Before we examine the workflow patterns and agent architectures in detail, we need to understand the fundamental unit from which they are all constructed: the **augmented LLM**.

A raw LLM is a function from text to text. It takes a prompt (a sequence of tokens) and returns a completion (another sequence of tokens). This is useful but limited. A raw LLM cannot look things up, take actions in the world, or remember what happened in previous conversations.

An **augmented LLM** extends this basic capability with three types of external integration that transform it from a text-completion engine into something that can *do work*:

### 1. Retrieval: The LLM's Library

Retrieval allows the LLM to pull in information from external sources at inference time. This is the family of techniques commonly grouped under **Retrieval-Augmented Generation (RAG)**:

- **Vector search** over a document corpus, codebase, or knowledge base. Documents are embedded as vectors, and the system retrieves documents whose embeddings are close to the query's embedding in vector space. This is effective for semantic similarity ("find code that does something similar to X") but can miss exact matches.
- **Keyword search** with BM25 or similar sparse retrieval methods. This is the traditional information retrieval approach: exact and partial string matching, term frequency weighting, document length normalization. It excels at finding exact terms ("find all files that import `UserService`") but misses semantic paraphrases.
- **Structured queries** against databases, APIs, or configuration stores. This is not "search" in the traditional sense — it is precise data retrieval based on known schemas and keys.
- **File system access** — reading specific files, listing directories, examining file metadata. In software engineering contexts, this is often the most important retrieval mechanism.

The design principle for retrieval is **precision over recall in context-limited settings**. An LLM's context window is finite. Every retrieved document that is irrelevant wastes tokens and dilutes the LLM's attention. It is better to retrieve three highly relevant documents than thirty loosely relevant ones.

For harness engineering, retrieval is how you make documentation, architecture decisions, coding conventions, and historical context available to the agent *at the point of need*. Rather than stuffing everything into a massive system prompt (which wastes tokens and dilutes attention), you design retrieval mechanisms that surface the right information for the specific task at hand.

The analogy is a well-organized physical library versus a pile of books on a desk. The pile gives you "all the information," but you cannot find anything. The library has less information immediately visible, but everything is findable when you need it.

### 2. Tools: The LLM's Hands

Tools allow the LLM to take actions in the world. The LLM "calls" a tool by emitting a structured output (typically JSON conforming to a schema) that the runtime environment intercepts, executes, and returns the result of.

Common tools in software engineering contexts include:

- **File operations**: reading files, writing files, editing specific portions of files, creating new files, deleting files, listing directory contents.
- **Code execution**: running code in a sandbox — executing test suites, running scripts, compiling programs.
- **Shell commands**: executing arbitrary commands in a terminal — git operations, package manager commands, build system invocations.
- **Search**: querying search engines, code search indices, documentation search, or grep-like pattern matching across a codebase.
- **API calls**: interacting with external services — creating GitHub issues, posting Slack messages, querying monitoring systems.
- **Browser automation**: navigating web pages, filling forms, extracting content from web applications.

Tools are how the agent *acts*. Without tools, an LLM can only produce text. With tools, it can modify codebases, run tests, deploy services, query databases, and interact with the full surface area of a modern software system.

The design of tools is a critical harness engineering concern that we will explore in depth later. For now, the key insight is that **the agent can only do what its tools allow it to do, and it can only do it well if the tools are well-designed**. A tool with a confusing name, an unclear parameter schema, or unhelpful error messages is a bug in your harness — it will cause agent failures just as surely as a bug in your code causes runtime failures.

### 3. Memory: The LLM's Notebook

Memory allows the LLM to persist information across interactions. Without memory, each LLM call is stateless — the LLM has no idea what it did five seconds ago unless you tell it.

Memory comes in several forms, each with different persistence and scope:

- **Conversational memory**: the message history within a single session. This is the most basic form — the LLM sees all previous messages in the conversation. It is limited by context window size: when the conversation exceeds the window, older messages must be summarized or dropped.

- **Short-term working memory**: scratchpads, intermediate results, and state that persists within a task but not across tasks. For example, an agent working on a bug fix might maintain a "scratchpad" file where it records its hypotheses, observations, and plan. This is distinct from conversational memory because it is *explicit* and *structured* — the agent deliberately writes information to the scratchpad and reads it back later.

- **Long-term memory**: information that persists across sessions — learned preferences about the codebase, facts about the architecture, historical decisions, patterns observed in previous tasks. This is typically implemented through external storage (databases, files, vector stores) combined with retrieval mechanisms. The agent writes memories to the store during one session and retrieves relevant ones in subsequent sessions.

Memory is what transforms a stateless text-completion engine into something that can maintain coherence across a complex, multi-step task. It is also one of the hardest things to get right. The agent must decide *what* to remember (not everything is worth persisting), *how* to organize it (unstructured memory is hard to retrieve), and *when* to retrieve it (retrieving irrelevant memories wastes context).

### The Augmented LLM as a Graph Node

The mental model: think of each augmented LLM as a **node in a directed graph**. Each node has:

- **Inputs**: the prompt, retrieved context, injected memories, and any state passed from upstream nodes.
- **Processing**: the LLM's reasoning, which may include multiple internal steps (chain-of-thought, reflection, self-correction).
- **Tool calls**: zero or more invocations of external tools, whose results feed back into the LLM's reasoning within the same node activation.
- **Outputs**: the final response — text, structured data, tool call requests, or a combination.

Workflow patterns and agent architectures are different ways of *connecting* these nodes. The patterns differ in **who controls the connections** — the developer (workflows) or the LLM (agents) — and in **how many nodes** are involved.

This graph-node mental model will recur throughout the book. When you are designing a harness, you are designing the environment that each node operates in: the retrieval sources it can access, the tools it can invoke, the memory it can read and write, the constraints on its outputs, and the feedback it receives about the quality of those outputs.

---

## Workflow Pattern 1: Prompt Chaining

**Prompt chaining** decomposes a task into a fixed sequence of steps, where each step is a separate LLM call and the output of one step becomes the input to the next. Between steps, you can insert programmatic **gates** — validation checks that determine whether the chain should continue, retry the current step, or abort entirely.

### The Pattern

```
Input → [LLM Step 1] → Gate₁ → [LLM Step 2] → Gate₂ → [LLM Step 3] → Output
```

Each step has a focused, well-defined subtask. The gates between steps enforce quality constraints before passing results downstream. Gates can be simple (check that the output is valid JSON) or complex (run a suite of validation tests against the output).

This pattern is an application of the **Pipes and Filters** architectural pattern from software architecture: each step is a filter that transforms its input, and the gates are the pipes that connect them with optional validation.

### When to Use It

Prompt chaining is ideal when:

- The task can be cleanly decomposed into **sequential subtasks** where each subtask's output feeds the next.
- Each subtask is **simple enough** for a single LLM call to handle reliably.
- You want to **validate intermediate results** before proceeding — catching errors early rather than discovering them in the final output.
- You need a **clear audit trail** of how the final output was produced.
- The **decomposition is known in advance** — you do not need the LLM to decide what the steps are.

### Example: Automated Code Review

Consider automating a structured code review for a pull request. The task is too complex for a single prompt (you would need security analysis, performance analysis, correctness analysis, and style analysis all at once), but it decomposes naturally into sequential steps:

**Step 1 — Structural Summary:** The LLM receives the diff and produces a structured summary: which files were changed, what type of change each represents (new feature, bug fix, refactor, test, documentation), which subsystems are affected, and the estimated scope of change (trivial, small, medium, large).

**Gate 1:** Validate that the summary is structurally complete. Does it reference all changed files? Is the change-type classification from the permitted set of values? Is the scope estimate one of the allowed values? If validation fails, retry Step 1 with a corrective prompt that includes the validation errors. If it fails three times, abort and flag for human review.

**Step 2 — Issue Analysis:** The LLM receives the diff plus the validated summary from Step 1. For each changed file, it identifies potential issues: bugs, security vulnerabilities, performance concerns, missing error handling, missing tests, convention violations. Each issue includes a severity rating (critical, major, minor, suggestion), a specific line reference, and a brief explanation.

**Gate 2:** Check that the analysis covers all files listed in the summary. Verify that each issue has the required fields (severity, line reference, explanation). Filter out low-confidence findings — issues where the LLM's explanation contains hedging language ("might," "possibly," "could") below a confidence threshold.

**Step 3 — Review Synthesis:** The LLM receives the original diff, the summary, and the filtered analysis. It produces a cohesive review comment: an executive summary ("This PR adds user preference storage with a new database migration, model, and API endpoint. Three issues were identified, one critical."), followed by the issues organized by severity, each with a concrete suggestion for resolution.

### Example: Database Migration Generation

A more harness-relevant example — generating database migrations:

**Step 1 — Schema Analysis:** The LLM reads the current database schema and the requested change (e.g., "add a `preferences` JSONB column to the `users` table with a default of `{}`"). It produces a structured migration plan: what SQL statements are needed, what indices should be created, what constraints should be added, and whether the migration is reversible.

**Gate 1:** Validate the migration plan against the team's migration conventions: all new columns must have defaults, all migrations must be reversible, no column drops without a data migration plan, no full table locks on tables with more than 1M rows.

**Step 2 — SQL Generation:** The LLM generates the actual migration SQL (up and down) based on the validated plan.

**Gate 2:** Parse the generated SQL. Verify it is syntactically valid. Run it against a schema-only copy of the database to verify it executes without errors. Check that the down migration reverses the up migration (apply up, then down, and verify the schema matches the original).

**Step 3 — Test Generation:** The LLM generates test cases for the migration: a test that the up migration succeeds on a clean database, a test that the down migration reverses it, and edge case tests for any constraints or defaults.

### The Tradeoff

Prompt chaining trades **latency for reliability**. Each step adds an LLM call (and thus latency and cost), but the decomposition makes each individual call simpler and more likely to succeed. The gates catch errors before they propagate, which is far cheaper than debugging an error in the final output and trying to figure out which step went wrong.

The key design decision is **step granularity**: how fine should the steps be?

- **Too fine**: You incur unnecessary overhead and lose the LLM's ability to reason holistically about the task. If each step is a single sentence of analysis, the LLM cannot consider interactions between steps.
- **Too coarse**: You lose the benefits of decomposition. If the entire code review is one step, you are back to a single LLM call with all its reliability issues.

A useful heuristic: **each step should be something you could evaluate independently.** If you cannot write a clear gate condition between two adjacent steps, they probably should be a single step. If a step is doing two things that could be evaluated separately, it should probably be two steps.

---

## Workflow Pattern 2: Routing

**Routing** classifies the input and directs it to a specialized handler — a different prompt, a different model, a different pipeline, or even a non-LLM code path. It is the **Strategy Pattern** from object-oriented design applied to LLM systems: the routing classifier selects the appropriate strategy for processing the input.

### The Pattern

```
                          → [Handler A] →
Input → [Classifier] ──  → [Handler B] →  → Output
                          → [Handler C] →
```

The classifier examines the input and determines which handler is appropriate. Each handler is optimized for its specific category of input — it may use a different system prompt, different tools, different retrieval sources, or even a different model.

### When to Use It

Routing is ideal when:

- Your inputs fall into **distinct categories** that benefit from different processing strategies.
- You want to use **cheaper or faster models** for simple cases and reserve expensive models for complex ones (the **tiered model** strategy).
- Different categories require **different tools, context, or expertise** — and providing all tools and context to every request would be wasteful or confusing.
- You need to **separate concerns cleanly** for maintainability — each handler can be developed, tested, and evolved independently.

### Example: Multi-Tier Developer Support System

Consider an internal developer support system that handles requests from engineers:

**Classifier:** An LLM (or even a traditional ML classifier — routing classifiers do not need to be LLMs, and simpler classifiers are often more reliable and faster) examines the incoming request and categorizes it:

- **Documentation query**: "How do I configure the rate limiter for the payments service?" Route to a RAG pipeline that searches the documentation corpus and generates an answer with source citations.
- **Bug report**: "The checkout API returns 500 when the cart has more than 100 items." Route to a diagnostic agent that can query logs, examine recent deployments, check monitoring dashboards, and correlate symptoms with known issues.
- **Code generation request**: "Create a new endpoint for exporting user data as CSV." Route to a code generation workflow (probably an orchestrator-worker pattern) that can examine the existing codebase patterns, generate code, and run tests.
- **Infrastructure issue**: "Staging environment is returning 502 errors." Route to a runbook executor that follows predefined incident response procedures, checking load balancers, service health, and recent deployments.

Each handler is purpose-built for its category. The documentation query handler has access to the vector store and documentation index. The bug report handler has access to logging infrastructure and monitoring dashboards. The infrastructure handler has access to deployment tools and runbooks. None of them need *all* of these capabilities — routing ensures each handler has exactly the tools and context it needs, which reduces token usage, lowers cost, and — critically — reduces the probability that the LLM gets confused by irrelevant tools.

### Example: Model Tier Routing

A cost-optimization application of routing:

**Classifier:** Assess the complexity of the incoming coding task on a scale of 1-5.

- **Complexity 1-2** (boilerplate, simple CRUD, straightforward bug fixes): Route to a smaller, faster, cheaper model (e.g., Claude Haiku or GPT-4o-mini). These tasks do not require deep reasoning.
- **Complexity 3-4** (multi-file changes, non-trivial logic, performance optimization): Route to a mid-tier model (e.g., Claude Sonnet). These tasks need strong coding ability but not maximum reasoning.
- **Complexity 5** (architectural changes, complex debugging, security-critical code): Route to the most capable model (e.g., Claude Opus). These tasks justify the higher cost because the cost of an error is high.

This pattern can reduce costs by 60-80% for codebases where the majority of tasks are low to moderate complexity, with no loss in quality — because the cheap model is *adequate* for simple tasks.

### The Tradeoff

Routing trades **generality for specialization**. A single general-purpose agent could handle all cases, but it would need access to all tools, all context, and all instructions simultaneously — which means a larger prompt, more potential for confusion, higher cost per request, and more failure modes.

Routing is an application of the **Single Responsibility Principle**: each handler does one thing well. But the principle has a cost: the classification step itself can fail. If the classifier routes a request to the wrong handler, the output will be poor — and the failure mode is insidious, because the handler will confidently produce an answer to the wrong question. The user sees a well-formatted, plausible-looking response that is completely wrong.

Robust routing requires:

- A **high-quality classifier** with clear, mutually exclusive category definitions. Ambiguous categories are the primary source of misclassification.
- A **fallback path** for ambiguous inputs: route to a general-purpose handler, ask the user for clarification, or process through multiple handlers and let the user choose.
- A **confidence threshold**: if the classifier's confidence is below a threshold, do not route — escalate to a human or use the fallback path.
- **Monitoring** that tracks classification accuracy over time. Classification drift — where the distribution of inputs changes and the classifier's accuracy degrades — is a real operational concern.

---

## Workflow Pattern 3: Parallelization

**Parallelization** runs multiple LLM calls simultaneously and aggregates their results. There are two distinct sub-patterns, each with different motivations and tradeoffs.

### Sub-pattern A: Sectioning

**Sectioning** divides a task into independent subtasks that can be processed in parallel, then combines the results.

```
            → [Subtask A] →
Input ──    → [Subtask B] →    ── [Aggregator] → Output
            → [Subtask C] →
```

The key word is **independent**. Sectioning only works when the subtasks do not depend on each other — the security analysis does not need the performance analysis's output, and vice versa. If there are dependencies between subtasks, you need prompt chaining or orchestrator-workers, not sectioning.

**Example: Multi-Perspective Code Analysis**

You need to analyze a pull request from multiple angles. These perspectives are genuinely independent — each can be performed without knowledge of the others:

- **Branch A — Security Review:** The LLM, equipped with a security-focused system prompt and OWASP reference material, analyzes the diff for vulnerabilities: injection flaws, authentication bypasses, sensitive data exposure, insecure defaults.
- **Branch B — Performance Review:** The LLM, equipped with a performance-focused system prompt and the application's SLA requirements, analyzes the diff for performance regressions: N+1 queries, missing indices, unbounded allocations, unnecessary synchronous I/O.
- **Branch C — Correctness Review:** The LLM, equipped with the test suite and the feature specification, analyzes the diff for logical errors: incorrect conditionals, missing edge cases, race conditions, incorrect error handling.
- **Branch D — Style Review:** The LLM, equipped with the team's style guide, checks for convention violations: naming, formatting, comment quality, import ordering, module organization.

The aggregator combines the four analyses into a unified review. It deduplicates any overlapping findings (the security review and the correctness review might both flag the same unvalidated input), organizes by severity, and produces a single cohesive document.

Sectioning gives you **linear speedup**: a four-way parallel analysis takes roughly the same wall-clock time as a single analysis, but produces four times the depth. The total token cost is the same as running the analyses sequentially, but the latency is reduced by a factor equal to the degree of parallelism.

**Example: Parallel Test Generation**

You need to generate tests for a new module with five public functions. Each function's tests are independent of the others:

- **Branch 1-5:** Each branch generates tests for one function. Each branch receives the function signature, its documentation, the test framework conventions, and examples of existing tests in the codebase.

The aggregator combines the five test files, checks for naming conflicts, ensures consistent test setup/teardown, and produces the final test file.

### Sub-pattern B: Voting

**Voting** runs the *same* task multiple times — potentially with different prompts, different models, or different temperatures — and aggregates the results through a voting or consensus mechanism.

```
            → [Attempt 1] →
Input ──    → [Attempt 2] →    ── [Voter] → Output
            → [Attempt 3] →
```

**Example: High-Stakes Classification**

You need to determine whether a code change is a **breaking change** to a public API. This is a high-stakes binary classification — a false negative means users experience an unexpected breaking change and lose trust; a false positive means the team does unnecessary backwards-compatibility work. The cost of error in either direction is significant.

You run three independent classifications:

- **Attempt 1:** Standard prompt at temperature 0. Direct question: "Does this diff change any public API signatures, remove any public functions, or change the behavior of any existing function in a way that would break callers?"
- **Attempt 2:** Chain-of-thought prompt at temperature 0 that forces explicit reasoning: "List every public API element in the before-state. List every public API element in the after-state. For each element, compare signatures and documented behavior. Conclude whether any breaking change exists."
- **Attempt 3:** Adversarial prompt at temperature 0.3: "You are a developer who depends on this library. Your CI just ran against the new version. Think about what tests might fail and why."

The voter examines the three outputs:
- **All three agree → High confidence.** Accept the classification.
- **Two of three agree → Moderate confidence.** Accept the majority but flag the disagreement in the review.
- **All three disagree → Low confidence.** Escalate to a human reviewer.

Voting is an application of **ensemble methods** from machine learning. It increases reliability at the cost of multiplied compute. The theoretical justification is the **Condorcet jury theorem**: if each classifier is better than random, the majority vote of multiple independent classifiers is more reliable than any single classifier — and the reliability improves with the number of classifiers.

The word "independent" is doing real work in that sentence. If all three attempts use the same prompt and model, their errors will be correlated, and voting provides less benefit. Using diverse prompts, models, and temperatures produces more independent errors and thus more effective ensembles.

### The Tradeoff

Parallelization trades **compute cost for either speed (sectioning) or reliability (voting)**:

- **Sectioning** multiplies tokens spent but divides wall-clock time. Use it when individual LLM calls are the latency bottleneck and the subtasks are genuinely independent.
- **Voting** multiplies tokens spent *and* may not reduce latency (all attempts run the same task). Use it when the cost of an incorrect output is high enough to justify 2-5x the inference cost.

Both sub-patterns are most valuable when the per-call cost is small relative to the value of the outcome. If an LLM call costs $0.05 and a breaking-change misclassification costs $10,000 in engineer time and user trust, running five parallel classifications at $0.25 total is an obviously good investment.

---

## Workflow Pattern 4: Orchestrator-Workers

The **orchestrator-worker** pattern uses a central LLM (the orchestrator) to dynamically break down a task and delegate subtasks to worker LLMs. Unlike prompt chaining, where the decomposition is fixed at design time, the orchestrator decides the decomposition at runtime based on the specific input.

This is the **Mediator Pattern** from object-oriented design: a central coordinator that manages interactions between components, with the components themselves unaware of each other.

### The Pattern

```
                                → [Worker A] →
Input → [Orchestrator] ──      → [Worker B] →    ── [Orchestrator] → Output
                                → [Worker C] →
```

The orchestrator has three responsibilities:

1. **Planning:** Analyze the input, determine what subtasks are needed, and define each subtask precisely.
2. **Delegation:** Dispatch subtasks to workers, providing each worker with the specific context and constraints it needs.
3. **Synthesis:** Receive worker outputs, check them for consistency and completeness, and combine them into the final output. If workers produce conflicting or incomplete results, the orchestrator resolves conflicts or dispatches correction tasks.

### When to Use It

Orchestrator-workers is ideal when:

- The decomposition of the task **depends on the input** and cannot be determined in advance. You do not know how many subtasks there will be or what they will be until you see the specific input.
- The number and type of subtasks **varies across inputs**. One input might require three workers; another might require twelve.
- Subtasks may need to be **sequenced dynamically** — the output of one worker may determine what other workers need to do.
- The task is **too complex for a single LLM call** but the decomposition is too input-dependent for a fixed prompt chain.

### Example: Multi-File Feature Implementation

A developer requests: "Add a new API endpoint for user preferences, including database migration, model, controller, route registration, tests, and API documentation."

A prompt chain *could* handle this, but the exact set of files to create and modify depends on the codebase — which framework is being used, where models live, how routes are organized, what the testing conventions are, whether there is an API documentation generator. The orchestrator handles this dynamically:

**Orchestrator — Planning Phase:** The orchestrator examines the codebase structure using file listing and code search tools. It identifies the existing patterns:

- Migrations live in `db/migrations/` and follow the naming convention `YYYYMMDD_HHMMSS_description.sql`.
- Models are in `src/models/` and use the Active Record pattern.
- Controllers are in `src/controllers/` and follow RESTful conventions.
- Routes are registered in `src/routes/api.go` using a declarative routing table.
- Tests mirror the source structure under `tests/`.
- API docs are auto-generated from OpenAPI annotations in the controller files.

Based on this analysis, the orchestrator produces a plan:

1. Create migration: `db/migrations/20260322_120000_add_user_preferences.sql`
2. Create model: `src/models/user_preference.go`
3. Create controller: `src/controllers/preferences_controller.go` (with OpenAPI annotations)
4. Modify routes: add preference routes to `src/routes/api.go`
5. Create tests: `tests/controllers/preferences_controller_test.go`
6. Create tests: `tests/models/user_preference_test.go`

**Workers — Execution Phase:** The orchestrator dispatches each task to a worker. Each worker receives:

- The specific file to create or modify.
- Relevant context: existing code patterns from similar files (e.g., the worker creating the preferences controller receives the existing payments controller as a reference), the database schema, the API specification.
- Constraints: coding standards, naming conventions, error handling patterns, test coverage requirements.

Workers execute in parallel where possible. The migration and model can be written in parallel. The controller depends on the model (it needs to know the model's field names and types). The route registration depends on the controller (it needs to know the handler function names). The tests depend on the code they test.

The orchestrator manages these dependencies, dispatching workers in the correct order and passing upstream outputs to downstream workers.

**Orchestrator — Synthesis Phase:** The orchestrator reviews the workers' outputs for consistency:

- Does the controller reference the model's fields correctly?
- Do the routes match the controller's handler methods?
- Do the tests exercise the specified behavior?
- Are the OpenAPI annotations consistent with the actual request/response types?
- Does the migration create the columns that the model expects?

If inconsistencies are found, the orchestrator dispatches correction tasks to the relevant workers: "The controller references `user_preference.Settings` but the model defines the field as `user_preference.Preferences`. Update the controller to use the correct field name."

### Example: Codebase-Wide Refactor

The task: "Rename the `UserManager` class to `UserService` across the entire codebase, updating all imports, references, documentation, and tests."

**Orchestrator — Planning Phase:** The orchestrator searches the codebase for all references to `UserManager`:

- 47 files import `UserManager`
- 12 files reference `UserManager` in comments or documentation
- 3 configuration files reference `UserManager`
- The class definition is in `src/services/user_manager.go`

**Workers — Execution Phase:** The orchestrator dispatches workers to handle each file. For simple changes (updating an import statement), a single worker handles multiple files. For complex changes (the class definition itself, which requires renaming methods that contain "Manager" in their names), a dedicated worker handles each file with additional context about what should and should not be renamed.

**Orchestrator — Synthesis Phase:** The orchestrator verifies that all 62 references have been updated, that no new references to `UserManager` appear in the modified files, and that the test suite passes.

### The Tradeoff

Orchestrator-workers trades **predictability for flexibility**. The dynamic decomposition handles novel inputs well, but it introduces several costs:

- **Planning quality is a single point of failure.** If the orchestrator produces a poor plan (misses a file, creates wrong dependencies, misunderstands the task), all downstream work is wasted. This is why the orchestrator typically needs to be a more capable (and more expensive) model than the workers.
- **Cost is unpredictable.** You do not know in advance how many workers will be needed or how many iterations of synthesis/correction will occur.
- **Debugging is harder.** When the final output has an error, you must determine whether it was a planning error (orchestrator), an execution error (worker), or a synthesis error (orchestrator again). The trace is longer and more branching than a linear prompt chain.

This pattern is closely related to **MapReduce** in distributed computing: the orchestrator's planning phase is the map (decompose into subtasks), the workers are the distributed computation, and the synthesis phase is the reduce (combine results). It is also an instance of the **Divide and Conquer** algorithmic pattern: break the problem into smaller subproblems, solve them independently, and combine the solutions.

---

## Workflow Pattern 5: Evaluator-Optimizer

The **evaluator-optimizer** pattern creates a tight feedback loop between two LLMs (or two invocations of the same LLM with different prompts): one that **generates** output and one that **evaluates** it. The generator iterates on its output based on the evaluator's feedback until the evaluator accepts the output or a maximum iteration count is reached.

### The Pattern

```
Input → [Generator] → Output → [Evaluator] → Feedback
              ↑                                    |
              |____________________________________|
              (loop until accepted or max iterations)
```

This is the **Generate-and-Test** pattern from AI, combined with the **iterative refinement** approach from optimization. It is also structurally identical to the code review loop that every software engineer is familiar with: submit a PR, receive feedback, revise, resubmit, repeat until approved.

### When to Use It

Evaluator-optimizer is ideal when:

- The **quality of the output can be meaningfully assessed** by an LLM (or by a combination of LLM evaluation and programmatic checks).
- There is a clear notion of **"good enough"** that can be expressed as evaluation criteria.
- **Iterative refinement is likely to improve the output** — the task is one where feedback is actionable and subsequent attempts tend to be better. (Not all tasks have this property. If the generator's errors are random rather than systematic, iteration will not help.)
- **First-pass quality is insufficient** for your needs, but the generator is capable of producing adequate output given enough guidance.

### Example: Code Generation with Quality Gates

The task is to generate a function that parses a complex, semi-structured log format used by a legacy system.

**Generator — Iteration 1:** Produces an initial implementation based on the specification and example log entries.

**Evaluator — Iteration 1:** Reviews the implementation against multiple criteria:

- *Correctness:* Does it correctly parse all provided example log entries? (The evaluator runs the code against the examples using a code execution tool and checks the output.)
- *Edge cases:* Does it handle malformed input gracefully? Empty lines? Log entries that span multiple lines (e.g., stack traces)?
- *Performance:* Is the approach efficient? Is it doing anything quadratic? Does it compile the regex on every invocation instead of caching it?
- *Style:* Does it follow the codebase's conventions for error handling, naming, and documentation?

The evaluator produces structured feedback: "The implementation fails on multiline log entries. It assumes each log entry is a single line, but the specification shows that Java stack traces produce multi-line entries. The parser needs to accumulate lines until it sees the next timestamp prefix `[YYYY-MM-DD HH:MM:SS]`. Additionally, the regex for timestamp parsing is compiled inside the loop — move it to a package-level variable for efficiency."

**Generator — Iteration 2:** Receives its original implementation plus the evaluator's feedback. Produces a revised implementation that handles multiline entries and moves the regex compilation.

**Evaluator — Iteration 2:** All examples now parse correctly. The regex is compiled once. But: "The function returns a `[]string` but the specification calls for a `[]LogEntry` struct with separate fields for timestamp, level, source, and message. The return type needs to change."

**Generator — Iteration 3:** Defines the `LogEntry` struct and updates the function to return structured data.

**Evaluator — Iteration 3:** Accepts. All criteria pass.

### Example: Documentation Generation

The task is to generate architecture documentation for a module based on its source code.

**Generator — Iteration 1:** Produces initial documentation based on reading the module's files.

**Evaluator — Iteration 1:** Reviews against the documentation template and quality criteria: "The documentation describes *what* the module does but not *why* it exists or what alternatives were considered. The dependency section lists the direct dependencies but does not explain the rationale for choosing Redis over Memcached for the cache layer. The 'Getting Started' section assumes the reader has already set up the development environment, but the documentation template requires self-contained setup instructions."

**Generator — Iteration 2:** Revises with the structural and content feedback.

**Evaluator — Iteration 2:** "The rationale section is good. The setup instructions are complete. However, the API reference section is missing — the template requires a table of all public functions with their signatures, parameters, return types, and one-line descriptions."

**Generator — Iteration 3:** Adds the API reference table. Evaluator accepts.

### The Tradeoff

Evaluator-optimizer trades **compute cost and latency for output quality**. Each iteration requires at least two LLM calls (one for generation, one for evaluation), and the number of iterations is unpredictable. Three iterations is common; ten is possible; infinite loops can occur if the evaluation criteria are contradictory or if the generator cannot satisfy them.

The critical design decisions are:

1. **Evaluation criteria.** If the criteria are too vague ("make it better"), the evaluator will produce unhelpful feedback and the loop will oscillate without converging. If the criteria are too strict ("achieve 100% test coverage"), the loop may never terminate. The sweet spot is criteria that are *specific enough to be actionable* and *permissive enough to allow multiple valid solutions*.

2. **Maximum iteration count.** Always set one. An evaluator-optimizer without a maximum iteration count is a potential infinite loop. Three to five iterations is usually sufficient; beyond that, the marginal improvement per iteration tends to diminish.

3. **Convergence detection.** If the generator produces the same output twice (or the evaluator's feedback is the same as the previous iteration), the loop is stuck. Detect this and exit gracefully rather than burning through your iteration budget.

4. **Evaluation modality.** The evaluator does not have to be a pure LLM. The most effective evaluators combine LLM judgment with programmatic checks: the LLM evaluates clarity, completeness, and reasoning quality; the programmatic checks verify syntax, test passage, schema conformance, and other mechanically verifiable properties. This **hybrid evaluation** approach gets the best of both worlds — the richness of LLM evaluation and the reliability of programmatic checks.

The evaluator-optimizer pattern is deeply relevant to harness engineering because **the harness itself functions as an evaluator**. When an agent runs code and the tests fail, the test failures are evaluation feedback. When a linter flags issues, those are evaluation signals. The difference is that in the evaluator-optimizer pattern, an LLM performs the evaluation, which provides richer, more contextual feedback than a static tool — but at higher cost and lower determinism.

In a well-designed harness, the agent operates in an implicit evaluator-optimizer loop *with the harness as the evaluator*: the agent generates code, the harness (tests, linters, type checker) evaluates it, the agent revises based on the feedback, and this continues until all checks pass. The harness's feedback quality directly determines the efficiency of this loop — which is why designing fast, specific, actionable feedback is one of the highest-leverage harness engineering activities.

---

## Full Agents: Dynamic Tool Use in a Loop

We now arrive at the far end of the autonomy spectrum: **full agents**. An agent is an LLM that uses tools based on environmental feedback in a loop, with the LLM itself determining the control flow at each step.

### The Pattern

```
Input → [LLM decides action] → [Execute action] → [Observe result]
              ↑                                          |
              |__________________________________________|
              (loop until goal is achieved or abandoned)
```

This is the **observe-think-act loop**, also known as the **ReAct pattern** (after the seminal paper "ReAct: Synergizing Reasoning and Acting in Language Models" by Yao et al., 2022). At each iteration:

1. **Observe:** The agent examines the current state — tool outputs from the previous iteration, error messages, file contents, test results, or any other environmental feedback.
2. **Think:** The agent reasons about the goal, the current state, and the available actions. What has been accomplished? What remains? What is the best next step?
3. **Act:** The agent selects a tool, determines the arguments, and invokes it. Or, if the goal is achieved, the agent produces a final output and terminates.

The critical distinction from workflows: **steps 1-3 are not predefined**. The agent decides at each iteration which tool to use, what arguments to pass, and whether to continue or stop. The control flow is *emergent*, not *prescribed*. The developer provides the goal, the tools, and the constraints — the agent determines the path.

### When to Use Full Agents

Full agents are appropriate when:

- The task requires **exploration**: the agent does not know in advance which tools it will need, how many steps it will take, or what intermediate states it will encounter.
- The task involves **error recovery**: the agent must be able to detect failures, diagnose root causes, and try alternative approaches without human intervention.
- The task has **open-ended goals**: "fix this bug," "implement this feature," "refactor this module" — goals where the concrete steps depend on what the agent discovers as it works.
- The **environment is complex and varied**: different inputs may require fundamentally different approaches, and the set of possible approaches is too large to enumerate at design time.

### Example: Autonomous Bug Investigation and Fix

The agent receives a bug report: "The user profile API returns a 500 error when the user's display name contains emoji characters."

**Iteration 1 — Understand the problem:** The agent reads the bug report, identifies the relevant API endpoint (`/api/v1/users/{id}/profile`), and searches the codebase for the handler function.

**Iteration 2 — Locate the code:** The agent finds the handler in `src/controllers/user_controller.go`, reads it, and traces the code path that processes the display name. It identifies a call to a `sanitizeString()` utility function.

**Iteration 3 — Analyze the suspect function:** The agent reads `sanitizeString()` in `src/utils/strings.go`. The function uses a byte-level operation that assumes ASCII encoding — it iterates over individual bytes and rejects any byte with a value above 127. Emoji characters are multi-byte UTF-8 sequences with bytes above 127, so they trigger the rejection.

**Iteration 4 — Write a reproduction test:** The agent creates a test that calls the profile update endpoint with an emoji display name and asserts it should succeed (not return 500). It runs the test. The test fails with the expected 500 error, confirming the diagnosis.

**Iteration 5 — Implement the fix:** The agent modifies `sanitizeString()` to operate on runes (Unicode code points) instead of bytes. It preserves the function's intent (rejecting control characters and HTML-significant characters) while allowing emoji and other valid Unicode.

**Iteration 6 — Verify the fix:** The agent runs the reproduction test. It passes. The agent runs the full test suite.

**Iteration 7 — Handle regressions:** Two tests fail. The agent examines them. One is testing the old (incorrect) behavior — it expects `sanitizeString()` to reject a string containing a Japanese character. The agent updates this test to expect the character to be accepted. The other failing test is unrelated (a flaky test that fails intermittently due to a race condition). The agent notes this in its output but does not attempt to fix it.

**Iteration 8 — Final verification:** Full test suite passes (except the known flaky test). The agent creates a pull request with a description that explains the bug, the root cause, the fix, and the test changes, including a note about the pre-existing flaky test.

Notice the emergent complexity: the agent encountered two unexpected complications (a test asserting the old broken behavior, and a flaky test) and handled both appropriately. A workflow could not have anticipated these — the specific complications depend on the specific codebase state at the time of the fix.

### The Tradeoffs of Full Agents

Full agents offer maximum flexibility but come with significant costs that you must account for in your harness design:

**1. Unpredictable cost and latency.** You do not know in advance how many iterations the agent will take. A simple bug might be fixed in four iterations (read, reproduce, fix, verify). A complex bug might take thirty iterations as the agent explores blind alleys, tries and reverts approaches, and discovers that the root cause is three layers of indirection away from the symptom. Cost and time are proportional to iterations, and there is no reliable way to estimate them a priori.

*Harness implication:* You need cost controls — maximum iteration limits, per-task spending caps, and alerts when an agent exceeds expected bounds. You also need the ability to review agent traces and understand *why* a task took many iterations, so you can improve the harness to reduce iterations for similar future tasks.

**2. Compounding errors.** Each iteration builds on the results of previous iterations. If the agent makes a mistake in iteration 3 (e.g., misidentifies the root cause of a bug), iterations 4-7 may all be based on a false premise — the agent confidently modifying the wrong code, writing tests for the wrong behavior, and producing a PR that "fixes" a problem that was never the actual problem. This is the **error accumulation problem**, and it is the primary risk of agentic systems.

*Harness implication:* You need guardrails that catch errors early — comprehensive test suites that reveal incorrect fixes, linters that catch convention violations, and architectural constraints that prevent the agent from modifying code it should not touch. You also need review processes that are sensitive to this specific failure mode: when reviewing an agent's PR, start by checking the diagnosis, not the code.

**3. Difficult to debug.** When an agent produces incorrect output after fifteen iterations, figuring out *where it went wrong* requires reviewing the entire trace. Was the initial understanding correct? At which iteration did the reasoning go off track? Was it a tool failure, a reasoning error, or a missing piece of context?

*Harness implication:* You need comprehensive logging — every observation, every reasoning step, every tool invocation and result. The logs should be structured and searchable. You should be able to replay a specific iteration and understand the agent's state at that point.

**4. Hard to test.** Testing a workflow is relatively straightforward: you provide inputs and check outputs for each step. Testing an agent requires testing its *decision-making process* across a range of scenarios, which is non-deterministic and context-dependent. The same agent with the same tools might take different paths on different runs of the same task.

*Harness implication:* You need evaluation suites — collections of representative tasks with known-correct outcomes. You test the agent not on a single run but on its *distribution of outcomes* across many runs. If the agent fixes the bug correctly in 9 out of 10 runs, that may be acceptable. If it fixes it correctly in 3 out of 10 runs, your harness needs improvement.

---

## Three Core Principles

Across all of these patterns — from simple prompt chains to full agents — three principles consistently distinguish well-designed agentic systems from poorly designed ones. These principles are not guidelines. They are **load-bearing design constraints**. Violate them at your peril.

### Principle 1: Simplicity

**Use the simplest pattern that solves the problem.** Every additional layer of complexity — every additional LLM call, every additional tool, every additional decision point — is a potential failure mode, a source of latency, a contributor to cost, and a debugging burden.

This is not an abstract philosophical position. It has concrete engineering implications:

- **Prefer a single LLM call** over prompt chaining when the task fits in one step.
- **Prefer prompt chaining** over orchestrator-workers when the decomposition is known in advance.
- **Prefer a fixed workflow** over a full agent when the task does not require dynamic decision-making.
- **Prefer fewer tools** over more tools — each tool the agent does not need is noise in its decision space. An agent with access to 50 tools will make worse tool selection decisions than an agent with access to 5 relevant tools.
- **Prefer explicit programmatic gates** over LLM-based evaluation when the quality criteria can be checked mechanically (does the JSON parse? do the tests pass? is the function under 50 lines?).
- **Prefer no LLM at all** when a traditional program, a regex, a database query, or a shell script can solve the problem. LLMs are expensive, slow, and non-deterministic. Use them where their unique capabilities (natural language understanding, reasoning, code generation) are actually needed.

The temptation to over-engineer is strong. LLM-based agents are exciting technology, and there is a natural desire to use them everywhere. Resist it. The OpenAI experiment succeeded not because they used the most sophisticated agent architecture available, but because they used the *right level of sophistication for each task* — and erred on the side of simplicity.

The principle has a corollary: **start simple and add complexity incrementally, with evidence.** Begin with a single LLM call. If it is unreliable, try prompt chaining. If the decomposition is too input-dependent, try orchestrator-workers. If the task requires exploration, try a full agent. At each step, measure whether the additional complexity actually improves outcomes. If it does not, step back.

This is the **YAGNI principle** (You Aren't Gonna Need It) applied to agent architecture. Do not build a multi-agent orchestration system because you *might* need it. Build the simplest thing that works, and evolve it when you have evidence that you need more.

### Principle 2: Transparency

When an agentic system fails — and it will fail — you need to understand *why*. This requires transparency at every level of the system.

**Log everything.** Every LLM call should be logged: the full prompt, the full response, the latency, the token count, the cost. Every tool invocation should be logged: the tool name, the arguments, the result, and any errors. Every decision point should be logged: what the agent considered, what it chose, and why.

**Make reasoning visible.** Use chain-of-thought prompting so the agent's reasoning is externalized and inspectable. The agent's "thinking" tokens are the `printf`-debugging of agentic systems — they are the primary mechanism by which you understand what the agent is doing and why.

**Provide human-readable traces.** The raw logs should be transformable into a narrative that a human can follow: "The agent read the bug report. It searched for the relevant endpoint. It found the handler in `user_controller.go`. It traced the code path to `sanitizeString()`. It identified the byte-level operation as the root cause. It wrote a test. It modified the function. It ran the tests. Two tests failed. It investigated the failures..."

**Instrument for metrics.** Track quantitative signals that tell you where your harness needs improvement:

- **Iterations per task:** High iteration counts suggest the agent is struggling — either the task is poorly scoped, the tools are inadequate, or the documentation is missing critical context.
- **Tool usage patterns:** Which tools does the agent use most? Which does it never use? Which does it misuse? These patterns reveal ACI design problems.
- **Error rates by category:** Are failures predominantly tool failures (the tool returned unhelpful errors), reasoning failures (the agent made a logical mistake), or context failures (the agent lacked information it needed)?
- **Backtracking frequency:** How often does the agent undo previous work and try a different approach? Frequent backtracking suggests the agent's initial decisions are poorly informed.
- **Cost per task:** What is the distribution of costs across task types? Are some tasks disproportionately expensive?

Transparency is not just a debugging aid. It is a **trust mechanism**. When stakeholders — your team lead, your product manager, your security team — can see *how* the agent arrived at its output, they can make informed decisions about whether to accept it. An opaque system that produces correct output 95% of the time is less trustworthy than a transparent system that produces correct output 90% of the time, because the transparent system tells you *when to be skeptical*.

Transparency is also the foundation of **harness improvement**. You cannot improve what you cannot observe. If you do not know why the agent failed, you cannot fix the harness to prevent the failure. If you do not know which tools the agent struggles with, you cannot improve the ACI. If you do not know which types of tasks are expensive, you cannot optimize your task decomposition.

### Principle 3: Carefully Crafted Agent-Computer Interface (ACI)

The **Agent-Computer Interface** is the set of tools, commands, file structures, and conventions through which the agent interacts with the world. ACI design is to agentic systems what API design is to software systems — and it deserves the same level of rigor, iteration, and testing.

The core insight: **the agent can only do what its tools allow it to do, and it can only do it *well* if the tools are well-designed.** A poorly named tool, a confusing parameter schema, an unhelpful error message, a missing tool — these are not inconveniences. They are **bugs in your harness** that will cause agent failures just as surely as a null pointer dereference causes a runtime crash.

Good ACI design follows principles that are familiar from API design, but applied in a context where the consumer is an LLM rather than a human developer:

**Clear naming.** A tool called `search_codebase` is better than one called `search`, because the agent may have access to multiple search tools and needs to distinguish them. A tool called `create_file` is better than `write` when the action specifically creates a new file (as opposed to overwriting an existing one). The name should be a **verb-noun pair** that unambiguously describes the action: `read_file`, `run_tests`, `search_logs`, `create_branch`.

**Focused functionality.** Each tool should do one thing well. A tool that searches the codebase *and* formats the results is doing two things. A tool that reads a file *and* parses its AST is doing two things. Separate them. The **Single Responsibility Principle** applies to tools exactly as it applies to classes and functions.

**Rich, actionable error messages.** When a tool fails, the error message should tell the agent *what went wrong, why, and how to fix it*. Compare:

- Bad: `Error`
- Better: `File not found: /src/models/user.go`
- Best: `File not found: /src/models/user.go. The /src/models/ directory contains: user_model.go, preference_model.go, session_model.go. Did you mean user_model.go?`

The third message does not just report the error — it provides the context the agent needs to recover. This is the difference between a tool that causes the agent to waste three iterations guessing file names and a tool that enables recovery in one iteration.

**Discoverable documentation.** Tool descriptions should explain not just *what* the tool does, but *when* to use it and *when not* to use it. "Use `search_codebase` to find patterns across multiple files. Prefer this over `read_file` when you don't know which file contains the relevant code. Do not use this for searching documentation — use `search_docs` instead."

**Sensible defaults.** If a tool has optional parameters, the defaults should produce the most commonly desired behavior. A file search tool should default to searching from the repository root, not the current directory. A test runner should default to running all tests, not a specific test file. The agent should not need to specify the default case explicitly.

**Minimal surprise.** The **Principle of Least Astonishment** applies to agents just as it applies to human users. A tool should behave the way its name and description suggest. If a tool called `delete_file` actually moves the file to a trash directory, that is a violation — and the agent will be confused, just as a human developer would be confused by an API that does not do what its name says.

**Appropriate granularity.** Tools that are too low-level force the agent to make many small decisions, increasing the chance of error. Tools that are too high-level hide important details and remove the agent's ability to make fine-grained choices. The right granularity depends on the task: for code editing, a tool that replaces a specific string in a file is better than one that rewrites the entire file (less risk of unintended changes) but worse than one that requires the agent to specify byte offsets (too low-level, too error-prone).

---

## When Frameworks Help and When They Hurt

The agentic systems ecosystem is rich with frameworks: LangChain, LangGraph, CrewAI, AutoGen, Semantic Kernel, and many others. These frameworks promise to simplify the construction of agentic systems by providing pre-built abstractions for common patterns — tool management, memory, orchestration, conversation history, agent loops, and more.

Frameworks can help. They can also hurt. Understanding which is which is an essential harness engineering skill.

### When Frameworks Help

**Prototyping and exploration.** When you are exploring a new pattern — trying out orchestrator-workers, experimenting with evaluator-optimizer loops, testing different tool configurations — a framework lets you get something working in hours rather than days. The speed of iteration matters when you are still figuring out the right approach.

**Standard patterns with standard requirements.** When your use case aligns closely with a pattern the framework was designed for, and you do not need to customize the behavior significantly. If the framework has a well-tested implementation of prompt chaining and that is exactly what you need, using it saves engineering effort and gives you the benefit of the framework team's testing and bug fixing.

**Ecosystem integration.** When the framework provides connectors to tools and services you need — vector databases, cloud APIs, monitoring systems, message queues — that would be tedious to build from scratch. The value here is not in the abstraction but in the integration work that someone else has already done.

### When Frameworks Hurt

**Abstraction opacity.** When the framework's abstractions hide the underlying behavior in ways that make debugging difficult. If you cannot explain what happens when you call `agent.run(task)` — which LLM calls are made, in what order, with what prompts, with what tools — you will not be able to diagnose failures. And you *will* have failures. The question is whether you can diagnose them.

This is the **Law of Leaky Abstractions** (coined by Joel Spolsky) applied to agentic systems: all non-trivial abstractions leak, and when they leak, you need to understand the layer below. If you do not understand what the framework is doing under the hood, you are helpless when things go wrong.

**Premature generality.** When the framework provides abstractions that are more general than your use case requires. A framework designed to support arbitrary multi-agent orchestration with pluggable memory systems, dynamic tool registration, and customizable conversation management carries complexity and performance overhead for capabilities you may never use. The overhead is not just runtime — it is cognitive. Every abstraction you do not need is an abstraction you must understand, work around, and maintain.

**Version coupling.** When upgrading the framework requires non-trivial changes to your code, and the framework's release cycle does not align with your needs. Frameworks in this space are evolving rapidly — breaking changes are common, deprecation cycles are short, and the "recommended approach" changes every few months. If your production harness depends on framework internals, you are on a treadmill.

**Customization friction.** When you need to modify the framework's behavior in ways it was not designed to support. You need the agent loop to retry on a specific error type. You need the tool calling format to include additional metadata. You need the memory system to use your existing database instead of the framework's built-in one. Each customization is a friction point — a place where you are fighting the framework's opinions rather than benefiting from them.

### The Decision Rule

**Ensure you understand the underlying code.** This is the single most important principle when deciding whether to use a framework.

If you use a framework without understanding what it does under the hood, you are building on a foundation you cannot inspect. When something goes wrong — and in agentic systems, things go wrong in novel and confusing ways — you will be debugging a system you do not understand, using error messages that may be misleading, in a codebase that is not yours.

This does not mean you should never use frameworks. It means you should be able to, at any point, **drop down to the raw LLM calls** and understand exactly what is happening. The framework is a convenience layer, not a replacement for understanding.

For production harnesses, consider this spectrum:

1. **Build from scratch.** Use raw LLM APIs. Maximum control, maximum understanding, maximum engineering effort. Best for core business logic where reliability is paramount.
2. **Use a thin wrapper.** A minimal library that handles message formatting, tool calling conventions, streaming, and retry logic — but does not impose architectural patterns or orchestration flows. Libraries like Anthropic's SDK or OpenAI's SDK are in this category.
3. **Use a framework selectively.** Adopt the framework for the specific patterns it handles well, but understand every abstraction you use and be prepared to replace them. Do not use the framework's memory system if you do not understand how it stores and retrieves memories.
4. **Use a framework wholesale.** Accept the framework's architecture and conventions entirely. Appropriate for prototypes, internal tools, and non-critical applications — risky for production systems where reliability and debuggability matter.

The further you move toward production, the further you should move toward options 1 and 2. The further you move toward exploration and prototyping, the more options 3 and 4 make sense.

A useful heuristic: **if you could not reimplement the framework's behavior in a weekend, you do not understand it well enough to depend on it in production.** This is not about actually reimplementing it — it is about having the depth of understanding that would make reimplementation feasible.

---

## The Spectrum Revisited: Choosing the Right Pattern

Let us now place all of these patterns on the spectrum from simplest to most complex, and provide a practical decision framework for choosing among them:

```
Simplest                                                         Most Complex
   |                                                                   |
   v                                                                   v
Single      Prompt       Routing     Parallel-    Orchestrator-   Evaluator-    Full
 LLM        Chain                    ization      Workers        Optimizer     Agent
 Call
```

Each step to the right adds capability and flexibility — but also adds complexity, cost, unpredictability, and failure modes. The harness engineer's job is to find the **leftmost point on this spectrum that solves the problem**. Not the most impressive. Not the most "agentic." The simplest that works.

### Decision Framework

When evaluating which pattern to use, ask these questions in order:

**1. Can this be done without an LLM at all?** If the task can be accomplished with a regex, a database query, a script, or a traditional program — do that. LLMs are expensive, slow, and non-deterministic. Use them for tasks that require natural language understanding, reasoning about code semantics, or generating novel content. Do not use them for tasks that are deterministic and well-specified.

**2. Can a single LLM call handle it?** If the task fits in one prompt and the LLM can produce a reliable result in one shot, a single call is the right pattern. Write a good prompt. Add a few examples. Check the output programmatically. Move on.

**3. Can a fixed sequence of LLM calls handle it?** If the task decomposes into known sequential steps, use prompt chaining. Define the steps, write the gates, and enjoy the predictability.

**4. Does the input determine the processing strategy?** If different inputs need different handling, use routing. Build a classifier, define the handlers, implement a fallback path.

**5. Are there independent subtasks that can run in parallel?** If so, use sectioning. If you need reliability on a critical decision, use voting.

**6. Does the decomposition depend on the input?** If you cannot know the steps in advance, use orchestrator-workers. Invest in a good planning prompt for the orchestrator.

**7. Does the output need iterative refinement?** If first-pass quality is insufficient and you can define meaningful evaluation criteria, use evaluator-optimizer.

**8. Does the task require open-ended exploration?** If the agent needs to discover the approach at runtime, make decisions based on intermediate results, and recover from errors, use a full agent. Invest heavily in guardrails, logging, and cost controls.

At each level, the question is not "Could a more complex pattern work?" — it always could. The question is "Does the simpler pattern demonstrably fail in ways that the more complex pattern addresses?" If the answer is no, stay simple.

### Combining Patterns

These patterns are not mutually exclusive. Production systems commonly combine them, and the combinations can be powerful:

- **Routing + Prompt Chaining:** Route complex inputs to a detailed prompt chain and simple inputs to a single LLM call. This is the tiered-complexity approach.
- **Orchestrator-Workers + Evaluator-Optimizer:** The orchestrator dispatches workers, and each worker uses an evaluator-optimizer loop to refine its output before returning to the orchestrator.
- **Routing + Full Agent:** Route most inputs to efficient workflows, but route genuinely novel or complex inputs to a full agent. The agent handles the long tail of cases that the workflows cannot.
- **Prompt Chaining + Parallelization:** A prompt chain where one step fans out into parallel subtasks and the next step aggregates the results.

The combinatorial space is large, but the principle remains constant: **each pattern should earn its place by solving a specific problem that a simpler pattern cannot.** When you add a pattern to your architecture, you should be able to articulate: "We added evaluator-optimizer to the code generation worker because first-pass code quality was only 73% acceptable, and the eval-opt loop brings it to 95%."

| If your task has...                    | Consider...                    |
|----------------------------------------|--------------------------------|
| A deterministic solution               | No LLM at all                  |
| A single, well-defined step            | Single LLM call                |
| A fixed sequence of steps              | Prompt chaining                |
| Distinct input categories              | Routing                        |
| Independent subtasks                   | Parallelization (sectioning)   |
| Need for reliability on key decisions  | Parallelization (voting)       |
| Input-dependent decomposition          | Orchestrator-workers           |
| Clear quality criteria for refinement  | Evaluator-optimizer            |
| Open-ended goals requiring exploration | Full agent                     |

---

## Summary

This chapter has established the conceptual foundation for everything that follows. Here are the key ideas to carry forward:

1. **Agents and workflows are fundamentally distinct.** Workflows use predefined control flow determined by the developer at design time. Agents use dynamic, LLM-directed control flow determined at runtime. The distinction has concrete implications for harness design: workflows need fewer guardrails but less flexibility; agents need more guardrails but offer more capability.

2. **The augmented LLM** — an LLM enhanced with retrieval, tools, and memory — is the basic building block of all agentic systems. Every pattern in this chapter is a composition of augmented LLMs connected in different topologies.

3. **Five workflow patterns** cover the vast majority of structured use cases:
   - **Prompt chaining** for sequential decomposition with quality gates.
   - **Routing** for directing inputs to specialized handlers.
   - **Parallelization** (sectioning and voting) for speed and reliability.
   - **Orchestrator-workers** for input-dependent task decomposition.
   - **Evaluator-optimizer** for iterative refinement with feedback.

4. **Full agents** — LLMs operating in an observe-think-act loop with dynamic tool use — are the most powerful and the most dangerous pattern. They handle open-ended tasks that simpler patterns cannot, but they bring unpredictable cost, compounding errors, debugging difficulty, and testing challenges. Use them only when the task genuinely requires dynamic decision-making.

5. **Three principles** guide all agentic system design:
   - **Simplicity**: Use the simplest pattern that works. Add complexity only with evidence.
   - **Transparency**: Log everything, make reasoning visible, instrument for metrics.
   - **Carefully crafted ACI**: Invest in tool design as seriously as API design.

6. **Frameworks are tools, not foundations.** Use them when they accelerate development, understand what they do under the hood, and be prepared to replace them when they get in the way. The rule: if you cannot explain what happens when you call `agent.run()`, you do not understand the system well enough to debug it.

7. **The spectrum from simple to complex is a decision framework**, not a maturity model. Being at the "simple" end is not a sign of immaturity — it is a sign of disciplined engineering. The goal is the leftmost point on the spectrum that solves the problem.

In the next chapter, we will examine the agent loop in detail — the inner mechanics of how an agent observes, reasons, and acts. If this chapter told you *what* the patterns are, Chapter 2 will tell you *how they actually execute*.

---

## Annotated Bibliography

**[1]** Anthropic. "Building Effective Agents." *Anthropic Engineering Blog*, December 19, 2024. https://www.anthropic.com/engineering/building-effective-agents

> The primary source for this chapter. Defines the agents-vs-workflows distinction, catalogs all five workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer), and articulates the three core principles of simplicity, transparency, and ACI design. The appendices on agents in practice and prompt engineering tools are essential companion material.

**[2]** Anthropic. "Building Agents with the Claude Agent SDK." *Anthropic Blog*, September 29, 2025. https://claude.com/blog/building-agents-with-the-claude-agent-sdk

> Extends the building blocks discussion with the "give agents a computer" design principle. Covers the agent loop (gather context, take action, verify work, repeat) that implements the full agent pattern described in this chapter. Provides concrete guidance on subagent orchestration, compaction, and the tool categories that make up the augmented LLM.

**[3]** Anthropic. "Writing Effective Tools for AI Agents." *Anthropic Engineering Blog*, September 11, 2025. https://www.anthropic.com/engineering/writing-tools-for-agents

> Directly relevant to the ACI design principle discussed in this chapter. Covers tool consolidation (fewer, task-oriented tools over many fine-grained ones), tool description prompt engineering, and the evaluation-driven approach to tool development. Provides the practical implementation guidance for the ACI principles outlined here.

**[4]** Lopopolo, Ryan. "Harness Engineering: Leveraging Codex in an Agent-First World." *OpenAI*, February 2026. https://openai.com/index/harness-engineering/

> Demonstrates the full agent pattern operating at production scale. The OpenAI Harness team's experience validates this chapter's guidance on when full agents are appropriate: their Codex agents routinely operated in 6-hour autonomous sessions, using the observe-think-act loop with environmental feedback to implement features end-to-end.

**[5]** Anthropic. "Prompting Best Practices — Claude 4.6." *Claude Documentation*, 2025. https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices

> Covers model-specific guidance for implementing the patterns described in this chapter, including adaptive thinking, parallel tool calling, subagent orchestration, and the anti-patterns (overthinking, overengineering, test-gaming) that harness engineers must watch for when deploying agentic systems.
