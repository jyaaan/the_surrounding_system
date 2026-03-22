# Chapter 1: Foundations — Agentic Systems

> "The purpose of abstraction is not to be vague, but to create a new semantic level in which one can be absolutely precise."
> — Edsger W. Dijkstra

Before you can design a harness, you must understand the thing being harnessed. This chapter provides a rigorous foundation in agentic systems: what they are, how they are structured, and the canonical patterns for composing them. We will be precise about terminology, because imprecise terminology leads to imprecise thinking, which leads to poorly designed harnesses.

By the end of this chapter, you will have a taxonomy of agentic patterns that ranges from simple prompt-response interactions to fully autonomous agents — and, critically, you will understand *when to use which*.

---

## The Spectrum of Autonomy

It is tempting to think of agentic systems in binary terms: either you have an agent or you do not. This is wrong. What you actually have is a **spectrum of autonomy**, and your job as a harness engineer is to choose the right point on that spectrum for each task.

At one end: a single LLM call with a well-crafted prompt. No tools. No loops. No branching. The LLM receives input, produces output, and you are done.

At the other end: a fully autonomous agent that receives a high-level goal, decomposes it into subtasks, selects and uses tools, evaluates its own progress, recovers from errors, and iterates until the goal is achieved — potentially over hours or days of continuous operation.

Between these extremes lies a rich landscape of **workflow patterns** — systems where LLMs are composed with tools, other LLMs, and control flow logic in structured ways. Understanding this landscape is the prerequisite for everything else in this book.

The key insight, which we will return to throughout this chapter, is Anthropic's distinction:

- **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**. The control flow is determined by the developer at design time.
- **Agents** are systems where LLMs **dynamically direct their own processes and tool usage**. The control flow is determined by the LLM at runtime.

This is not a value judgment. Workflows are not inferior to agents. In many cases, workflows are the *correct* choice — they are more predictable, more debuggable, more efficient, and more reliable. The agent pattern is warranted only when the task genuinely requires dynamic decision-making that cannot be anticipated at design time.

**Start simple. Add complexity only when it demonstrably improves outcomes.** This principle will save you from the most common failure mode in agentic systems: over-engineering.

---

## The Augmented LLM: The Basic Building Block

Before we examine workflows and agents, we need to understand the fundamental unit from which they are all constructed: the **augmented LLM**.

A raw LLM is a function from text to text. It takes a prompt and returns a completion. This is useful but limited. An augmented LLM extends this basic capability with three types of external integration:

### 1. Retrieval

The LLM can pull in information from external sources at inference time. This is the family of techniques commonly grouped under **Retrieval-Augmented Generation (RAG)**:

- **Vector search** over a document corpus, codebase, or knowledge base.
- **Keyword search** with BM25 or similar sparse retrieval methods.
- **Structured queries** against databases or APIs.
- **File system access** — reading specific files from a codebase.

Retrieval addresses the fundamental limitation of context windows: the LLM cannot hold everything it needs to know in its prompt. Retrieval allows it to *look things up* at the moment it needs them.

For harness engineering, retrieval is how you make documentation, architecture decisions, and coding conventions available to the agent *at the point of need*. Rather than stuffing everything into a massive system prompt (which wastes tokens and dilutes attention), you design retrieval mechanisms that surface the right information for the task at hand.

### 2. Tools

The LLM can take actions in the world by invoking **tools** — functions that the LLM calls by emitting structured outputs (typically JSON) that are then executed by the runtime environment. Common tools include:

- **Code execution**: running code in a sandbox.
- **File operations**: reading, writing, and modifying files.
- **Shell commands**: executing arbitrary commands in a terminal.
- **API calls**: interacting with external services.
- **Search**: querying search engines, code search indices, or documentation.
- **Browser automation**: navigating web pages, filling forms, extracting content.

Tools are how the agent *acts*. Without tools, an LLM can only produce text. With tools, it can modify codebases, run tests, deploy services, query databases, and interact with the full surface area of a modern software system.

The design of tools — their names, their parameter schemas, their documentation, their error messages — is a critical harness engineering concern. We dedicate Chapter 2 to this topic under the name **Agent-Computer Interface (ACI) design**.

### 3. Memory

The LLM can persist information across interactions. Memory comes in several forms:

- **Conversational memory**: the message history within a single session. This is the most basic form and is limited by context window size.
- **Short-term working memory**: scratchpads, intermediate results, and state that persists within a task but not across tasks.
- **Long-term memory**: information that persists across sessions — learned preferences, facts about the codebase, historical decisions. This is typically implemented through external storage (databases, files, vector stores) combined with retrieval.

Memory is what transforms a stateless text-completion engine into something that can maintain coherence across a complex, multi-step task. It is also one of the hardest things to get right, because the LLM must decide what to remember, what to forget, and how to organize what it retains.

### The Augmented LLM as a Node

The mental model: think of each augmented LLM as a **node in a graph**. Each node has:

- **Inputs**: the prompt, retrieved context, and any state passed from upstream nodes.
- **Processing**: the LLM's reasoning, which may include multiple internal steps (chain-of-thought, reflection, etc.).
- **Tool calls**: zero or more invocations of external tools, whose results feed back into the LLM's reasoning.
- **Outputs**: the final response, which may be text, structured data, tool calls, or a combination.

Workflow patterns and agent architectures are different ways of connecting these nodes. The patterns differ in *who controls the connections* — the developer (workflows) or the LLM (agents).

---

## Workflow Pattern 1: Prompt Chaining

**Prompt chaining** decomposes a task into a sequence of steps, where each step is a separate LLM call and the output of one step becomes the input to the next. Between steps, you can insert programmatic **gates** — validation checks that determine whether the chain should continue, retry, or abort.

### The Pattern

```
Input -> [LLM Step 1] -> Gate -> [LLM Step 2] -> Gate -> [LLM Step 3] -> Output
```

Each step has a focused, well-defined subtask. The gates between steps enforce quality constraints before passing results downstream.

### When to Use It

Prompt chaining is ideal when:

- The task can be cleanly decomposed into sequential subtasks.
- Each subtask is simple enough for a single LLM call to handle reliably.
- You want to validate intermediate results before proceeding.
- You need a clear audit trail of how the final output was produced.

### Example: Automated Code Review

Consider automating a structured code review. The task is too complex for a single prompt, but it decomposes naturally:

**Step 1: Summarize the changes.** The LLM receives the diff and produces a structured summary: which files were changed, what type of change each represents (new feature, bug fix, refactor), and which subsystems are affected.

**Gate 1:** Validate that the summary is structurally complete. Does it reference all changed files? Is the change-type classification from a permitted set of values? If not, retry step 1 with a corrective prompt.

**Step 2: Analyze each change for issues.** The LLM receives the diff plus the summary from step 1, and for each file, identifies potential issues: bugs, style violations, performance concerns, security risks, missing tests.

**Gate 2:** Check that the analysis covers all files listed in the summary. Verify that each issue has a severity rating and a specific line reference. Filter out low-confidence findings.

**Step 3: Generate the review.** The LLM receives the original diff, the summary, and the filtered analysis, and produces a cohesive review comment with actionable feedback, organized by severity.

### The Tradeoff

Prompt chaining trades **latency for reliability**. Each step adds an LLM call (and thus latency and cost), but the decomposition makes each individual call simpler and more likely to succeed. The gates catch errors before they propagate, which is far cheaper than debugging an error in the final output and trying to figure out which step went wrong.

The key design decision is **granularity**: how fine should the steps be? Too fine, and you incur unnecessary overhead and lose the LLM's ability to reason holistically about the task. Too coarse, and you lose the benefits of decomposition.

A useful heuristic: **each step should be something you could evaluate independently.** If you cannot write a clear gate condition between two adjacent steps, they probably should be a single step.

---

## Workflow Pattern 2: Routing

**Routing** classifies the input and directs it to a specialized handler — a different prompt, a different model, a different pipeline, or even a non-LLM code path.

### The Pattern

```
                          -> [Handler A] ->
Input -> [Classifier] -  -> [Handler B] ->  -> Output
                          -> [Handler C] ->
```

The classifier examines the input and determines which handler is appropriate. Each handler is optimized for its specific category of input.

### When to Use It

Routing is ideal when:

- Your inputs fall into distinct categories that benefit from different processing.
- You want to use cheaper or faster models for simple cases and reserve expensive models for complex ones.
- Different categories require different tools, context, or expertise.
- You need to separate concerns cleanly for maintainability.

### Example: Multi-Tier Support System

Consider an internal developer support system that handles requests from engineers:

**Classifier:** An LLM (or even a traditional ML model — routing classifiers do not need to be LLMs) examines the incoming request and categorizes it:

- **Documentation query**: the developer is looking for existing information. Route to a RAG pipeline that searches the documentation corpus and generates an answer.
- **Bug report**: the developer has found a bug. Route to a diagnostic agent that can query logs, examine recent deployments, and correlate symptoms with known issues.
- **Feature request**: the developer wants something new. Route to a triage handler that formats the request, checks for duplicates, estimates effort, and creates a ticket.
- **Infrastructure issue**: something is down or degraded. Route to a runbook executor that follows predefined incident response procedures.

Each handler is purpose-built for its category. The documentation query handler has access to the vector store; the bug report handler has access to logging infrastructure; the infrastructure handler has access to deployment tools. None of them need *all* of these capabilities — routing ensures each handler has exactly the tools and context it needs.

### The Tradeoff

Routing trades **generality for specialization**. A single general-purpose agent could handle all of these cases, but it would need access to all tools, all context, and all instructions simultaneously — which means a larger prompt, more potential for confusion, and higher cost per request.

Routing is an application of the **Single Responsibility Principle** to agentic systems. Each handler does one thing well.

The risk is misclassification. If the classifier routes a request to the wrong handler, the output will be poor — and the failure mode is confusing, because the handler will confidently produce an answer to the wrong question. Robust routing requires:

- A high-quality classifier with clear category definitions.
- A fallback path for ambiguous inputs (e.g., route to a general-purpose handler, or ask the user for clarification).
- Monitoring that tracks classification accuracy over time.

---

## Workflow Pattern 3: Parallelization

**Parallelization** runs multiple LLM calls simultaneously and aggregates their results. There are two sub-patterns:

### Sub-pattern A: Sectioning

**Sectioning** divides a task into independent subtasks that can be processed in parallel, then combines the results.

```
            -> [Subtask A] ->
Input  --   -> [Subtask B] ->   -- [Aggregator] -> Output
            -> [Subtask C] ->
```

**Example: Comprehensive Code Analysis**

You need to analyze a pull request from multiple angles: security, performance, correctness, and style. These analyses are independent — the security analysis does not depend on the performance analysis. So you run them in parallel:

- **Branch A**: Security review. The LLM, equipped with a security-focused system prompt and OWASP reference material, analyzes the diff for vulnerabilities.
- **Branch B**: Performance review. The LLM, equipped with a performance-focused system prompt and profiling data, analyzes the diff for performance regressions.
- **Branch C**: Correctness review. The LLM, equipped with the test suite and specification, analyzes the diff for logical errors.
- **Branch D**: Style review. The LLM, equipped with the style guide, checks for convention violations.

The aggregator combines the four analyses into a unified review, deduplicating any overlapping findings and organizing by severity.

Sectioning gives you **linear speedup** for tasks with independent subtasks. A four-way parallel analysis takes roughly the same wall-clock time as a single analysis, but produces four times the coverage.

### Sub-pattern B: Voting

**Voting** runs the *same* task multiple times (potentially with different prompts, models, or temperatures) and aggregates the results through a voting mechanism.

```
            -> [Attempt 1] ->
Input  --   -> [Attempt 2] ->   -- [Voter] -> Output
            -> [Attempt 3] ->
```

**Example: Reliable Classification**

You need to classify whether a code change is a breaking change. This is a high-stakes classification — a false negative means users hit an unexpected breaking change; a false positive means unnecessary work. You run three independent classifications:

- **Attempt 1**: Standard prompt, temperature 0.
- **Attempt 2**: Chain-of-thought prompt that forces explicit reasoning, temperature 0.
- **Attempt 3**: Different model (perhaps a smaller, faster model that has been fine-tuned on breaking-change classification).

The voter examines the three outputs. If all three agree, the classification is accepted with high confidence. If two out of three agree, the majority wins but the confidence is flagged as moderate. If all three disagree, the classification is escalated to a human reviewer.

Voting is an application of **ensemble methods** to LLM outputs. It increases reliability at the cost of increased latency and compute. Use it when the cost of an incorrect output is high enough to justify the multiplied inference cost.

### The Tradeoff

Parallelization trades **compute cost for either speed (sectioning) or reliability (voting)**. It is most valuable when:

- Individual LLM calls are the bottleneck (sectioning).
- Individual LLM calls are unreliable for the task (voting).
- The cost of additional LLM calls is small relative to the cost of a wrong answer.

---

## Workflow Pattern 4: Orchestrator-Workers

The **orchestrator-worker** pattern uses a central LLM (the orchestrator) to dynamically break down a task and delegate subtasks to worker LLMs. Unlike prompt chaining, the decomposition is not fixed at design time — the orchestrator decides the decomposition at runtime based on the specific input.

### The Pattern

```
                                -> [Worker A] ->
Input -> [Orchestrator]  --     -> [Worker B] ->    -- [Orchestrator] -> Output
                                -> [Worker C] ->
```

The orchestrator receives the task, analyzes it, determines what subtasks are needed, dispatches them to workers, and synthesizes the results.

### When to Use It

Orchestrator-workers is ideal when:

- The decomposition of the task depends on the input and cannot be determined in advance.
- The number and type of subtasks varies across inputs.
- The subtasks may need to be sequenced dynamically (e.g., the output of one worker informs which workers to invoke next).

### Example: Multi-File Code Generation

A user requests: "Add a new API endpoint for user preferences, including database migration, model, controller, route, tests, and documentation."

A prompt chain could handle this, but the exact set of files to create and modify depends on the existing codebase structure — which framework is being used, where models live, how routes are organized, what the testing conventions are. The orchestrator handles this dynamically:

**Orchestrator (Step 1 — Analysis):** The orchestrator examines the codebase structure (using file listing and code search tools) and determines the plan:

- Create a migration file at `db/migrations/20260315_add_user_preferences.sql`.
- Create a model at `src/models/user_preference.go`.
- Add a controller at `src/controllers/preferences_controller.go`.
- Register routes in `src/routes/api.go`.
- Create tests at `tests/controllers/preferences_controller_test.go`.
- Update API documentation at `docs/api/preferences.md`.

**Workers (Step 2 — Execution):** The orchestrator dispatches each file creation/modification to a worker LLM. Each worker receives:

- The specific file to create or modify.
- Relevant context: existing code patterns from similar files, the database schema, the API specification.
- Constraints: coding standards, naming conventions, error handling patterns.

Workers execute in parallel where possible (the model and controller can be written in parallel; the route registration depends on the controller being defined).

**Orchestrator (Step 3 — Synthesis):** The orchestrator reviews the workers' outputs for consistency. Does the controller reference the model correctly? Do the routes match the controller methods? Do the tests cover the specified behavior? If inconsistencies are found, the orchestrator dispatches correction tasks to the relevant workers.

### The Tradeoff

Orchestrator-workers trades **predictability for flexibility**. The dynamic decomposition handles novel inputs well, but it means the exact execution path is not known in advance — which makes debugging harder and costs less predictable.

The orchestrator is a **single point of failure** for planning quality. If the orchestrator produces a poor decomposition, all downstream work is wasted. This is why the orchestrator typically needs to be a more capable (and more expensive) model than the workers. It is also why the orchestrator's prompt requires careful engineering: it needs to understand the problem domain deeply enough to produce good plans.

This pattern is closely related to the **Map-Reduce** pattern in distributed computing and to the **Facade** pattern in object-oriented design — a single entry point that coordinates complex internal operations.

---

## Workflow Pattern 5: Evaluator-Optimizer

The **evaluator-optimizer** pattern creates a feedback loop between two LLMs: one that generates output and one that evaluates it. The generator iterates on its output based on the evaluator's feedback until the evaluator is satisfied (or a maximum iteration count is reached).

### The Pattern

```
Input -> [Generator] -> Output -> [Evaluator] -> Feedback
              ^                                      |
              |______________________________________|
              (loop until accepted or max iterations)
```

### When to Use It

Evaluator-optimizer is ideal when:

- The quality of the output can be meaningfully evaluated by an LLM.
- There is a clear notion of "good enough" that can be expressed as evaluation criteria.
- Iterative refinement is likely to improve the output (i.e., the task is one where feedback is actionable).
- First-pass quality is insufficient for your needs.

### Example: Code Generation with Quality Gates

The task is to generate a function that parses a complex log format.

**Generator (Iteration 1):** Produces an initial implementation based on the specification and example log entries.

**Evaluator (Iteration 1):** Reviews the implementation against multiple criteria:
- *Correctness*: Does it handle all the example log entries correctly? (The evaluator can actually run the code against examples if given a code execution tool.)
- *Edge cases*: Does it handle malformed input, empty lines, multiline entries?
- *Performance*: Is the approach efficient for large log files, or is it doing something quadratic?
- *Style*: Does it follow the codebase's conventions?

The evaluator returns structured feedback: "The implementation fails on multiline log entries. It assumes each log entry is a single line, but the specification shows that stack traces produce multi-line entries. The parser needs to accumulate lines until it sees the next timestamp prefix."

**Generator (Iteration 2):** Receives its original output plus the evaluator's feedback. Produces a revised implementation that handles multiline entries.

**Evaluator (Iteration 2):** Confirms the multiline handling is correct, but notes: "The regex for timestamp parsing is too permissive — it would match timestamps in the middle of log messages, not just at the start of lines. Use an anchor."

**Generator (Iteration 3):** Fixes the regex. The evaluator accepts the output.

### The Tradeoff

Evaluator-optimizer trades **compute cost and latency for output quality**. Each iteration doubles the LLM calls (one for generation, one for evaluation), and the number of iterations is unpredictable.

The critical design decision is the **evaluation criteria**. If the criteria are too vague ("make it better"), the evaluator will produce unhelpful feedback and the loop will oscillate without converging. If the criteria are too strict, the loop will never terminate. The sweet spot is criteria that are *specific enough to be actionable* and *permissive enough to allow multiple valid solutions*.

This pattern is related to **Generative Adversarial Networks (GANs)** in the machine learning literature — though the analogy is imperfect, because the evaluator is not adversarial but constructive. A better analogy is **code review as a loop**: the generator submits a PR, the reviewer provides feedback, the generator revises, and this continues until the reviewer approves.

The evaluator-optimizer pattern is also deeply relevant to harness engineering because *the harness itself functions as an evaluator*. When an agent runs code and the tests fail, the test failures are evaluation feedback. When a linter flags issues, those are evaluation signals. The difference is that in the evaluator-optimizer pattern, the evaluation is performed by an LLM, which can provide richer, more contextual feedback than a static tool — but at higher cost and lower reliability.

---

## Full Agents: Dynamic Tool Use in a Loop

We now arrive at the far end of the autonomy spectrum: **full agents**. An agent is an LLM that uses tools based on environmental feedback in a loop, with the LLM itself determining the control flow.

### The Pattern

```
Input -> [LLM decides action] -> [Execute action] -> [Observe result]
              ^                                            |
              |____________________________________________|
              (loop until goal is achieved or abandoned)
```

This is the **observe-think-act loop** (sometimes called the **ReAct pattern**, after the seminal paper by Yao et al.). At each iteration:

1. The agent **observes** the current state: tool outputs, error messages, file contents, test results.
2. The agent **thinks** about what to do next: reasons about the goal, the current state, and the available actions.
3. The agent **acts** by selecting and invoking a tool, or by producing a final output.

The key distinction from workflows is that **steps 1-3 are not predefined**. The agent decides at each iteration which tool to use, what arguments to pass, and whether to continue or stop. The control flow is emergent, not prescribed.

### When to Use Full Agents

Full agents are appropriate when:

- The task requires **exploration**: the agent does not know in advance which tools it will need or how many steps will be required.
- The task involves **error recovery**: the agent must be able to detect failures, diagnose root causes, and try alternative approaches.
- The task has **open-ended goals**: "fix this bug" rather than "apply this specific patch."
- The **environment is complex and varied**: different inputs may require fundamentally different approaches.

### Example: Autonomous Bug Fixing

The agent receives a bug report: "The API returns a 500 error when the user's display name contains Unicode characters."

**Iteration 1 — Understand:** The agent reads the bug report, identifies the relevant API endpoint, and examines the endpoint's handler code.

**Iteration 2 — Reproduce:** The agent writes a test case that calls the endpoint with a Unicode display name and confirms it produces a 500 error.

**Iteration 3 — Diagnose:** The agent examines the error logs and finds a stack trace pointing to a string encoding function. It reads the function and identifies the issue: the function assumes ASCII input and throws on non-ASCII characters.

**Iteration 4 — Fix:** The agent modifies the function to handle Unicode correctly.

**Iteration 5 — Verify:** The agent runs the new test case. It passes. The agent runs the full test suite to check for regressions. Two unrelated tests fail.

**Iteration 6 — Investigate regressions:** The agent examines the failing tests. One is a flaky test (it fails intermittently regardless of code changes). The other is testing the old (incorrect) behavior — it expects the function to throw on non-ASCII input. The agent updates this test to expect correct Unicode handling.

**Iteration 7 — Final verification:** Full test suite passes. The agent creates a pull request with a description explaining the bug, the fix, and the test changes.

Notice what happened: the agent encountered an unexpected complication (regression tests) and adapted its plan dynamically. A workflow could not have anticipated this — the number and type of iterations depended on the specific bug and the specific state of the codebase.

### The Tradeoffs of Full Agents

Full agents offer maximum flexibility but come with significant costs:

**Unpredictable cost and latency.** You do not know in advance how many iterations the agent will take. A simple bug might be fixed in three iterations; a complex one might take thirty. Cost and time are proportional to iterations, and there is no reliable way to estimate them a priori.

**Compounding errors.** Each iteration builds on the previous ones. If the agent makes a mistake in iteration 3, iterations 4-7 may all be wasted — or worse, they may compound the error into something hard to diagnose. This is the **error accumulation problem**, and it is the primary risk of agentic systems.

**Difficult to debug.** When an agent produces incorrect output after fifteen iterations, figuring out *where it went wrong* requires reviewing the entire trace. This is why transparency — logging every observation, thought, and action — is essential.

**Hard to test.** Testing a workflow is relatively straightforward: you provide inputs and check outputs. Testing an agent requires testing its *decision-making process*, which is non-deterministic and context-dependent.

These tradeoffs are why the guidance is clear: **use the simplest pattern that solves the problem.** If a prompt chain works, do not use an orchestrator-worker. If an orchestrator-worker works, do not use a full agent. Reserve full agents for tasks that genuinely require dynamic decision-making.

---

## Three Core Principles

Across all of these patterns, three principles consistently distinguish well-designed agentic systems from poorly designed ones.

### Principle 1: Simplicity

The most reliable agentic system is the simplest one that solves the problem. Every additional layer of complexity — every additional LLM call, every additional tool, every additional decision point — is a potential failure mode.

This is not an abstract philosophical position. It has concrete engineering implications:

- **Prefer prompt chaining over orchestrator-workers** when the task decomposition is known in advance.
- **Prefer a single focused tool over ten general-purpose tools** — each unused tool is noise in the agent's decision space.
- **Prefer explicit gates over evaluator-optimizer loops** when the quality criteria can be checked programmatically.
- **Prefer no agent at all** when a traditional program, a script, or a static pipeline can solve the problem.

The temptation to over-engineer is strong, especially when the technology is exciting. Resist it. The OpenAI experiment succeeded not because they used the most sophisticated agent architecture, but because they used the *right level of sophistication for each task* — and erred on the side of simplicity.

### Principle 2: Transparency

When an agentic system fails — and it will fail — you need to understand *why*. This requires transparency at every level:

- **Log everything.** Every LLM call (prompt and response), every tool invocation (arguments and results), every decision point.
- **Make reasoning visible.** Use chain-of-thought prompting so the agent's reasoning is externalized, not hidden. Reasoning tokens are the printf-debugging of agentic systems.
- **Provide human-readable traces.** The log should tell a story that a human can follow: "The agent read the file, noticed the function on line 42, decided to modify it, ran the tests, saw the failure, and tried a different approach."
- **Instrument for metrics.** Track iterations per task, tool usage patterns, error rates, backtracking frequency, and cost per task. These metrics tell you where your harness needs improvement.

Transparency is not just a debugging aid. It is a **trust mechanism**. When stakeholders can see *how* the agent arrived at its output, they can make informed decisions about whether to accept it. Opaque systems that produce correct output 95% of the time are less useful than transparent systems that produce correct output 90% of the time — because the transparent system tells you when to be skeptical.

### Principle 3: Carefully Crafted Agent-Computer Interface (ACI)

The **Agent-Computer Interface** is the set of tools, commands, and conventions through which the agent interacts with the world. ACI design is to agent systems what API design is to software systems — and it is at least as important.

The core insight: **the agent can only do what its tools allow it to do, and it can only do it well if the tools are well-designed.** A poorly named tool, a confusing parameter schema, an unhelpful error message — these are bugs in the harness, just as surely as a null pointer dereference is a bug in code.

Good ACI design follows familiar principles, applied in a new context:

- **Clear naming.** A tool called `search_codebase` is better than one called `search`, because the agent may have access to multiple search tools and needs to distinguish them. A tool called `create_file` is better than `write` when the action specifically creates a new file (as opposed to overwriting an existing one).
- **Focused functionality.** Each tool should do one thing. A tool that searches the codebase and also formats the results is trying to do two things. Separate them.
- **Rich error messages.** When a tool fails, the error message should tell the agent *what went wrong and how to fix it*. "File not found: `/src/models/user.go`" is better than "Error." "File not found: `/src/models/user.go`. Did you mean `/src/models/users.go`?" is better still.
- **Discoverable documentation.** Tool descriptions should explain not just *what* the tool does, but *when* to use it. "Use this tool to search for patterns across the codebase. Prefer this over reading individual files when you don't know which file contains the relevant code."
- **Sensible defaults.** If a tool has optional parameters, the defaults should produce the most commonly desired behavior. The agent should not need to specify the default case explicitly.
- **Minimal surprise.** A tool should behave the way its name and description suggest. If a tool called `delete_file` actually moves the file to a trash directory, that is a violation of the principle of least astonishment — and the agent will be as confused as a human would.

We will explore ACI design in much greater depth in Chapter 2. For now, the key takeaway is: **the quality of the agent's output is bounded by the quality of the interface it operates through.** Invest in ACI design as seriously as you invest in API design.

---

## When Frameworks Help and When They Hurt

The agentic systems ecosystem is rich with frameworks: LangChain, LangGraph, CrewAI, AutoGen, and many others. These frameworks promise to simplify the construction of agentic systems by providing pre-built abstractions for common patterns — tool management, memory, orchestration, conversation history, and more.

Frameworks can help. They can also hurt. Here is how to decide:

### When Frameworks Help

- **Prototyping.** When you are exploring a new pattern and want to get something working quickly. Frameworks let you stand up a prompt chain or an agent loop in minutes rather than hours.
- **Common patterns.** When your use case aligns closely with a pattern the framework was designed for. If the framework has a well-tested implementation of orchestrator-workers and that is what you need, using it saves engineering effort.
- **Ecosystem integration.** When the framework provides connectors to tools and services you need — vector databases, APIs, cloud services — that would be tedious to build from scratch.

### When Frameworks Hurt

- **Abstraction opacity.** When the framework's abstractions hide the underlying behavior in ways that make debugging difficult. If you cannot explain what happens when you call `agent.run(task)` — which LLM calls are made, in what order, with what prompts — you will not be able to diagnose failures.
- **Premature generality.** When the framework's abstractions are more general than your use case requires, introducing complexity and performance overhead for capabilities you do not use.
- **Version coupling.** When upgrading the framework requires non-trivial changes to your code, and the framework's release cycle does not align with your needs.
- **Customization friction.** When you need to modify the framework's behavior in ways it was not designed to support, leading to fragile workarounds and maintenance burden.

### The Rule

**Ensure you understand the underlying code.** This is the single most important principle when deciding whether to use a framework.

If you use a framework without understanding what it does under the hood, you are building on a foundation you cannot inspect. When something goes wrong — and it will — you will be debugging a system you do not understand, using error messages that may be misleading, in a codebase that is not yours.

This does not mean you should never use frameworks. It means you should be able to, at any point, drop down to the raw LLM calls and understand exactly what is happening. The framework is a convenience, not a replacement for understanding.

For production harnesses, consider this spectrum:

1. **Build from scratch.** Use raw LLM APIs. Maximum control, maximum understanding, maximum engineering effort.
2. **Use a thin wrapper.** A minimal library that handles message formatting, tool calling conventions, and retry logic — but does not impose architectural patterns.
3. **Use a framework selectively.** Adopt the framework for the specific patterns it handles well, but understand every abstraction you use and be prepared to replace them.
4. **Use a framework wholesale.** Accept the framework's architecture and conventions entirely. Appropriate for prototypes and internal tools; risky for production systems.

The further you move toward production, the further you should move toward options 1 and 2. The further you move toward exploration, the more options 3 and 4 make sense.

---

## The Spectrum Revisited

Let us now place all of these patterns on the spectrum, from simplest to most complex:

```
Simplest                                                    Most Complex
   |                                                              |
   v                                                              v
Single    Prompt     Routing    Parallel-    Orchestrator-   Evaluator-    Full
 LLM      Chain                 ization      Workers        Optimizer     Agent
 Call
```

Each step to the right adds capability and flexibility — but also adds complexity, cost, unpredictability, and failure modes.

**The harness engineer's job is to find the leftmost point on this spectrum that solves the problem.** Not the most impressive. Not the most "agentic." The simplest that works.

This is a specific instance of a more general engineering principle: the **YAGNI principle** (You Aren't Gonna Need It). Do not add orchestration until you have evidence that simpler patterns are insufficient. Do not add a full agent loop until you have evidence that the task genuinely requires dynamic decision-making.

Here is a decision framework:

| If your task has...                    | Consider...           |
|----------------------------------------|-----------------------|
| A fixed sequence of steps              | Prompt chaining       |
| Distinct input categories              | Routing               |
| Independent subtasks                   | Parallelization (sectioning) |
| Need for reliability on key decisions  | Parallelization (voting) |
| Input-dependent decomposition          | Orchestrator-workers  |
| Clear quality criteria for refinement  | Evaluator-optimizer   |
| Open-ended goals requiring exploration | Full agent            |

### Combining Patterns

These patterns are not mutually exclusive. Production systems commonly combine them. An orchestrator-worker system might use prompt chaining within each worker. A routing system might direct complex inputs to a full agent and simple inputs to a single LLM call. An evaluator-optimizer might be embedded as a step within a prompt chain.

The combinatorial space is large, but the principle remains constant: **each pattern should earn its place by solving a specific problem that a simpler pattern cannot.**

---

## Summary

This chapter has established the conceptual foundation for everything that follows. Here are the key ideas to carry forward:

1. **Agents and workflows are distinct.** Workflows use predefined control flow. Agents use dynamic, LLM-directed control flow. The distinction matters for harness design because workflows are predictable and agents are not.

2. **The augmented LLM** (retrieval + tools + memory) is the basic building block. Every pattern in this chapter is a composition of augmented LLMs.

3. **Five workflow patterns** cover most structured use cases: prompt chaining, routing, parallelization, orchestrator-workers, and evaluator-optimizer. Each has specific strengths, specific costs, and specific design decisions.

4. **Full agents** are the most powerful and the most dangerous pattern. They should be used only when the task genuinely requires dynamic decision-making.

5. **Three principles** guide all agentic system design: simplicity (use the simplest pattern that works), transparency (make everything observable), and carefully crafted ACI (invest in the interface the agent operates through).

6. **Frameworks are tools, not foundations.** Use them when they help, replace them when they hinder, and always understand what they do under the hood.

In the next chapter, we will dive deep into the Agent-Computer Interface — the critical layer between the agent and the world it operates in. If this chapter told you *what* agents are, Chapter 2 will tell you how to give them *hands*.
