# Chapter 3: Context Engineering — The Most Critical Skill

If you take one thing from this book, let it be this chapter.

Every failure mode in agent systems — hallucination, instruction-following failures, lost context, degraded reasoning, tool misuse — traces back to the same root cause: the model did not have the right tokens in its context window at the moment it needed them. Context engineering is the discipline of ensuring it does.

This is not prompt engineering. Prompt engineering is about crafting the right words — phrasing instructions clearly, structuring few-shot examples, choosing the right tone. Prompt engineering is a subset of context engineering the way a single SQL query is a subset of database administration. Context engineering is the broader, harder, more consequential problem: **curating and maintaining the optimal set of tokens throughout an entire LLM inference session.**

The distinction matters because prompt engineering is static. You write a prompt, you test it, you ship it. Context engineering is dynamic. The context changes with every tool call, every user message, every iteration of the agent loop. The tokens that were optimal at step 1 may be noise at step 15. Managing this evolution — deciding what enters the context, what stays, what gets summarized, and what gets evicted — is the central engineering challenge of building agent systems.

---

## 3.1 Context as a Finite Resource

A context window is not a bottomless bag into which you can throw tokens and expect the model to use them all equally. It is a finite resource with complex economics, and treating it carelessly is one of the most common mistakes in agent engineering.

### The Attention Budget

Think of every token in the context window as drawing from a fixed attention budget. Each token the model processes must be attended to — compared against every other token to determine relevance and relationships. Adding a token does not just cost the compute to process that token. It costs the compute to relate that token to *every other token already in the context.*

This is not a metaphor. It is a direct consequence of transformer architecture.

The self-attention mechanism at the heart of transformer models computes pairwise relationships between all tokens in the context. For a context of *n* tokens, the model computes *n*^2 attention scores (per head, per layer). This quadratic relationship means that the cost of each additional token is not constant — it is proportional to the current context length.

But the compute cost is not the real problem. The real problem is what happens to the model's *reasoning quality* as context grows.

### Why Context Degrades Performance

When you add 1,000 tokens of irrelevant content to a context window, you are not just wasting 1,000 tokens of capacity. You are creating 1,000 * *n* new pairwise relationships that the attention mechanism must evaluate, where *n* is the number of tokens already in context. Each of these relationships is a potential distraction — a spurious correlation that the model might attend to instead of the relevant information.

This is the **signal-to-noise ratio** problem applied to transformer attention. The model's attention heads have finite capacity. They can only strongly attend to a limited number of positions. As you add noise tokens, the attention heads must spread their focus across more positions, diluting the attention available for the tokens that actually matter.

The empirical evidence is overwhelming. Researchers have repeatedly demonstrated that adding irrelevant context to prompts degrades performance on tasks where the model performs well with lean context. This is not a hypothetical concern — it is a measured, reproducible phenomenon.

The implication is counterintuitive for engineers accustomed to the "more data is better" heuristic of traditional software: **in context engineering, less is often more.** A 500-token context with precisely relevant information will outperform a 50,000-token context where the relevant information is buried among irrelevant details.

### Context Rot

There is a temporal dimension to context degradation that is particularly insidious in agent systems. As an agent works through a long task, its context window accumulates artifacts from every iteration: tool calls, tool results, reasoning traces, error messages, intermediate outputs. Each iteration adds tokens. The context grows monotonically.

This creates a phenomenon we can call **context rot**: the progressive degradation of the model's ability to recall and utilize information as the context fills with historical artifacts. Information from early in the conversation gets pushed further and further from the model's current position, surrounded by an ever-growing mass of intermediate content.

Context rot manifests in predictable ways:

- The agent "forgets" instructions from the system prompt.
- The agent re-reads files it already read earlier in the conversation.
- The agent contradicts decisions it made in earlier iterations.
- The agent's reasoning becomes less coherent and more prone to hallucination.

These are not bugs in the model. They are the inevitable consequence of finite attention spread across too many tokens. Context rot is the primary reason why long-running agent sessions degrade in quality and why context management strategies (compaction, summarization, subagent architectures) are not optional optimizations — they are essential infrastructure.

### Diminishing Marginal Returns

The relationship between context size and performance follows a characteristic curve. The first few hundred tokens of relevant context produce dramatic improvements — the model goes from having no information to having the key facts. The next few thousand tokens add nuance, edge cases, and supporting details. Beyond that, each additional token contributes less and less to the model's understanding.

But the *cost* of each token — in terms of attention dilution, compute, and latency — remains constant or increases. This means that past a certain point, adding more context actively harms net performance: the dilution effect outweighs the informational benefit.

The optimal context size is task-dependent. A simple code fix might need 500 tokens of context. A complex architectural refactoring might need 20,000. The engineering skill is in recognizing where you are on the curve and stopping before you cross into negative returns.

---

## 3.2 The Anatomy of Effective Context

Not all tokens are created equal. The structure and composition of the context window matter as much as the total token count. This section examines the key components of effective context and the principles that govern their design.

### System Prompts: Finding the Right Altitude

The system prompt is the foundation of the context window. It establishes the agent's identity, capabilities, constraints, and behavioral patterns. Every subsequent token is interpreted in the light of the system prompt.

The central challenge in system prompt design is finding the **Goldilocks zone** — the right level of abstraction between two failure modes:

**Too specific (brittle).** A system prompt that hardcodes exact procedures for every scenario is fragile. It works perfectly for anticipated cases and fails completely for unanticipated ones. "When the user asks about authentication, first check the auth.ts file, then look for middleware in server.ts, then..." This breaks the moment the codebase uses a different file structure.

**Too vague (unhelpful).** A system prompt that provides only high-level guidance gives the agent too many degrees of freedom. "You are a helpful coding assistant. Write good code." The model has no constraints to guide its behavior, and you are relying entirely on its pretraining to make reasonable choices.

The right altitude is **principle-based with concrete anchors.** State the principles that should guide behavior, then provide a few concrete examples that illustrate how those principles apply. This gives the model enough structure to generalize without being so rigid that it cannot adapt.

Compare:

*Brittle:* "Always use the `read_file` tool before the `edit_file` tool. Never use `edit_file` without first calling `read_file` on the same path."

*Vague:* "Make sure you understand files before editing them."

*Right altitude:* "Read files before editing them to ensure your changes are based on the current content. The `read_file` tool returns the file with line numbers, which you should use to verify the exact text you intend to modify."

The right-altitude version explains *why* (ensure changes are based on current content), *how* (use the read_file tool), and provides a concrete detail (line numbers for verification) without rigidly prescribing the exact sequence of operations.

### Tools: The Action Contract

Tool definitions are not just interface specifications — they are a **contract between the agent and its environment** that shapes the agent's behavior in fundamental ways.

When a model sees a tool definition, it learns three things:

1. **What is possible.** The available tools define the agent's action space. An agent with a `run_tests` tool will consider running tests. An agent without one will not. Tool availability directly shapes the agent's problem-solving strategy.

2. **How to act.** Parameter schemas, descriptions, and examples teach the model the correct way to invoke each tool. Well-documented tools are used correctly more often.

3. **What to expect.** Return type descriptions and error documentation prepare the model to interpret results and handle failures.

This makes tool set design a high-leverage activity. But there is a critical failure mode: **tool set bloat.**

Every tool definition consumes tokens in the context window. A tool with a detailed description, five parameters with descriptions, and example usage might consume 200-400 tokens. Twenty tools consume 4,000-8,000 tokens — a significant fraction of a smaller context window, and enough to noticeably dilute attention even in a large one.

Worse, a bloated tool set confuses the model's action selection. Given forty available tools, the model must reason about which tool is appropriate for the current step. This is a combinatorial problem that scales with the number of tools. Research consistently shows that models perform better with a focused set of well-designed tools than with an extensive set that covers every possible need.

The pattern to follow is the **Minimal Sufficient Tool Set**: include the fewest tools that cover the agent's required capabilities, and make each tool's purpose clearly distinct from every other tool. If two tools have overlapping capabilities, merge them or remove the less essential one.

For agent systems that genuinely need many tools, consider **dynamic tool loading** — only include tool definitions relevant to the current phase of the task. A code exploration phase might load search and read tools. An editing phase might swap those for write and test tools. This keeps the active tool set small even if the total available tools are numerous.

### Few-Shot Examples: Pictures Worth a Thousand Words

Few-shot examples are the most token-efficient way to communicate complex behavioral expectations to a model. A single well-chosen example can replace paragraphs of instruction.

This is because examples communicate at multiple levels simultaneously:

- **Format.** The example shows what the output should look like — its structure, length, style, and formatting.
- **Reasoning pattern.** The example demonstrates how to approach the problem — what to consider, what to prioritize, how to handle ambiguity.
- **Quality bar.** The example sets the standard for thoroughness, precision, and care.
- **Edge case handling.** The example can demonstrate how to handle tricky situations that are difficult to describe in abstract rules.

The empirical evidence strongly supports a specific pattern for few-shot example design:

**Three to five diverse, canonical examples beat a laundry list of edge cases.**

Diversity is critical. Three examples that cover different scenarios, different complexities, and different edge cases teach the model to generalize. Three examples that are minor variations of the same scenario teach the model to memorize a template.

Canonicality matters too. Each example should be a *textbook* instance of its category — clear, unambiguous, and representative. Save the weird edge cases for explicit rules. Examples are for teaching the common pattern; rules are for specifying exceptions to it.

The reason this works relates to how in-context learning functions in transformers. The model identifies patterns across examples and generalizes them. With diverse examples, the model extracts the *underlying pattern* rather than surface-level similarities. With too many examples, the model has more patterns to reconcile and may overfit to superficial features.

The practical guidance: invest significant effort in choosing and crafting your few-shot examples. They are among the highest-leverage tokens in your entire context window. A mediocre example does not just waste tokens — it teaches the wrong lesson.

---

## 3.3 Progressive Disclosure Applied to Agent Context

The progressive disclosure pattern from Chapter 2 is so important for context engineering that it deserves its own deep treatment. The core insight, formalized by teams at OpenAI, Anthropic, and elsewhere, is this:

**Structure your agent's knowledge as a hierarchy, and load only the level of detail needed for the current decision.**

### The Table of Contents Pattern

The most successful implementation of progressive disclosure for agent context is the **Table of Contents Pattern**, exemplified by OpenAI's `AGENTS.md` convention and Anthropic's `CLAUDE.md`.

The pattern works as follows:

1. **The index file** (~100 lines) is loaded into the context at the start of every session. It contains high-level orientation: what the project is, what the major components are, what conventions to follow, and — critically — *pointers to where detailed information lives.*

2. **The knowledge base** (a structured `docs/` directory) contains the actual detailed information: design documents, execution plans, product specifications, API references, architecture decision records. Each document is self-contained and focused on a single topic.

3. **The agent navigates between levels** based on need. When working on authentication, it reads the auth design doc. When working on the API, it reads the API spec. It never loads both simultaneously unless the task requires understanding their interaction.

This is the **Inversion of Control** principle applied to context. Instead of the system deciding upfront what information the agent needs (and inevitably getting it wrong), the system provides a map and lets the agent decide what to load.

The index file should follow specific design principles:

- **Brevity.** ~100 lines maximum. Every line must earn its place.
- **Orientation over instruction.** Tell the agent what things are and where they live, not step-by-step procedures.
- **Stable references.** Point to files and directories that exist and are kept up to date. Stale pointers are worse than no pointers.
- **Convention documentation.** Naming conventions, file organization patterns, and architectural principles belong here because they apply to *every* task.

### The Cost of Over-Loading Context

It is worth stating this point with maximum emphasis because it is so frequently violated:

**Every token in the agent's context window that is not directly relevant to the current task is noise. And noise is not neutral — it is actively harmful.**

A 2,000-line instruction file does not just waste tokens. It actively degrades the agent's performance in three measurable ways:

1. **Attention dilution.** The model's attention heads distribute focus across all tokens. Irrelevant tokens steal attention from relevant ones. The model is literally paying less attention to your important instructions because it is busy processing your unimportant ones.

2. **Instruction confusion.** Long instruction files inevitably contain contradictions, ambiguities, and edge cases that conflict. The model must reconcile these conflicts, and its resolution may not match your intent. Shorter, focused instructions have fewer internal contradictions.

3. **Behavioral instability.** With a large instruction surface, small changes to the context (a different user message, a different tool result) can cause the model to attend to different parts of the instructions, producing inconsistent behavior across runs. A focused instruction set produces more predictable behavior.

The practical test is simple: for every block of text in your agent's context, ask "Does the agent need this information *right now* to make its *current* decision?" If the answer is no, it should not be in the context. It should be in a file the agent can read when it needs it.

---

## 3.4 Context for Long-Horizon Tasks

Short tasks — answer a question, fix a bug, write a function — fit comfortably within a single context window. The agent gathers context, acts, verifies, and finishes. Context management is straightforward because there is not enough history to cause problems.

Long-horizon tasks are different. A multi-hour coding session, a complex refactoring across dozens of files, a research task that requires synthesizing information from many sources — these tasks generate far more tokens than any context window can hold. Without active context management, they inevitably succumb to context rot.

This section presents four strategies for managing context across long-horizon tasks, along with guidance on when to use each.

### Strategy 1: Compaction

Compaction is the most straightforward approach: when the conversation history approaches the context limit, summarize it and reinitialize the context with the summary.

The mechanics are simple:

1. Monitor the token count of the current context.
2. When it reaches a threshold (typically 70-80% of the maximum), trigger compaction.
3. Generate a summary of the conversation so far, focusing on: decisions made, current state, remaining tasks, and key context that must be preserved.
4. Start a new context with the system prompt, the summary, and any currently relevant artifacts.

The art of compaction is in the **selection of what to preserve.** Not all information in the conversation history has equal value. Some information is critical for future decisions; some was only relevant to past decisions.

The compaction heuristic should follow a two-phase approach:

**Phase 1: Maximize recall.** First, err on the side of including too much in the summary. It is better to carry forward a slightly verbose summary than to lose a critical piece of context. The goal is to ensure that nothing *important* is lost.

**Phase 2: Improve precision.** Once you have a recall-optimized summary, review it and remove information that is unlikely to be needed. Details of error messages that were already resolved. Intermediate reasoning that led to a final decision (keep the decision, drop the reasoning). Raw file contents that have been processed and acted upon.

The result should be a summary that is 5-15% of the original conversation length, preserving the *conclusions* and *state* while discarding the *process*.

Compaction works best for tasks characterized by extensive back-and-forth — debugging sessions, iterative refinement, exploratory conversations — where the conversation generates a lot of tokens but the *net* information content is relatively low. Much of the conversation is exploration, dead ends, and intermediate states that are no longer relevant.

### Strategy 2: Tool Result Clearing

A more surgical approach than full compaction is selectively clearing tool results that are no longer relevant. This targets the single largest source of token bloat in agent conversations: raw tool output.

Consider a typical agent workflow:

1. Agent calls `grep` to search for a pattern. Result: 200 lines, ~2,000 tokens.
2. Agent reads a file. Result: 300 lines, ~3,000 tokens.
3. Agent reads another file. Result: 150 lines, ~1,500 tokens.
4. Agent makes an edit. Result: confirmation, ~50 tokens.
5. Agent runs tests. Result: 100 lines of output, ~1,000 tokens.

After five tool calls, the context contains ~7,550 tokens of tool results. After fifty tool calls — a modest number for a complex task — the context might contain 75,000 tokens of tool results. The vast majority of these results are irrelevant to the current step. The agent has already extracted the relevant information and used it.

Tool result clearing replaces old tool results with compact summaries or removes them entirely. The file the agent read 20 steps ago is replaced with a note: `[Read file: src/auth.ts, 287 lines. Key findings: JWT validation in validateToken(), token refresh in refreshAuth().]` This preserves the *metadata* (the agent read this file and found these things) while reclaiming the tokens consumed by the raw content.

The implementation requires tracking which tool results have been "consumed" — used by the model in a subsequent response — and which are still potentially needed. Unconsumed tool results should be preserved; consumed results can be cleared or summarized.

### Strategy 3: Structured Note-Taking (Agentic Memory)

Note-taking is the most sophisticated single-agent context management strategy. Instead of trying to fit everything into the context window, the agent maintains an *external memory* — a structured set of notes persisted outside the context window — that it can write to and read from as needed.

The key insight is that the context window is *working memory*, not *long-term memory*. Humans do not keep every fact they have ever learned in active working memory. They maintain a vast long-term memory and selectively load relevant information into working memory when needed. Agents should do the same.

The structured note-taking pattern works as follows:

1. **The agent maintains a scratchpad** — a file, a database, or a structured document — outside its context window.
2. **At milestones, the agent writes notes** summarizing what it has learned, what decisions it has made, and what remains to be done.
3. **At the start of each major phase, the agent reads relevant notes** to reload context it needs for the current phase.
4. **The notes are structured** — organized by topic, tagged by relevance, dated for recency — so the agent can selectively load only what it needs.

The canonical example of this pattern is Anthropic's demonstration of Claude playing Pokemon. The game requires tracking objectives, navigating complex maps, remembering NPC dialogues, managing inventory, and making strategic decisions — across thousands of steps that far exceed any context window.

The agent maintained structured notes organized by category:
- **Current objectives:** What it is trying to accomplish right now.
- **Map knowledge:** What it has learned about the game world layout.
- **Inventory and team:** Current state of game resources.
- **Key NPC information:** Important dialogue and quest information.
- **Strategic observations:** Patterns it has noticed, strategies that work.

At each step, the agent loaded only the notes relevant to its current situation. Walking through a familiar area, it loaded map notes. Entering a battle, it loaded team and strategy notes. Talking to an NPC, it loaded quest notes. This kept the context window focused and lean while giving the agent access to far more total information than the window could hold.

This is the **Working Set** pattern from operating systems, applied to LLM context. The operating system keeps the active working set of pages in physical memory and pages out inactive data to disk. The agent keeps the active working set of notes in context and "pages out" inactive notes to external storage.

Note-taking works best for tasks with clear milestones and phases — iterative development, multi-step research, long-running projects. The milestones provide natural points for note-writing, and the phases provide natural boundaries for selective note-loading.

### Strategy 4: Sub-Agent Architectures

Sub-agent architectures address context management by distributing work across multiple agents, each with its own clean context window. Instead of one agent trying to hold everything in a single window, an orchestrator delegates subtasks to specialized sub-agents that start fresh.

We introduced this pattern in Chapter 2 for context gathering. Here we examine it as a context management strategy for long-horizon tasks.

The architectural pattern is **Hierarchical Task Decomposition**:

1. **The orchestrator** maintains a high-level view of the task: overall objectives, progress tracking, major decisions, and inter-task dependencies.
2. **Sub-agents** receive focused mandates: "Refactor the authentication module to use JWT tokens." "Update all API tests to use the new authentication flow." "Update the documentation to reflect the new auth architecture."
3. **Each sub-agent starts with a clean context window** containing only the system prompt, the specific mandate, and any context the orchestrator determines is necessary for that subtask.
4. **Sub-agents return condensed results** to the orchestrator: what they did, what they changed, what issues they encountered.

The context management benefit is profound. A sub-agent working on authentication does not carry tokens about API tests in its context. Its attention is fully focused on the authentication code. When it finishes and returns a summary, the orchestrator does not carry the raw details of the authentication changes — just the summary.

This achieves **context isolation**: each agent's context window contains only tokens relevant to its current task. There is no context rot because each sub-agent's context is short-lived and focused. There is no attention dilution because irrelevant information is never loaded.

The tradeoff is coordination overhead. The orchestrator must decompose the task correctly, provide sufficient context to each sub-agent, handle dependencies between subtasks, and synthesize results coherently. This requires careful prompt engineering for the orchestrator and well-defined interfaces between agents.

Sub-agent architectures work best for tasks with **natural parallelism** — complex research requiring exploration of multiple independent areas, large-scale refactoring affecting multiple independent modules, or any task that can be decomposed into subtasks with minimal inter-dependencies.

### Choosing the Right Strategy

The strategies are not mutually exclusive — production systems often combine multiple approaches. But as a starting point:

| Strategy | Best For | Key Strength | Key Weakness |
|---|---|---|---|
| **Compaction** | Extended back-and-forth, debugging sessions | Simple to implement | Lossy — important details can be lost in summarization |
| **Tool Result Clearing** | Tool-heavy workflows with many reads/searches | Surgical, preserves conversation structure | Only addresses one source of bloat |
| **Structured Note-Taking** | Iterative development with clear milestones | Agent controls what to remember | Requires the agent to reliably write good notes |
| **Sub-Agent Architecture** | Parallelizable tasks, complex research | Clean context isolation | Coordination overhead, potential for duplicated work |

For a typical complex coding task — a multi-file refactoring that takes an hour — you might use:

- **Tool result clearing** continuously, to prevent raw file contents from bloating the context.
- **Structured note-taking** at each milestone ("Finished refactoring module A. Changed the interface from X to Y. Module B depends on this and needs to be updated next.").
- **Sub-agents** for independent subtasks that can be parallelized ("Update all test files to use the new interface" while the main agent continues with the next module).
- **Compaction** as a safety net, triggered if the context grows too large despite the other strategies.

---

## 3.5 Putting It All Together: The Context Engineering Mindset

Context engineering is not a technique. It is a mindset — a way of thinking about every design decision in your agent system through the lens of "what will be in the context window when this decision is made?"

**When designing tools,** think about context. Verbose tool output wastes tokens. Structured, concise output preserves the attention budget for reasoning. A tool that returns a 5,000-line file when the agent only needs 20 lines is a context engineering failure.

**When writing system prompts,** think about context. Every instruction competes with task-specific information for attention. An instruction that applies to 1% of tasks but is present for 100% of them is a net negative.

**When structuring agent knowledge,** think about context. Information should be organized for selective retrieval, not bulk loading. The goal is not to make information *available* — it is to make the *right* information available *at the right time*.

**When designing the agent loop,** think about context. Each iteration adds tokens. What strategies will you use to prevent context rot? At what point will you compact, summarize, or delegate to a sub-agent?

**When debugging agent failures,** think about context. When an agent makes a mistake, the first question should be: "What was in the context window when it made this decision?" Often, the answer reveals the problem: the relevant information was missing, buried under irrelevant tokens, or contradicted by other instructions.

The engineers who build the best agent systems are not the ones who write the cleverest prompts or use the most advanced models. They are the ones who are most disciplined about what goes into the context window — and what stays out.

---

## Summary

Context engineering is the discipline of curating and maintaining the optimal set of tokens during LLM inference. It is the most critical skill in agent engineering because every agent failure mode traces back to suboptimal context.

The context window is a finite resource with diminishing marginal returns, governed by the quadratic attention mechanism of transformer architecture. Every irrelevant token actively harms performance by diluting attention from relevant tokens.

Effective context is structured in layers: system prompts at the right altitude, focused tool definitions, diverse few-shot examples, and task-specific information loaded just in time. The progressive disclosure pattern — maintaining a table of contents in context and loading details on demand — is the single most important structural pattern for agent context.

Long-horizon tasks require active context management: compaction, tool result clearing, structured note-taking, and sub-agent architectures. These are not optional optimizations — they are essential infrastructure for any agent that operates beyond a single short conversation.

The context engineering mindset is simple: for every token in the context window, ask whether it is earning its keep. If it is not directly helping the model make its current decision, it is hurting.
