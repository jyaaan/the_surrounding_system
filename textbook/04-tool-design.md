# Chapter 4: Designing Tools for Agents — The Agent-Computer Interface

> "The best tool is invisible to the person using it. The best agent tool is invisible to the agent using it — it simply does the right thing given a natural description of intent."

For decades, software engineering has operated under a single dominant assumption: the consumer of an API is another developer. Every REST convention, every SDK design guide, every function signature — all of it optimized for a human reader who understands types, reads documentation linearly, and builds mental models through experience. That assumption is now broken.

When you build a tool for an agent, you are writing software for a fundamentally different kind of consumer. The agent does not read your README top to bottom. It does not remember the last time it called your function. It does not "know" your codebase's conventions unless you surface them explicitly. It operates from a context window — a finite, precious budget of tokens — and it reasons about your tool based almost entirely on what you put in the tool's name, description, and parameter schema.

This chapter is about designing tools that make agents effective. Not tools that are technically correct. Not tools that expose maximum flexibility. Tools that produce correct behavior when wielded by a probabilistic reasoning engine under token constraints. This is a new discipline, and it requires a new vocabulary.

---

## Tools as a New Kind of Software

Let us be precise about what a "tool" is in the agent context. A tool is a callable unit — a function, an API endpoint, an MCP server method — that an agent can invoke during its reasoning loop. The agent sees the tool's schema (name, description, parameters, return type) and decides whether and how to call it based on the current task context.

This is fundamentally different from a library function or a microservice endpoint in three ways:

**1. The caller is non-deterministic.** A developer calling `getUserById(id)` knows what `id` is, knows the return type, and handles errors in predictable branches. An agent calling the same function may hallucinate an ID, misinterpret the return shape, or call it when `searchUsers(query)` would have been more appropriate. Your tool must be resilient to this.

**2. The interface is the documentation.** When a developer uses a library, the interface (function signature) and the documentation (README, docstrings, examples) are separate artifacts. For an agent, they are the same thing. The tool description *is* the documentation. The parameter schema *is* the usage guide. There is nothing else. If critical information lives only in a README that the agent hasn't ingested, that information does not exist.

**3. Token cost is a first-class constraint.** Every character in your tool's description, every parameter name, every response payload — all of it consumes tokens from the agent's context window. A verbose response that returns 500 lines when 20 would suffice doesn't just waste compute; it actively degrades the agent's ability to reason about subsequent steps because it pushes earlier context out of the window or dilutes attention.

These three properties together define what we might call the **Agent-Computer Interface (ACI)** — a term coined in analogy to the Human-Computer Interface (HCI). Just as HCI research taught us that good UI design requires understanding human cognition (Fitts's Law, Miller's Law, the Gulf of Execution), good ACI design requires understanding how language models process context, make decisions, and recover from errors.

The core thesis of this chapter: **invest as much effort in your ACI as you would in your HCI.** The payoff is the same — dramatically better outcomes from the same underlying system — but the design principles are different.

### The Contract Between Deterministic and Non-Deterministic Systems

A tool sits at the boundary between two fundamentally different computational regimes. On one side is the agent: a non-deterministic, probabilistic system that generates tool calls through autoregressive token prediction. On the other side is your backend: a deterministic system of databases, APIs, and business logic that expects precise inputs and returns structured outputs.

The tool is the **adapter** between these two regimes. It must perform two translations:

1. **Intent to precision.** The agent expresses an approximate intent ("find the customer who complained about billing last week"). The tool must translate that into a precise query your backend can execute. This means the tool often needs to be *smarter* than a simple pass-through — it needs to handle fuzzy inputs, resolve ambiguities, and fill in defaults.

2. **Data to context.** Your backend returns raw data (JSON blobs, database rows, log lines). The tool must translate that into *useful context* — information the agent can reason about in its next step. This means filtering, summarizing, and annotating the response, not just forwarding it.

This dual translation is what makes tool design a genuine engineering discipline rather than a wrapper-writing exercise.

---

## Building Effective Tools

### Start with a Prototype, Iterate with Evals

The single most common mistake in agent tool design is attempting to design the perfect tool up front. This fails for the same reason waterfall fails in product development: you cannot predict how a non-deterministic consumer will use your tool until you observe it doing so.

The correct approach follows the same **build-measure-learn** loop that drives product development, but adapted for agent consumers:

1. **Build a prototype tool.** Keep it simple. Expose the minimum functionality needed for the task. Use clear names and descriptions, but don't over-optimize them yet.

2. **Run evals.** Create a set of tasks that require the tool. Run them through your agent. Record the full transcripts — every tool call, every response, every reasoning step.

3. **Analyze failures.** When the agent fails, classify the failure mode:
   - **Wrong tool selected:** The agent chose a different tool when yours was appropriate. Your description is unclear, or there's confusing overlap with another tool.
   - **Wrong parameters:** The agent called the right tool but with incorrect arguments. Your parameter names or descriptions are ambiguous.
   - **Correct call, unusable response:** The agent called the tool correctly, but couldn't reason about the response. Your return format is too verbose, too cryptic, or missing critical context.
   - **Correct call, correct reasoning, wrong next step:** The agent used the tool fine but made a bad decision afterward. Your response may be missing information the agent needed to plan its next move.

4. **Iterate on the tool design.** Fix the most common failure mode. Re-run evals. Repeat.

This cycle typically converges within 3-5 iterations. The key insight is that you are not debugging code — you are debugging a *communication protocol* between your tool and a language model. The fixes are often in the descriptions and response formats, not in the underlying logic.

### Choosing the Right Tools: More Tools Does Not Mean Better Outcomes

There is a strong intuition — inherited from the Unix philosophy of small, composable tools — that giving an agent more tools gives it more capability. This intuition is wrong, or at least dangerously incomplete.

Every tool you add to an agent's toolkit has three costs:

1. **Description overhead.** Each tool's schema consumes tokens in the system prompt. With 50 tools, you might spend 5,000-10,000 tokens just describing the toolkit, leaving less room for actual task context.

2. **Selection complexity.** The agent must choose among all available tools at each step. More tools means more opportunities for mis-selection, especially when tools have overlapping functionality. This is the agent equivalent of Hick's Law: decision time increases with the number of options.

3. **Composition burden.** With many fine-grained tools, the agent must plan multi-step tool chains to accomplish tasks that a single well-designed tool could handle atomically. Each step in the chain is an opportunity for error to compound.

The empirical evidence is clear: **focused toolkits outperform expansive ones.** In the SWE-bench benchmark, the best-performing agent configurations used fewer than 10 tools, not hundreds.

### Consolidating Functionality

The antidote to tool sprawl is **consolidation** — combining related fine-grained operations into single, task-oriented tools. This is not the same as building monolithic tools; it is about matching the granularity of tools to the granularity of *agent intent*.

Consider these three consolidation patterns:

**Pattern 1: Workflow Consolidation**

Instead of exposing three separate tools:
```
list_users(filters) → [User]
list_events(user_id, date_range) → [Event]
create_event(user_id, title, time, attendees) → Event
```

Provide one higher-level tool:
```
schedule_event(description: str, constraints: str) → ScheduleResult
```

The consolidated tool internally resolves user references, checks calendar availability, and creates the event. The agent expresses intent ("schedule a 1:1 with Sarah next Tuesday afternoon") and the tool handles the orchestration.

Why this works: the agent doesn't need to understand the three-step workflow. It doesn't need to know that users must be looked up before events can be queried. It doesn't need to handle the error case where a user ID doesn't match any events. All of that complexity is absorbed by the tool.

**Pattern 2: Query Consolidation**

Instead of exposing a raw log reader:
```
read_logs(file: str, start_line: int, end_line: int) → [str]
```

Provide a semantic search tool:
```
search_logs(query: str, time_range: str, severity: str) → SearchResult
```

Where `SearchResult` includes matching lines, surrounding context, and metadata (timestamps, source files, request IDs). The agent doesn't need to know which log file to read or which line numbers are relevant — it expresses what it's looking for, and the tool does the finding.

**Pattern 3: Context Aggregation**

Instead of three separate data-fetching tools:
```
get_customer_by_id(id) → Customer
list_transactions(customer_id, date_range) → [Transaction]
list_notes(customer_id) → [Note]
```

Provide one context-gathering tool:
```
get_customer_context(identifier: str, include: list[str]) → CustomerContext
```

Where `identifier` can be an ID, email, or name (the tool resolves it), and `include` specifies which facets of context to retrieve (`["transactions", "notes", "support_tickets"]`). The response is a pre-assembled briefing that gives the agent everything it needs to reason about the customer.

### Each Tool Needs a Clear, Distinct Purpose

The consolidation principle leads directly to a design rule: **every tool in your toolkit should occupy a unique niche in the agent's decision space.** If two tools have overlapping functionality, the agent will sometimes pick the wrong one — and "sometimes" at scale means "frequently."

Test this by asking yourself: *Could a reasonable person, reading only the tool names and descriptions, confuse these two tools?* If yes, consolidate or differentiate.

A useful heuristic is the **one-sentence test**: you should be able to describe each tool's purpose in a single sentence that does not overlap with any other tool's sentence. If your sentence includes "or" — as in "use this tool to read files or search for patterns" — you likely have two tools masquerading as one.

---

## Tool Format Design

The format of tool inputs and outputs has an outsized impact on agent performance. The reason is subtle and important: language models generate output *autoregressively*, one token at a time, left to right. Each token conditions all subsequent tokens. This means the *order* in which information appears in a tool's output, and the *format* in which the agent must express its input, directly affect the quality of reasoning.

### Give the Model Enough Tokens to "Think"

Consider a tool that requires the agent to specify a complex edit. If the tool's input format requires the target location *first* and the edit content *second*, the agent must decide where to edit before articulating what to edit. But often, the act of formulating the edit *is what clarifies* where it should go.

This is an instance of a general principle: **design tool input formats so that the agent can reason its way into the answer, not guess it up front.** Place open-ended, generative fields (like descriptions, rationales, or search queries) before constrained, precise fields (like file paths, line numbers, or enum selections) wherever possible.

Some tool frameworks support a "thinking" or "scratchpad" parameter — a free-text field that the agent fills in before the structured parameters. This is not a gimmick; it meaningfully improves accuracy because it gives the model token-space to reason before committing to a decision.

### Keep Format Close to Naturally Occurring Text

Language models are trained on natural language and code. They are most accurate when tool inputs and outputs resemble text they have seen during training. This has practical implications:

- **Prefer prose descriptions over abstract encodings.** A tool parameter described as `"action": "MOVE_FILE"` is fine. A parameter described as `"action_code": 7` (where 7 means "move file" per some internal enum) forces the agent to maintain a mental lookup table, and it will sometimes get the mapping wrong.

- **Prefer code-like formats for code-related tools.** If your tool operates on source code, accept and return actual code strings, not AST node references or line-number ranges. The agent reasons about code as text; force it to reason about code as data structures and accuracy drops.

- **Prefer markdown or plain text for explanatory content.** If your tool returns a report or summary, format it as readable text with headers and bullet points, not as a nested JSON structure that the agent must mentally parse.

### Minimize Formatting Overhead

Every bit of formatting that the agent must handle — counting lines, escaping strings, managing indentation, tracking cursor positions — is an opportunity for error. Consider two approaches to a file-editing tool:

**High overhead (fragile):**
```json
{
  "file": "src/app.py",
  "line_start": 42,
  "line_end": 47,
  "replacement": "    def process(self, data):\n        validated = self.validate(data)\n        return self.transform(validated)\n"
}
```

The agent must correctly count line numbers, manage string escaping for newlines, and get indentation right inside a JSON string. Each of these is a common failure mode.

**Low overhead (robust):**
```json
{
  "file_path": "/absolute/path/to/src/app.py",
  "old_string": "    def process(self, data):\n        return self.transform(data)",
  "new_string": "    def process(self, data):\n        validated = self.validate(data)\n        return self.transform(validated)"
}
```

The search-and-replace format eliminates line counting entirely. The agent only needs to reproduce the existing code (which it can copy from context) and specify the replacement. This format dramatically reduces edit errors in practice.

### Absolute Paths vs. Relative Paths: A Lesson from SWE-bench

One of the most instructive lessons from competitive coding benchmarks is the impact of path formats on agent accuracy. Early SWE-bench configurations used relative file paths, and agents frequently produced errors after changing directories — they would refer to `src/utils.py` when the working directory had shifted and the correct reference was `../../src/utils.py` or `/project/src/utils.py`.

Switching to **absolute paths** eliminated this entire class of errors. The lesson generalizes: **prefer canonical, unambiguous identifiers over context-dependent ones.** An absolute path is a canonical identifier for a file. A relative path is context-dependent — its meaning changes based on the current working directory, which is invisible state that the agent must track across tool calls.

This principle applies beyond file paths:
- Use full resource URIs instead of short names that might collide.
- Use ISO 8601 timestamps instead of relative time expressions ("2 hours ago").
- Use fully qualified function names (`module.submodule.function`) instead of bare names that might be ambiguous.

---

## Namespacing

In production systems, an agent may have access to tools from dozens of sources: internal APIs, third-party MCP servers, built-in framework tools, and custom task-specific tools. Without namespacing, you quickly run into collisions and confusion. Two different MCP servers might both expose a `search` tool. Three different services might each provide a `get_status` tool.

**Namespacing** solves this by prefixing tool names with a source or domain identifier, following the same pattern as package namespacing in programming languages:

```
github_search_issues
github_create_pr
github_list_reviews

jira_search_tickets
jira_create_ticket
jira_update_status

database_query
database_explain_plan

logs_search
logs_tail
logs_get_context
```

The prefix accomplishes three things:

1. **Disambiguation.** `github_search_issues` and `jira_search_tickets` are clearly different tools, even though both involve searching.

2. **Grouping.** The agent can reason about tools as belonging to clusters. When it needs to interact with GitHub, it knows to look at `github_*` tools. This is analogous to how developers navigate APIs by package or module.

3. **Boundary delineation.** Namespaces make it clear which tools belong to which system, which helps the agent reason about data flow. A result from `github_search_issues` can be passed to `github_create_pr` but probably not directly to `jira_create_ticket` without transformation.

When designing namespace conventions, prefer short, recognizable prefixes (2-15 characters) that match the domain vocabulary your agent is likely to have encountered in training data. `github_` is better than `gh_` (more recognizable) and better than `github_dot_com_api_v3_` (unnecessarily verbose).

For MCP servers specifically, the server name itself often serves as the namespace. When registering multiple MCP servers with an agent framework, ensure their tool names don't collide, or add explicit prefixes during registration.

---

## Returning Meaningful Context

The response your tool returns is not just data — it is **context** that shapes the agent's next reasoning step. A response that dumps raw database rows forces the agent to parse, filter, and interpret. A response that provides pre-interpreted, high-signal context lets the agent reason at a higher level of abstraction.

### High Signal Over Flexibility

The traditional API design principle of returning raw data and letting the consumer interpret it does not transfer well to agent tools. Agents are better served by **opinionated, interpreted responses** than by flexible, raw ones.

Consider a deployment status tool. A raw response might look like:

```json
{
  "deployment_id": "d-3f8a2b",
  "status": "IN_PROGRESS",
  "started_at": "2026-03-22T14:30:00Z",
  "steps": [
    {"name": "build", "status": "COMPLETED", "duration_ms": 45000},
    {"name": "test", "status": "COMPLETED", "duration_ms": 120000},
    {"name": "deploy-staging", "status": "IN_PROGRESS", "started_at": "2026-03-22T14:32:45Z"},
    {"name": "deploy-prod", "status": "PENDING"}
  ],
  "triggered_by": "user-a1b2c3",
  "commit": "abc123def456"
}
```

A high-signal response might look like:

```
Deployment d-3f8a2b is currently deploying to staging (step 3 of 4).
Build and tests completed successfully (2m 45s total).
Staging deployment started 5 minutes ago and is still in progress.
Production deployment is pending and will begin after staging succeeds.

Triggered by: Alice Chen (alice@company.com)
Commit: abc123d "Fix payment retry logic for expired cards"
Branch: fix/payment-retry → main
```

The second response is dramatically more useful to an agent. It resolves the user ID to a name. It converts the commit hash to include the message. It calculates elapsed time. It describes the state in natural language. The agent can immediately reason about this response — it doesn't need to mentally parse JSON, look up user IDs, or calculate time deltas.

### Semantic Names Over Cryptic Identifiers

This principle deserves special emphasis because it has a measurable impact on agent accuracy. When a tool response includes identifiers — user IDs, resource ARNs, enum codes — the agent must either remember what they refer to or look them up in a subsequent tool call. Both options are error-prone.

**Resolve identifiers to human-readable names at the tool level.** Instead of returning `"assigned_to": "usr_7f3a2b"`, return `"assigned_to": "Sarah Martinez (usr_7f3a2b)"`. Include the ID for cases where the agent needs to pass it to another tool, but lead with the name so the agent can reason about the *meaning* without a lookup.

This practice — resolving UUIDs and opaque identifiers to natural language — has been shown to reduce hallucination rates in multi-step agent workflows. The mechanism is straightforward: when the agent sees `"usr_7f3a2b"` and needs to reference that user later, it may generate a plausible but incorrect ID. When it sees `"Sarah Martinez"`, it anchors to a meaningful referent that it's less likely to confuse.

### The `response_format` Pattern

Not every tool call needs the same level of detail. An agent that's scanning for a specific error in logs needs concise, targeted results. An agent that's diagnosing a complex issue needs detailed context with surrounding lines and metadata.

The **response_format enum** pattern addresses this by accepting a parameter that controls output verbosity:

```
search_logs(
  query: str,
  time_range: str,
  response_format: "concise" | "detailed" = "concise"
)
```

- **`"concise"`** returns matching lines with minimal context — just enough for the agent to decide if a result is relevant. This is the default because most tool calls are exploratory.
- **`"detailed"`** returns matching lines with surrounding context, metadata (timestamps, request IDs, source files), and potentially related entries. This is for deep investigation.

This pattern lets the agent self-regulate its token consumption. A well-designed agent will start with concise queries to orient itself, then drill down with detailed queries on specific results. The tool supports this workflow explicitly.

### Token Efficiency: Pagination, Range Selection, Filtering, and Truncation

Agent context windows are large but not infinite. A tool that returns 10,000 lines of log output is not being helpful — it is flooding the agent's context and degrading its ability to reason. Token efficiency is not a nice-to-have; it is a correctness concern.

**Pagination.** For tools that might return large result sets, implement cursor-based pagination and return a manageable page size by default (10-50 results, depending on result size). Include a `next_cursor` field that the agent can pass back to get the next page. Crucially, include a `total_count` or `has_more` indicator so the agent can decide whether to paginate further.

**Range selection.** For tools that operate on sequential data (files, logs, time series), accept range parameters (`start_line`/`end_line`, `start_time`/`end_time`) and default to a sensible recent window rather than returning everything.

**Filtering.** Build filtering into the tool rather than relying on the agent to filter results post-hoc. A `search_logs(severity="ERROR")` that returns 5 results is more useful than a `read_logs()` that returns 500 lines for the agent to scan through.

**Truncation with sensible defaults.** When a response would exceed a reasonable size, truncate it and include a clear indicator: `"[... 347 more results. Use pagination or add filters to narrow results.]"` This tells the agent both that results were truncated and how to get more.

A useful rule of thumb: **no single tool response should exceed 20% of the agent's effective context window.** If it does, the tool is pushing out other context that the agent likely needs. Design your defaults to stay well under this threshold.

### Helpful Error Responses

When a tool call fails, the error response is the agent's only guide for recovery. A bad error response leads to retries with the same bad input, or abandonment of a viable approach. A good error response leads to a corrected call.

**Bad error response:**
```json
{"error": "Invalid input"}
```

The agent has no idea what was invalid. It will likely retry with slightly different parameters, fail again, and eventually give up or hallucinate a workaround.

**Good error response:**
```json
{
  "error": "Parameter 'date_range' format is invalid. Expected ISO 8601 interval (e.g., '2026-03-01/2026-03-22'). Received: 'last week'. Use 'relative_time' parameter instead for relative time expressions like 'last week' or 'past 3 hours'."
}
```

This response tells the agent:
1. *Which* parameter was wrong.
2. *What* format was expected, with an example.
3. *What* was actually received.
4. *How* to accomplish the original intent using the correct parameter.

Design your error responses as **remediation guides**, not diagnostic codes. The agent doesn't need to look up error code `E_4012` in a manual — it needs to know what to do differently on the next call.

When possible, include **suggestions** in error responses:

```json
{
  "error": "User 'john.smith' not found.",
  "suggestions": [
    "Did you mean 'john.smith2' (John Smith, Engineering)?",
    "Did you mean 'jonathan.smith' (Jonathan Smith, Product)?"
  ],
  "hint": "Use search_users(query='john smith') to find users by name."
}
```

This pattern of fuzzy matching and alternative suggestions mirrors good CLI design (think `git`'s "Did you mean...?" messages) and is highly effective for agent recovery.

---

## Prompt-Engineering Tool Descriptions

The tool description is the single most important piece of text you write when building agent tools. It is the agent's *entire understanding* of what the tool does, when to use it, and how to use it correctly. A poorly written description can make an excellent tool useless; a well-written description can make a simple tool powerful.

### Think of Describing Tools to a New Hire

The right mental model is this: imagine you're explaining the tool to a new engineer on their first day. They're smart, they understand general concepts, but they don't know your system's specifics. They don't know your naming conventions, your query syntax, your edge cases, or your implicit assumptions.

This means:

**Make implicit context explicit.** If your tool uses a custom query language, describe the syntax in the tool description, not in a separate document. If certain parameter values have non-obvious effects ("setting `mode` to `dry_run` will validate the operation without executing it"), state that explicitly.

**Explain when to use the tool, not just what it does.** "Searches the audit log" is less useful than "Use this tool when you need to find who made a specific change to a resource, or to trace the sequence of operations that led to a particular state. For real-time log monitoring, use `logs_tail` instead."

**Document edge cases and gotchas.** "The `owner` field accepts either a username or team name. Team names must be prefixed with `team/` (e.g., `team/platform`). If no owner is specified, the tool defaults to the authenticated user."

**Provide examples for complex parameters.** If your tool accepts a query string with special syntax, include 2-3 examples in the description:
```
query: Search expression using Lucene syntax.
Examples:
  - "status:error AND service:payments" (find payment errors)
  - "user.email:*@company.com" (find events from company users)
  - "timestamp:[2026-03-20 TO 2026-03-22]" (date range)
```

### Specialized Query Formats, Niche Terminology, and Resource Relationships

Three categories of implicit knowledge cause the most agent errors when left undocumented:

**1. Query formats.** If your tool uses JQL, Lucene, SQL, GraphQL, regex, glob patterns, or any other query language, describe the relevant subset in the tool description. You don't need to document the entire language — just the parts most commonly needed, with examples.

**2. Niche terminology.** Every organization has domain-specific terms. If your tool operates on "workspaces" (which are actually team-scoped project containers), say so. If "deployment" in your system means something different from the common understanding (maybe it includes provisioning and configuration), clarify. The agent's training data has given it a general understanding of common terms; your description corrects for domain-specific meanings.

**3. Resource relationships.** If resources have parent-child relationships, ownership semantics, or cross-references, document them. "A `pipeline` belongs to a `project`, which belongs to an `organization`. You must specify the `project_id` to list pipelines. Use `list_projects` to find the project ID if you only know the project name."

### Real Impact: The SWE-bench Lesson

The impact of tool descriptions on agent performance is not theoretical. When Anthropic's Claude Sonnet 3.5 was evaluated on SWE-bench (a benchmark of real-world GitHub issues requiring code changes), achieving state-of-the-art performance required iterating extensively on tool descriptions.

The tools themselves — file reading, editing, searching, running tests — were straightforward. The difference between mediocre and state-of-the-art performance came down to how those tools were described. Precise descriptions of edge cases, clear guidance on when to use which tool, and explicit documentation of input formats turned a capable model into an effective coding agent.

This is not a coincidence unique to SWE-bench. It reflects a general principle: **the ceiling on agent performance is often set by tool description quality, not model capability.** A more capable model with vague tool descriptions will underperform a less capable model with precise ones. Investing in descriptions is the highest-leverage intervention available to tool designers.

---

## Writing Tools with LLMs

There is a satisfying circularity in using LLMs to build tools for LLMs. Claude Code and similar agent-assisted development tools are excellent at prototyping agent tools because they understand — implicitly, through their training — what makes a tool description clear and what response formats are easy to reason about.

### Use Claude Code to Prototype Tools

When building a new agent tool, start by describing the tool's purpose and desired behavior to Claude Code. Let it generate the initial implementation, including:

- The tool's schema (name, description, parameters)
- Input validation and error handling
- Response formatting
- Description text

Then *read the generated description critically*. The model will often produce descriptions that are clear to another model — which is exactly what you want. But it may also make assumptions about context that won't be available at runtime. Edit for precision.

### Generate Evaluation Tasks from Real-World Use Cases

The most common failure in agent tool evaluation is testing with artificial, simplistic tasks. A tool that works perfectly for "search for the word 'error' in the logs" may fail badly for "figure out why the payment processing latency spiked at 3am" — which is the actual use case.

Generate evaluation tasks from **real-world scenarios** your agents will face:

1. **Mine support tickets, incident reports, and Slack threads** for the kinds of questions people actually ask. These become your eval tasks.

2. **Record actual agent sessions** during early deployment. Extract the tasks that succeeded and the tasks that failed. The failures become your improvement targets; the successes become your regression tests.

3. **Ask domain experts** to describe their hardest recent tasks in natural language. These descriptions become eval prompts.

4. **Vary the phrasing.** A tool should work whether the agent says "find the error," "search for failures," or "look for what went wrong." Generate paraphrases of each eval task to test robustness.

Avoid the trap of testing only the "happy path." Your evals should include:
- Ambiguous queries that could match multiple tools
- Edge cases in parameter formats (empty strings, very long inputs, special characters)
- Tasks that require the agent to call the tool multiple times with refining queries
- Tasks where the tool should *not* be used (to test that the agent doesn't over-rely on it)

### Run Evals Programmatically with Simple Agentic Loops

You don't need a sophisticated evaluation framework to test agent tools. A simple loop suffices:

```python
for task in eval_tasks:
    transcript = run_agent(
        system_prompt=SYSTEM_PROMPT,
        tools=toolkit,
        user_message=task.prompt,
        max_turns=15
    )
    result = evaluate_transcript(transcript, task.expected_outcome)
    results.append(result)
```

The key components are:

- **`run_agent`**: Invokes your agent with the given tools and prompt, recording the full transcript of reasoning steps and tool calls.
- **`evaluate_transcript`**: Checks whether the agent achieved the expected outcome. This can be automated (did the agent produce the correct file edit? did it find the right log entry?) or semi-automated (did it call the right tools in a reasonable order?).
- **`max_turns`**: A bound on the number of reasoning steps. If the agent hasn't succeeded in 15 turns, it's likely stuck. This prevents runaway eval costs.

Run evals after every significant change to a tool — new parameters, modified descriptions, restructured responses. Track metrics over time: success rate, average turns to completion, average token consumption. Watch for regressions.

### Analyze Transcripts: What Agents Omit Is Often More Important Than What They Include

When reviewing agent transcripts from evaluations, most people focus on what the agent did wrong — the incorrect tool call, the hallucinated parameter, the confused reasoning. This is useful but incomplete.

**Pay close attention to what the agent *didn't* do.** Common patterns:

- **The agent didn't use your tool at all.** It solved the problem through reasoning alone, or used a different tool. This might mean your tool isn't needed, or its description doesn't make clear when it's useful.

- **The agent didn't use an available parameter.** Your tool accepts a `severity` filter, but the agent searched for "ERROR" in the query string instead. This means the parameter's purpose isn't clear from the description, or the agent doesn't realize it exists.

- **The agent didn't paginate.** It got the first page of results, reasoned about them, and moved on — missing the relevant result on page 3. This means your pagination metadata isn't prominent enough, or the first-page results didn't signal that more existed.

- **The agent didn't recover from an error.** It got an error response, tried once more with slightly different parameters, then gave up. Your error response didn't provide enough guidance for recovery.

These omissions are invisible in aggregate metrics (success/failure rates) but reveal deep insights about tool design when studied individually. Build a habit of reading 5-10 transcripts per eval run, focusing not just on what the agent did but on what it *could have done and didn't*.

---

## Summary

Designing tools for agents is a new discipline at the intersection of API design, technical writing, and cognitive science. The core principles are:

1. **Consolidate over proliferate.** Fewer, task-oriented tools outperform many fine-grained ones.
2. **The description is the documentation.** Invest heavily in tool names, descriptions, and parameter documentation.
3. **Return context, not data.** Pre-interpret responses. Resolve identifiers. Include enough information for the next reasoning step.
4. **Minimize formatting overhead.** Keep inputs and outputs close to natural text. Eliminate unnecessary precision requirements (line numbers, string escaping).
5. **Use canonical identifiers.** Absolute paths, full URIs, ISO timestamps — anything that removes context-dependency.
6. **Design error messages for recovery.** Tell the agent what went wrong, what format was expected, and how to fix it.
7. **Iterate with evals, not intuition.** Build, measure, learn. Read transcripts. Fix the communication, not just the code.

The ACI is young — we are at the equivalent of early GUI design, still discovering which patterns work and which fail. But the principles in this chapter have been validated through extensive empirical work on real-world agent systems, from SWE-bench to production deployments. They represent the current best understanding of how to build tools that make agents effective.

The next chapter addresses the other side of the equation: how to structure the codebase and enforce architectural constraints so that agents — and the tools they wield — operate within boundaries that ensure correctness at scale.

---

## Annotated Bibliography

**[1]** Anthropic. "Writing Effective Tools for AI Agents." *Anthropic Engineering Blog*, September 11, 2025. https://www.anthropic.com/engineering/writing-tools-for-agents

> The primary reference for this chapter's treatment of tool design principles. Covers the Agent-Computer Interface concept, tool consolidation patterns, response formatting, error message design, and the iterative eval-driven approach to tool development. The source of the empirical findings on SWE-bench tool description impact discussed in this chapter.

**[2]** Anthropic. "Building Effective Agents." *Anthropic Engineering Blog*, December 19, 2024. https://www.anthropic.com/engineering/building-effective-agents

> Provides foundational context for how agents select and invoke tools within agentic loops. Relevant to this chapter's discussion of tool selection complexity, the costs of expansive toolkits, and the importance of designing tools that fit naturally into an agent's reasoning cycle.

**[3]** Anthropic. "Prompting Best Practices — Claude 4.6." *Claude Documentation*, 2025. https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices

> Covers prompting techniques that directly inform the chapter's guidance on writing tool descriptions, including how to structure parameter documentation, provide examples, and handle edge cases in tool schemas. Relevant to the sections on prompt-engineering tool descriptions and the "new hire" mental model.

**[4]** Anthropic. "Code Execution with MCP: Building More Efficient Agents." *Anthropic Engineering Blog*, 2025. https://www.anthropic.com/engineering/code-execution-with-mcp

> Discusses the Model Context Protocol as a mechanism for exposing tools to agents, directly relevant to the chapter's coverage of namespacing MCP server tools, tool schema design for MCP endpoints, and the architectural considerations for tools that execute code on behalf of agents.

**[5]** Anthropic. "Demystifying Evals for AI Agents." *Anthropic Engineering Blog*, 2025. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

> Provides the methodological framework for the chapter's emphasis on iterating tool designs through evaluation. Relevant to the sections on running evals programmatically, analyzing agent transcripts, and the build-measure-learn cycle for tool development.
