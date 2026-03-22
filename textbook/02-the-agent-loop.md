# Chapter 2: The Agent Loop

Every agent system, regardless of its complexity, sophistication, or domain, runs on the same fundamental cycle:

**Gather Context -> Take Action -> Verify Work -> Repeat.**

This is the heartbeat. If you understand nothing else about agent engineering, understand this loop. It is the reason agents can accomplish tasks that would be impossible for a single-shot LLM call. It is the reason agents recover from mistakes. It is the mechanism through which emergent reliability arises from an unreliable substrate.

The loop is deceptively simple. A naive implementation takes twenty lines of code. But the difference between a toy agent and a production system lies entirely in *how well* each phase is executed — and how intelligently the transitions between phases are managed. This chapter dissects each phase, names the patterns that make them work, and explains the engineering tradeoffs you will face when building real systems.

---

## 2.1 Gather Context

An agent is only as good as the information it has when it makes a decision. The gather-context phase is where the agent builds its working model of the problem — reading files, searching codebases, querying APIs, inspecting error messages, and assembling the tokens that will inform its next action.

This is not a one-time operation. The agent gathers context *every iteration* of the loop. The first pass might be broad and exploratory; subsequent passes are surgical, guided by what the agent learned in previous iterations. This progressive narrowing is one of the most important dynamics in agent systems.

### Agentic Search vs. Semantic Search

There are two fundamentally different strategies for finding relevant information, and understanding their tradeoffs is essential.

**Semantic search** uses embedding models to convert queries and documents into vectors, then retrieves documents whose vectors are close to the query vector in embedding space. It is the backbone of most RAG (Retrieval-Augmented Generation) systems.

Semantic search is *fast*. A vector lookup in a well-indexed database returns results in milliseconds. It requires no LLM inference during retrieval. It scales to millions of documents. These are real advantages.

But semantic search has deep structural weaknesses:

1. **Lossy compression.** Embeddings collapse the full meaning of a document into a fixed-dimensional vector. Subtle distinctions — the difference between a function that *validates* input and one that *sanitizes* it — can be lost. The embedding model makes decisions about what matters before the reasoning model ever sees the content.

2. **Maintenance burden.** You need an embedding pipeline: chunking strategies, embedding model selection, index updates when documents change, re-embedding when you upgrade models. Every component is a potential failure point. Chunking strategy alone — how you split documents, how much overlap you use, whether you respect semantic boundaries — is a research problem unto itself.

3. **Opaque failures.** When semantic search returns the wrong results, debugging is extraordinarily difficult. You cannot inspect a 1536-dimensional vector and understand why document A ranked higher than document B. The failure mode is silent: the agent proceeds confidently with incomplete or misleading context.

**Agentic search** takes a different approach. The agent uses tools — file system operations, grep, glob patterns, code navigation — to actively explore the information space. It reads a file, notices an import, follows it to another file, reads a function signature, searches for callers of that function, and builds understanding incrementally.

Agentic search is *slower*. Each step requires an LLM inference call. Exploring a codebase might take dozens of tool calls where a vector lookup would take one. But agentic search has properties that semantic search lacks:

1. **Transparency.** Every step is visible. You can inspect the agent's trajectory and understand exactly why it found (or missed) a piece of information. When it fails, you can see where the search went wrong.

2. **Reasoning-guided retrieval.** The agent applies its full reasoning capability at each step. It can notice that a variable name suggests a certain architectural pattern, decide to look for configuration files, find an unexpected dependency, and adjust its search accordingly. The retrieval is *adaptive*.

3. **No infrastructure.** There is no embedding pipeline, no vector database, no chunking strategy. The tools are simple: read a file, search for a pattern, list a directory. The complexity lives in the model's reasoning, which improves automatically as models improve.

The right answer is almost always a hybrid. Use lightweight pre-computed indices (file path lists, symbol tables, dependency graphs) for broad orientation, and agentic exploration for deep understanding. Claude Code exemplifies this pattern: it loads `CLAUDE.md` upfront for high-level guidance, then uses `glob` and `grep` for just-in-time exploration of the actual codebase.

### The File System as Context

Engineers underestimate how much information is encoded in the file system itself, independent of file contents. An agent that pays attention to *structure* has a significant advantage.

**Directory hierarchy** reveals architectural boundaries. A `src/services/` directory tells the agent that the codebase separates concerns into services. A `tests/integration/` directory signals that integration tests exist and where to find them. A `migrations/` directory indicates a database with schema evolution. None of this requires reading a single file.

**Naming conventions** encode semantics. Files named `*_test.go` are Go tests. Files named `*.controller.ts` follow the controller pattern. A file called `schema.prisma` tells the agent the ORM technology, the database schema, and where to find model definitions — all from five characters in a filename.

**Timestamps** indicate recency and relevance. Recently modified files are more likely to be related to the current task. A file last touched three years ago in an otherwise active directory might be legacy code. Modification patterns across files can reveal which components are coupled — files that change together are likely related.

**File size** is a signal. A 5000-line file is likely doing too much and may be the source of complexity bugs. A 10-line configuration file is likely a high-leverage target for understanding system behavior.

The best agents treat `ls`, `find`, and `stat` as first-class intelligence-gathering tools, not just utilities.

### Just-in-Time Context vs. Pre-Inference Retrieval

There is a fundamental tension in agent design between loading context *before* the model reasons and loading it *during* reasoning.

**Pre-inference retrieval** — also called "eager loading" — front-loads context into the prompt before the model generates any output. The system prompt, any RAG results, relevant documentation, and conversation history are all assembled and passed to the model in a single call.

The advantage is latency: the model has everything it needs on the first inference. The disadvantage is waste: you are spending tokens on context that may not be relevant. Worse, irrelevant context actively degrades performance (we will explore this in depth in Chapter 3).

**Just-in-time context** takes the opposite approach. The agent starts with minimal context — perhaps just the user's request and a set of available tools — and loads information dynamically as it reasons about what it needs. Instead of pre-loading an entire codebase summary, it starts with a list of file paths and only reads the files it determines are relevant.

The key insight is that **lightweight identifiers are cheap; full content is expensive.** A file path is a few tokens. The file's contents might be thousands. A function signature is a line. The function body might be a hundred lines. An API endpoint URL is a token. The response payload might be a page.

Effective agents maintain a mental model composed primarily of lightweight identifiers — file paths, function names, class names, URLs, database table names — and only "inflate" these identifiers into full content when they need the details for a specific action. This is the *Lazy Loading* pattern applied to LLM context.

### Progressive Disclosure

Progressive disclosure is a design pattern borrowed from user interface design: reveal information progressively as it becomes relevant, rather than overwhelming the user (or, in our case, the model) with everything at once.

In agent systems, progressive disclosure manifests as a multi-phase exploration strategy:

1. **Orientation phase.** The agent scans high-level structure: directory layout, key configuration files, README content. It builds a mental map of the territory.

2. **Targeted exploration.** Based on its mental map, the agent identifies the most promising areas and dives deeper: reading specific files, following import chains, examining function signatures.

3. **Detail extraction.** Once the agent has located the relevant code, it reads the specific implementations, examines edge cases, and gathers the precise details needed for action.

Each phase informs the next. The orientation phase tells the agent *where to look*. Targeted exploration tells the agent *what to look at*. Detail extraction gives the agent *what it needs to act*.

This is not just an optimization — it is a *necessity*. Loading an entire codebase into context is impossible for any non-trivial project. Progressive disclosure is how agents cope with information spaces that vastly exceed their context window.

### Subagents for Context Gathering

One of the most powerful patterns in context gathering is **parallel subagent exploration**. Instead of a single agent serially exploring different areas of a codebase, the orchestrator spawns multiple subagents, each with a focused exploration mandate, and they work in parallel.

Each subagent operates in its own context window. This is a critical architectural property. A subagent exploring the authentication system does not need to carry tokens about the payment system in its context. Clean, focused context windows produce better results — we will formalize this in Chapter 3.

The subagent pattern follows a map-reduce structure:

1. **Fan-out.** The orchestrator identifies independent areas to explore and spawns a subagent for each. "Investigate how authentication works." "Find all database migration files and summarize the schema." "Identify the API endpoints related to user management."

2. **Parallel exploration.** Each subagent explores its domain using the full gather-context toolkit: reading files, searching patterns, following references. A subagent might consume tens of thousands of tokens of raw file content during its exploration.

3. **Compression and return.** Each subagent distills its findings into a condensed summary — typically 1,000 to 2,000 tokens. The critical art is in the compression: the summary must preserve the information the orchestrator needs for decision-making while discarding the raw details that were only needed for exploration.

4. **Synthesis.** The orchestrator receives the condensed summaries from all subagents and synthesizes them into a coherent understanding of the problem space. It now has broad, multi-faceted context at a fraction of the token cost of serial exploration.

This pattern achieves something remarkable: it allows an agent system to effectively "read" far more information than fits in any single context window, while maintaining the quality advantages of focused, reasoning-guided exploration.

The tradeoff is cost and complexity. Each subagent is an LLM inference call (often multiple calls). Orchestrating parallel exploration requires careful prompt design to ensure subagents know what to look for and how to report back. But for complex tasks in large codebases, the quality improvement is substantial.

---

## 2.2 Take Action

Once the agent has gathered sufficient context, it acts. The action phase is where the agent produces observable effects in the world: writing code, modifying files, executing commands, calling APIs. This is the phase that delivers value.

### Tools as Primary Building Blocks

In the agent paradigm, **tools are not accessories — they are the primary mechanism of action.** An agent without tools is just a chatbot. Tools are what transform a language model from a text generator into an actor in the world.

This has a subtle but important implication for agent design: **tools occupy prime real estate in the context window.** The tool definitions — their names, descriptions, and parameter schemas — are among the first things the model sees in every inference call. They shape the model's understanding of what actions are possible.

This means tool design is not a peripheral concern. It is one of the highest-leverage design decisions in an agent system. A well-designed tool set guides the agent toward productive actions. A poorly designed one confuses it.

The principles of good tool design mirror the principles of good API design:

- **Clear naming.** A tool called `read_file` is immediately understood. A tool called `process_data` is ambiguous. The model uses tool names as strong signals for action selection.

- **Focused responsibility.** Each tool should do one thing well. A tool that reads files, writes files, and executes shell commands is three tools wearing a trench coat. The model will struggle to use it correctly because the parameter space is overloaded.

- **Descriptive schemas.** Parameter descriptions are not just documentation — they are instructions to the model. A parameter described as `"The file path to read (absolute path required)"` will receive absolute paths. A parameter described as `"path"` will receive whatever the model guesses you want.

- **Informative error messages.** When a tool fails, the error message becomes context for the next iteration of the loop. `"File not found: /src/main.ts"` helps the agent recover. `"Error code 2"` does not.

### Bash and Scripts as General-Purpose Tools

There is a school of thought that says every agent capability should be wrapped in a purpose-built tool with a typed schema. This is wrong, or at least incomplete.

A general-purpose shell execution tool — the ability to run arbitrary bash commands — is one of the most valuable tools an agent can have. Here is why:

1. **Combinatorial coverage.** The Unix command line provides thousands of utilities that can be composed in arbitrary ways. No finite set of purpose-built tools can match this coverage. An agent that can run `jq '.dependencies | keys' package.json` does not need a dedicated "list npm dependencies" tool.

2. **Adaptability.** When an agent encounters an unexpected situation — a build system it has never seen, a file format it does not recognize, a deployment target with unusual requirements — bash gives it a way to investigate and adapt. Purpose-built tools only handle the scenarios their designers anticipated.

3. **Script generation.** Agents can write and execute scripts, creating new capabilities on the fly. This is a form of *meta-tooling*: the agent uses a tool (bash) to create new tools (scripts) that solve the specific problem at hand.

The tradeoff is safety. An unrestricted bash tool can delete files, exfiltrate data, or corrupt system state. Production agent systems must sandbox bash execution — containerized environments, restricted file system access, network policies, command allowlists. The engineering challenge is providing the power of general-purpose execution while constraining the blast radius of mistakes.

### Code Generation as Precise Output

When an agent writes code, it is producing one of the most precise and composable forms of output available. Code is:

- **Unambiguous.** Unlike natural language, code has a single interpretation (modulo undefined behavior in some languages). The agent's intent is captured exactly.

- **Executable.** The output can be directly tested, run, and verified. This creates a tight feedback loop with the verify phase of the agent loop.

- **Composable.** Generated code integrates with existing code. It can call functions, import modules, implement interfaces. The agent's output participates in the broader system.

- **Reusable.** Unlike a natural language answer that addresses one question, generated code can be used repeatedly. A utility function written by an agent serves the same purpose as one written by a human.

This is why code generation agents are among the most successful applications of the agent paradigm. The output format naturally supports verification (run the tests, check the types, lint the code) and composition (import it, call it, extend it).

### MCP: Model Context Protocol

MCP (Model Context Protocol) is an open standard for connecting agents to external tools and data sources. It addresses a real problem: without a standard protocol, every agent-tool integration is a bespoke implementation. If you have N agent frameworks and M tool providers, you need N x M integrations. MCP reduces this to N + M.

The protocol defines a standard interface for:

- **Tool discovery.** An agent can query an MCP server to learn what tools are available, their schemas, and their descriptions.
- **Tool invocation.** A standard request/response format for calling tools and receiving results.
- **Resource access.** A mechanism for agents to read structured data from external systems.

MCP matters for agent engineering because it enables a modular tool ecosystem. An agent can connect to an MCP server that provides GitHub integration, another that provides database access, and another that provides monitoring data — all through the same protocol. Tools become interchangeable components rather than tightly coupled integrations.

The architectural pattern here is the *Adapter* pattern at protocol scale. MCP adapts diverse external systems into a uniform interface that any agent framework can consume.

---

## 2.3 Verify Work

The verify phase is what separates robust agent systems from fragile ones. An agent that acts without verifying is guessing. An agent that verifies its work is *engineering*.

This is perhaps the single most important insight in agent system design: **agents that check and improve their own output are fundamentally more reliable than agents that do not.** The verification phase transforms a stochastic process (LLM generation) into an iterative refinement process that converges toward correctness.

### Rules-Based Feedback

The most reliable form of verification is deterministic: tools that apply fixed rules to the agent's output and report violations.

**Type checkers** (TypeScript's `tsc`, mypy, the Rust compiler) verify that the agent's code is structurally consistent. Type errors are precise, actionable, and unambiguous. An agent that generates TypeScript, runs the type checker, reads the errors, and fixes them will produce more reliable code than an agent that generates untyped JavaScript and hopes for the best.

This leads to a practical design principle: **prefer typed languages for agent-generated code.** The type system is a free verification layer. TypeScript over JavaScript. Python with type hints over untyped Python. Rust over C. The type checker acts as a tireless reviewer that catches structural errors the LLM might introduce.

**Linters** (ESLint, Pylint, Clippy) catch style violations, potential bugs, and anti-patterns. They are less precise than type checkers — some lint rules are opinionated rather than correctness-oriented — but they provide valuable signal, especially for catching common mistakes like unused variables, unreachable code, or missing error handling.

**Test suites** are the gold standard of verification. If the agent modifies code, running the existing test suite reveals whether the modification broke anything. If the agent writes new code, writing tests alongside it (or running tests written by a human) verifies the behavior. A failing test is an unambiguous signal that something is wrong, and the test output often contains enough information for the agent to diagnose and fix the issue.

The pattern here is the **Red-Green-Refactor loop** borrowed from test-driven development, but executed by the agent: generate code (potentially red), run tests (verify), fix failures (green), clean up (refactor). The agent does not need to be told to follow TDD. It naturally falls into this pattern when verification tools are available and the loop structure encourages iteration.

### Visual Feedback

Some outputs cannot be verified by rules alone. A web page that passes all lint checks and type checks might still look completely wrong — broken layout, overlapping elements, missing content, incorrect colors.

Visual verification tools like **Playwright** (for capturing screenshots of rendered web pages) give agents access to the same feedback channel that human developers use: looking at the output.

The pattern is straightforward:

1. The agent generates or modifies frontend code.
2. A tool renders the page and captures a screenshot.
3. The screenshot is fed back into the agent's context as an image.
4. The agent examines the screenshot and identifies visual issues.
5. The agent modifies the code to fix the issues.

This works because modern multimodal models can interpret screenshots with reasonable accuracy. They can identify layout problems, read text in rendered pages, notice missing elements, and compare the rendered output against design specifications.

Visual feedback is particularly valuable for CSS and layout work, where the relationship between code and visual output is complex and non-obvious. A model that can see the rendered result closes the feedback loop that would otherwise require a human in the loop.

### LLM-as-Judge

When neither rules-based nor visual verification is available, an LLM can evaluate the agent's output. This is the **LLM-as-Judge** pattern: a separate LLM call (or the same model with a different prompt) evaluates whether the output meets the requirements.

LLM-as-Judge is the least reliable verification method. It adds latency (another LLM inference call), cost (more tokens), and introduces its own error modes (the judge can be wrong). The judge's evaluation is probabilistic, not deterministic — run it twice and you might get different answers.

But LLM-as-Judge is sometimes the only option. Evaluating whether a natural language summary accurately captures the key points of a document, whether generated documentation is clear and complete, or whether a code refactoring preserves the original intent — these are judgment calls that resist formalization into rules.

The practical guidance is:

- **Use rules-based verification whenever possible.** It is faster, cheaper, and more reliable.
- **Use visual verification for visual outputs.** It catches things rules cannot.
- **Use LLM-as-Judge when the cost of errors is high enough to justify the overhead and no deterministic alternative exists.** In production systems, this typically means high-stakes outputs where any improvement in quality is worth the additional latency and cost.

Do not reach for LLM-as-Judge as a default. It is a tool of last resort for verification problems that resist formalization.

---

## 2.4 The Iteration

The individual phases — gather, act, verify — are valuable in isolation. But the real power of the agent loop emerges from the *iteration*: the way each phase informs the next, creating a compounding cycle of improvement.

### How the Loop Compounds

Consider what happens when an agent attempts a code modification:

**Iteration 1:**
- *Gather:* The agent reads the user's request and explores the relevant files.
- *Act:* The agent modifies the code based on its understanding.
- *Verify:* The type checker reports three errors.

**Iteration 2:**
- *Gather:* The agent reads the error messages. They point to specific lines and explain the type mismatches. This is *new context* — information that did not exist before the agent acted.
- *Act:* The agent fixes two of the three errors.
- *Verify:* The type checker reports one remaining error. The test suite reports a failing test.

**Iteration 3:**
- *Gather:* The agent reads the remaining type error and the test failure. The test failure reveals an edge case the agent did not consider.
- *Act:* The agent fixes the type error and adds handling for the edge case.
- *Verify:* All checks pass.

Each iteration produces information that makes the next iteration more targeted and more likely to succeed. The agent starts with a broad understanding and progressively refines it through interaction with the environment. This is the *Iterative Refinement* pattern, and it is why agents can solve problems that would be impossible for a single-shot generation.

Notice the critical role of verification in this process. Without the type checker and test suite, the agent would have stopped after Iteration 1 and delivered broken code. The verification tools do not just catch errors — they *generate context* for the next iteration.

### Error Recovery and Self-Correction

Errors in agent systems are not just tolerable — they are *expected and productive*. Every error message is information. Every failed test is a specification. Every stack trace is a map to the problem.

The agent loop naturally supports error recovery through the same mechanism it uses for everything else: gather the error context, take corrective action, verify the fix. There is no special error-handling pathway. The loop itself *is* the error-handling mechanism.

This has a profound implication for agent design: **you should optimize for fast, informative failures rather than trying to prevent all errors.** An agent that fails quickly and gets a clear error message will recover faster than an agent that spends extra time trying to get it right on the first attempt.

Practical patterns for error recovery:

- **Retry with context.** When a tool call fails, include the error message in the next iteration's context. The agent can often diagnose the problem from the error alone.

- **Fallback strategies.** If the agent's preferred approach fails repeatedly, it should consider alternative approaches. "The build fails with Webpack? Let me check if this project uses a different bundler."

- **Escalation.** If the agent cannot recover after several iterations, it should report what it tried, what failed, and why, rather than continuing to loop. Knowing when to stop is as important as knowing how to continue.

### Stopping Conditions and Checkpoints

An agent loop without a stopping condition is an infinite loop with a credit card attached. Defining when the agent should stop is an engineering decision with significant tradeoffs.

**Verification-based stopping** is the ideal: the agent stops when all verification checks pass. Tests pass, types check, linter is clean, visual output matches expectations. This is the most reliable stopping condition because it is grounded in objective criteria.

**Iteration limits** are a safety net: the agent stops after N iterations regardless of verification status. This prevents runaway loops where the agent makes the same mistake repeatedly or oscillates between two states. Typical limits range from 5 to 20 iterations depending on task complexity.

**Token budget limits** cap the total tokens consumed across all iterations. This is a cost control mechanism, but it also serves as a proxy for task complexity — if the agent has consumed its token budget without converging, the task may be too complex for the current approach.

**Checkpoints** are intermediate saving points within a long-running agent loop. After completing a significant subtask (e.g., successfully modifying one of three files), the agent saves its progress so that if a later step fails, it does not need to redo earlier work. Checkpoints are especially important for long-horizon tasks where the cost of restarting from scratch is high.

The best agent systems combine these approaches: verify until all checks pass, but stop after a maximum number of iterations, and checkpoint progress along the way.

---

## Summary

The agent loop — gather context, take action, verify work, repeat — is the foundation of every effective agent system. Its power comes not from any individual phase but from the interaction between phases: verification generates context, context informs action, action produces artifacts for verification.

Understanding this loop deeply changes how you design agent systems. You stop trying to make each phase perfect in isolation and start optimizing for the *quality of information flow between phases*. You invest in tools that produce informative errors. You design verification steps that generate useful context. You structure context gathering to support the specific actions the agent will take.

The agent loop is simple. Building it well is not. The next chapter examines the resource that constrains every phase of this loop: the context window.
