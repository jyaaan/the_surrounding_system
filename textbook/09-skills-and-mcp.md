# Chapter 9: Skills, MCP, and the Agent Toolkit

> "An agent with a filesystem and code execution tools does not need to read the entirety of a skill into its context window. The amount of context that can be bundled is effectively unbounded."

The previous chapters focused on how to direct agents and manage the code they produce. This chapter turns to a different question: how do you *equip* agents? How do you give them specialized knowledge, connect them to external services, and scale their capabilities without drowning them in context? The answers lie in three interconnected systems: Skills, the Model Context Protocol (MCP), and a set of API-level capabilities that together form the modern agent toolkit.

---

## 9.1 Agent Skills Architecture

A skill, at its most basic, is a bundle of instructions and resources that an agent can discover and load when relevant. But "basic" undersells the design. The Skills architecture solves a genuine tension in agent systems: agents need access to vast amounts of specialized knowledge, but their context windows are finite and every token of context has a cost — both in latency and in the model's ability to attend to what matters.

The solution is **progressive disclosure**: give the agent just enough information to know a skill exists, and let the agent decide when to load more.

### Anatomy of a Skill

A skill is a directory. That is the first important fact — not a configuration entry in a database, not a record in an API, but a directory on a filesystem. This matters because it means skills are versioned with the code, reviewable in pull requests, and discoverable by agents that have filesystem access.

The directory contains, at minimum, a file named `SKILL.md`. This file has YAML frontmatter with two required fields:

```markdown
---
name: database-migrations
description: >
  Procedures and tools for creating, testing, and deploying database
  schema migrations. Covers rollback strategies, zero-downtime migration
  patterns, and data backfill approaches.
---

## Overview

This skill provides guidance for all database migration work...
```

The `name` is a short identifier. The `description` is a natural-language summary of what the skill provides — what problems it addresses, what domain knowledge it contains, what tools it offers. These two fields are the *only* thing the agent sees initially.

Beyond `SKILL.md`, the directory can contain anything: shell scripts, Python files, additional Markdown documents, configuration templates, example files, test fixtures. The structure is freeform because different skills have different needs. A "database migrations" skill might contain migration templates and a validation script. A "deploy to production" skill might contain runbooks and a deployment checklist. A "code review" skill might contain nothing but detailed written guidance.

### Progressive Disclosure in Practice

Progressive disclosure is the architectural principle that makes skills tractable at scale. It operates in three levels:

**Level 1: Name and description in the system prompt.** When an agent starts a session, it receives a list of available skills — just the names and descriptions, not the full content. This costs a small, fixed number of tokens regardless of how many skills exist or how large they are. The agent uses this index to determine which skills are *potentially relevant* to the current task.

Consider an agent that has access to thirty skills. The Level 1 index might look like:

```
Available skills:
- database-migrations: Procedures and tools for creating, testing, and deploying database schema migrations...
- api-design: Guidelines for RESTful API design including versioning, pagination, error handling...
- incident-response: Runbooks for production incidents including severity classification, escalation procedures...
- frontend-testing: Patterns for testing React components including accessibility testing, visual regression...
[... 26 more entries ...]
```

This entire index fits in perhaps 1,000 tokens. The agent now knows what capabilities are available without having loaded any of them.

**Level 2: Full SKILL.md content.** When the agent determines that a skill is relevant — because the user asked about database migrations, or because the task involves creating a new API endpoint — it loads the full `SKILL.md` file. This file might be a few hundred tokens or several thousand, depending on the skill's complexity.

The loading is *demand-driven*. The agent makes the decision to load the skill based on its assessment of relevance. This is a critical design choice. An alternative approach — loading all skills upfront — would consume the context window with information that is mostly irrelevant to any given task. Progressive disclosure ensures that context budget is spent only on knowledge the agent actually needs.

**Level 3+: Referenced files.** The `SKILL.md` file can reference additional resources:

```markdown
## Migration Templates

For standard column-addition migrations, use the template in
`./templates/add-column.sql.template`.

For data backfill migrations, see the detailed guide in
`./guides/data-backfill.md`.

## Validation

Run `./scripts/validate-migration.sh` to check your migration
for common issues before submitting.
```

The agent loads these files only if it needs them. An agent creating a simple column-addition migration loads the template. An agent performing a data backfill loads the backfill guide. An agent that just needs to validate an existing migration loads neither — it runs the script directly.

This is where the "effectively unbounded" context claim becomes concrete. A skill can reference megabytes of documentation, templates, and scripts. The agent never loads all of it. It loads what it needs, when it needs it, and discards it when it is done. The context window remains focused on the task at hand.

### Why Filesystem, Not Database

The choice to implement skills as filesystem directories rather than database records or API resources is deliberate and has several consequences:

**Version control.** Skills evolve with the codebase. When the database migration patterns change, the migration skill changes in the same commit. This means the skill is always consistent with the code it describes. Skills stored in a database are decoupled from the code and inevitably drift.

**Code review.** Changes to skills go through pull requests. A teammate can review a modification to the API design skill and say, "Actually, we decided to use cursor-based pagination, not offset-based. Let me fix this." Skills stored outside the repository do not get this scrutiny.

**Agent access.** An agent with filesystem access can discover skills by traversing the directory tree. No API client is needed, no authentication, no network call. The skill is *right there*, in the same filesystem the agent is already working in.

**Composability.** Skills can reference each other. The "deploy to production" skill can say, "Before deploying, run the migration validation from the database-migrations skill." This cross-referencing works naturally with filesystem paths.

---

## 9.2 Developing Skills

Building effective skills is an iterative process that benefits from a structured approach. The naive method — writing a comprehensive document about a topic and dropping it into a skill directory — produces skills that are technically correct but practically ineffective. The following process produces better results.

### Start with Evaluation

Before building a skill, identify the gap it will fill. The most reliable way to do this is to give an agent a set of representative tasks in the target domain and observe where it struggles.

For example, suppose you want to build a skill for your team's code review process. Give the agent ten pull requests to review — a mix of straightforward changes, subtle bugs, architectural concerns, and style issues. Note where the agent:

- Misses issues a human reviewer would catch.
- Raises false concerns about code that is actually fine.
- Lacks context about team conventions or architectural decisions.
- Produces feedback that is generic rather than specific to your codebase.

These observations become the *requirements* for the skill. If the agent consistently misses N+1 query patterns, the skill needs a section on database query review. If the agent does not know your team's error handling conventions, the skill needs those conventions spelled out. If the agent raises concerns about patterns that are actually intentional, the skill needs a section on "accepted patterns that may look unusual."

This evaluation-first approach ensures that skills address real capability gaps rather than hypothetical ones. It is tempting to write skills based on what you *think* an agent needs to know. The evaluation reveals what it *actually* needs to know, and the two often diverge.

### Structure for Scale

A common failure mode is the monolithic `SKILL.md` — a single file that grows to thousands of lines as the team adds more and more guidance. This defeats the progressive disclosure design. If the agent must load 5,000 tokens of skill content to access a 200-token section, 96% of the loaded context is noise.

Structure skills for the Level 2 / Level 3 boundary:

- `SKILL.md` contains the **overview** and **decision tree**. It tells the agent what the skill covers and helps it determine which sub-resources to load.
- Referenced documents contain the **detailed guidance**. Each document covers one specific topic or procedure.

A well-structured skill directory might look like:

```
skills/
  database-migrations/
    SKILL.md                          # Overview + decision tree
    guides/
      zero-downtime-migrations.md     # Detailed guide for ZDM patterns
      data-backfill.md                # Detailed guide for backfills
      rollback-strategies.md          # When and how to roll back
    templates/
      add-column.sql.template         # Template for column additions
      create-table.sql.template       # Template for new tables
      backfill.py.template            # Template for Python backfill scripts
    scripts/
      validate-migration.sh           # Pre-submission validation
      estimate-runtime.py             # Migration runtime estimation
    examples/
      good-migration-pr.md            # Annotated example of a well-done migration PR
      bad-migration-pr.md             # Annotated example showing common mistakes
```

The `SKILL.md` for this skill might be 300 tokens. It tells the agent: "If you are adding a column, see `guides/zero-downtime-migrations.md` and use `templates/add-column.sql.template`. If you are backfilling data, see `guides/data-backfill.md` and use `templates/backfill.py.template`. Always run `scripts/validate-migration.sh` before submitting."

The agent doing a simple column addition loads perhaps 800 tokens total (SKILL.md + the ZDM guide + the template). The agent doing a complex data backfill loads different resources. Neither loads the entire skill.

### Use Code as Documentation

A distinctive property of agent skills — one that separates them from human-oriented documentation — is that **executable code is simultaneously a tool and its own documentation**.

When a skill includes a script like `validate-migration.sh`, the agent does not just read a description of what validation to perform and then implement it from scratch. It reads the script (to understand what it does), executes the script (to perform the validation), and interprets the output (to determine next steps). The script is documentation, tool, and specification all in one artifact.

This has a practical consequence for skill development: **prefer code over prose**. Instead of writing a 500-word description of how to validate a migration, write a 50-line script that does the validation and a 50-word description of when to run it. The script is more precise than prose, cannot become inconsistent with itself (it either works or it does not), and provides the agent with an executable tool in addition to knowledge.

### Think from Claude's Perspective

This is perhaps the most counterintuitive piece of guidance: when developing skills, monitor how the agent actually uses them.

Agents do not read skills the way humans read documentation. A human reads linearly, building a mental model. An agent scans for relevant sections, extracts actionable instructions, and moves on. Information that a human finds helpful (background context, historical motivation, architectural rationale) may be noise to an agent that just needs to know *what to do*.

After deploying a skill, observe:

- **Which sections does the agent actually reference?** Sections that are never loaded can be removed or relocated to a less prominent position.
- **Where does the agent misinterpret the guidance?** Ambiguous phrasing that a human would resolve through common sense may trip an agent. Rephrase for precision.
- **What questions does the agent ask that the skill should answer?** If the agent frequently asks the user for clarification about a topic the skill covers, the coverage is either insufficient or poorly organized.

This observation process is ongoing. Skills are not written once and forgotten. They evolve as the agent's capabilities change, as the codebase evolves, and as the team's understanding of effective agent guidance deepens.

### Iterate with Claude

The most effective skill development process is collaborative: the human provides domain expertise and the agent provides feedback on what helps it perform better.

A practical workflow:

1. **Perform a task together.** Work with the agent on a representative task in the skill's domain.
2. **Note successful approaches.** When the agent does something well — uses the right pattern, asks the right clarifying question, produces code that needs no modification — note what context led to that success.
3. **Capture into the skill.** Encode the successful context into the skill. If telling the agent "always check for existing migrations that touch this table" led to better results, add that instruction.
4. **Test again.** Run the agent through similar tasks with the updated skill. Verify improvement.
5. **Repeat.** Skills converge on effectiveness through iteration, not through upfront design.

This process leverages a fundamental property of skills: they are *cheap to modify*. Changing a skill is editing a Markdown file. There is no deployment, no build step, no migration. The next agent session picks up the changes immediately. This low cost of iteration means you can refine skills aggressively without ceremony.

---

## 9.3 Model Context Protocol (MCP)

Skills equip agents with knowledge. MCP equips them with *reach*.

The Model Context Protocol is a standardized interface that connects agents to external services — Slack, GitHub, Google Drive, Asana, Jira, databases, internal APIs, and anything else an engineering team interacts with. Without MCP, connecting an agent to an external service requires custom integration code: API client libraries, authentication handling, rate limiting, error handling, response parsing. With MCP, these integrations are packaged as servers that expose a uniform tool interface.

### The Integration Problem

Consider what it takes to connect an agent to Slack without MCP. You need to:

1. Register a Slack app and obtain OAuth tokens.
2. Implement the Slack Web API client with proper authentication headers.
3. Handle token refresh when OAuth tokens expire.
4. Parse the Slack API's response format (which varies by endpoint).
5. Implement rate limiting to avoid hitting Slack's API limits.
6. Handle pagination for endpoints that return large result sets.
7. Map Slack's data model (channels, users, messages, threads) into a format the agent can work with.

Now multiply this by every service the agent needs to interact with. GitHub has a different API, different authentication, different rate limits, different pagination. Google Drive is different again. Your internal deployment system is different again. Each integration is bespoke, fragile, and must be maintained as the external APIs evolve.

MCP solves this by introducing a standard protocol between agents and external services. An MCP server wraps an external service and exposes its capabilities as tools — functions that the agent can call with structured inputs and receive structured outputs. The server handles authentication, API calls, pagination, rate limiting, and response formatting internally. The agent sees only the tool interface.

### How MCP Works

An MCP server exposes three kinds of primitives:

**Tools**: functions the agent can call. A Slack MCP server might expose tools like `send_message(channel, text)`, `list_channels()`, `search_messages(query)`, and `get_thread(channel, timestamp)`. Each tool has a name, a description, and a typed parameter schema.

**Resources**: data the agent can read. A GitHub MCP server might expose resources like `pull_request(owner, repo, number)` or `file_contents(owner, repo, path, ref)`. Resources are read-only and represent the current state of external data.

**Prompts**: pre-built prompt templates that guide the agent in using the server's capabilities effectively. A deployment MCP server might include a prompt template for "deploy to staging" that structures the agent's approach to the multi-step deployment process.

From the agent's perspective, MCP tools are indistinguishable from any other tools. The agent calls `send_message(channel="#engineering", text="Deploy complete")` the same way it calls a filesystem tool or a code execution tool. The fact that this call is being translated into a Slack API request, authenticated with OAuth, and rate-limited — all of that is invisible.

### Practical MCP Usage

The power of MCP becomes apparent when agents need to perform cross-service workflows. Consider an agent tasked with the following:

> "Review the latest PR on the backend repo, check if it affects any of the issues tagged 'high-priority' in our project board, and post a summary in the #engineering Slack channel."

Without MCP, this task requires the agent to have authentication credentials for GitHub, Jira (or whatever project management tool), and Slack, plus client code for all three APIs. With MCP, the agent has access to three MCP servers and calls their tools:

```
1. github.list_pull_requests(repo="backend", state="open", sort="created", limit=1)
2. github.get_pull_request_diff(repo="backend", number=result.number)
3. jira.search_issues(project="BACKEND", labels=["high-priority"])
4. [Agent analyzes the diff against the issue list]
5. slack.send_message(channel="#engineering", text=summary)
```

Each tool call is simple, typed, and handled by the respective MCP server. The agent focuses on the *logic* of the task — understanding the diff, matching it against issues, composing a summary — rather than on the *mechanics* of API integration.

### Running MCP Servers

MCP servers can run in several configurations:

- **Local process**: the server runs on the same machine as the agent, communicating via stdio. This is simplest and works well for development and single-user scenarios.
- **Remote server**: the server runs on a separate machine and communicates over HTTP with server-sent events (SSE) or WebSocket transport. This is appropriate for shared servers that multiple agents or users access.
- **Managed service**: some MCP servers are offered as hosted services, eliminating the need to run infrastructure entirely.

The transport is abstracted by the protocol. An agent that works with a local Slack MCP server works identically with a remote one — the tool interface is the same.

---

## 9.4 Code Execution with MCP: The Scaling Problem

MCP elegantly solves the integration problem for simple, low-volume tool calls. But when agents need to perform complex data processing tasks that involve many tool calls, large data sets, or multi-step transformations, a fundamental scaling problem emerges.

### Two Problems

**Problem 1: Tool Definition Overload.** MCP servers expose tool definitions — the name, description, and parameter schema for each available tool. These definitions are loaded into the agent's context window so it knows what tools are available. A single MCP server might expose 20-50 tools. Five MCP servers might expose 150 tools. Each tool definition consumes tokens: a typical tool with a description and a typed parameter schema costs 100-300 tokens. At scale, the tool definitions alone can consume 15,000-45,000 tokens of context — before the agent has done any actual work.

This is the tool definition overload problem: the agent's context window fills up with capabilities it *might* use, leaving less room for the context it *needs* for the current task.

**Problem 2: Intermediate Result Redundancy.** Consider an agent that needs to analyze a GitHub repository's recent activity. It might:

1. Call `github.list_pull_requests(state="merged", since="2024-01-01")` — returns 200 PRs, perhaps 50,000 tokens of data.
2. For each PR, call `github.get_pull_request_reviews(number=pr.number)` — returns review data, another 100,000+ tokens.
3. Analyze the combined data to produce a summary.

In a standard MCP interaction, all of this data flows through the model's context window. The 200 PR descriptions are loaded into context so the agent can decide what to do next. The review data for each PR is loaded into context. The agent sees everything, reasons about it, and produces a summary.

The problem: the intermediate data (the raw PR listings, the individual review records) is not what the agent needs. The agent needs the *analysis* — the patterns, the outliers, the summary statistics. But the raw data must pass through the context window to reach the model, consuming tokens that could be used for reasoning.

In tested scenarios, this pattern can consume 150,000 tokens for a task whose final output is 2,000 tokens. That is a 75:1 overhead ratio.

### The Solution: Tools as Code APIs

The solution inverts the model's relationship with MCP tools. Instead of the agent calling MCP tools directly (with results flowing through context), the agent writes code that calls MCP tools (with results processed in an execution environment before returning to context).

Here is the key insight: **present MCP tools as code APIs rather than direct function calls**.

In the standard model:

```
Agent → calls tool → result enters context → Agent reasons → calls next tool → result enters context → ...
```

In the code-execution model:

```
Agent → writes code that calls tools, processes results, returns summary → summary enters context
```

The agent writes a program like:

```python
# Fetch and analyze PR activity
prs = github.list_pull_requests(state="merged", since="2024-01-01")
review_counts = {}
for pr in prs:
    reviews = github.get_pull_request_reviews(number=pr["number"])
    review_counts[pr["author"]] = review_counts.get(pr["author"], 0) + len(reviews)

# Return only the summary
top_reviewers = sorted(review_counts.items(), key=lambda x: -x[1])[:10]
print(f"Analyzed {len(prs)} PRs")
for author, count in top_reviewers:
    print(f"  {author}: {count} reviews")
```

This code runs in a sandboxed execution environment. The MCP tool calls happen inside the environment. The raw data (150,000 tokens of PR and review data) never enters the model's context window. Only the final output — a concise summary — returns to the agent.

The token reduction is dramatic. In tested scenarios, this approach reduced token consumption from approximately 150,000 to approximately 2,000 — a **98.7% reduction**.

### Benefits Beyond Token Efficiency

The code-execution model for MCP provides advantages beyond raw token savings:

**Progressive disclosure of tool definitions.** Instead of loading all tool definitions upfront, the agent loads only the definitions it needs for the code it is writing. An agent that needs to analyze GitHub data loads GitHub tool definitions. An agent that needs to post to Slack loads Slack tool definitions. This naturally partitions the tool space and prevents definition overload.

**Context-efficient results.** The agent controls what comes back from the execution environment. It can compute aggregations, filter outliers, format summaries — all within the execution environment. The model's context window receives only the distilled output.

**Advanced control flow.** Code supports loops, conditionals, error handling, and data transformations that are awkward or impossible to express as a sequence of individual tool calls. An agent that needs to "fetch all PRs, for each PR fetch reviews, group by author, sort by count" can express this naturally in code. Expressing the same logic as a sequence of tool calls requires the model to maintain state across dozens of turns.

**Privacy preservation.** Sensitive data that passes through MCP tools (employee records, financial data, personal information) can be processed in the execution environment without ever entering the model's context. The agent writes code that queries, filters, and aggregates sensitive data, but only non-sensitive summaries return to the model. This is not just a performance benefit — it is a compliance benefit in regulated industries.

**State persistence.** Variables defined in the execution environment persist across code blocks within a session. The agent can fetch data, store it in a variable, and reference it in later code blocks without re-fetching. This is impossible with direct tool calls, where each call is stateless.

---

## 9.5 API Capabilities for Agent Builders

The Skills and MCP systems described above operate at the level of individual agent interactions. For teams building agent-powered products and systems — deploying agents at scale, integrating them into workflows, serving them to end users — a set of API-level capabilities provides the underlying infrastructure.

### Code Execution Tool

The code execution tool provides agents with access to a sandboxed Python environment. This is the same execution environment described in the MCP scaling section, but available as a general-purpose capability.

The sandbox provides:

- **A standard Python runtime** with common libraries (NumPy, pandas, requests, etc.) pre-installed.
- **Isolation**: code runs in a container that cannot access the host system, other containers, or the internet (unless explicitly configured).
- **Persistence**: variables and state persist within a session, allowing multi-step computations.
- **Output capture**: stdout, stderr, and return values are captured and returned to the agent.

The code execution tool is the foundation for the "tools as code APIs" pattern described above. It is also used for data analysis, mathematical computation, format conversion, and any other task where writing and executing code is more efficient than having the model reason about the task directly.

For agent builders, the code execution tool eliminates a significant infrastructure burden. Without it, providing agents with code execution requires spinning up sandboxed environments, managing security, handling resource limits, and capturing output — a substantial engineering project. The API provides this as a managed capability.

### MCP Connector

The MCP Connector allows API consumers to specify remote MCP server URLs directly in their API requests. This means an agent built on the API can connect to MCP servers without any local infrastructure — no local MCP server process, no stdio communication, no server management.

A typical API request with MCP might include:

```json
{
  "model": "claude-sonnet-4-20250514",
  "messages": [...],
  "mcp_servers": [
    {
      "url": "https://mcp.example.com/github",
      "auth": {"type": "bearer", "token": "..."}
    },
    {
      "url": "https://mcp.example.com/slack",
      "auth": {"type": "bearer", "token": "..."}
    }
  ]
}
```

The API handles the MCP protocol negotiation, tool discovery, and result marshaling. The agent receives the MCP tools as if they were native API tools.

This is particularly valuable for multi-tenant agent applications where different users need access to different external services. Each API request can specify different MCP servers with different authentication credentials, allowing the same agent logic to operate in different users' contexts.

### Files API

The Files API allows uploading files once and referencing them across multiple API requests. This solves a practical problem: if an agent needs to analyze a 10MB codebase or a 50-page PDF, uploading the file with every API request is wasteful and slow.

With the Files API:

1. Upload the file once, receiving a file identifier.
2. Reference the file identifier in subsequent API requests.
3. The file is available to the agent in every request that references it.

This is especially useful for agentic loops where the same file is relevant across many turns of conversation. A code review agent might upload the entire diff once and reference it across multiple analysis steps. A document analysis agent might upload a contract once and answer multiple questions about it.

The Files API also enables a clean separation between file management and conversation logic. Files can be uploaded by a different part of the system than the one conducting the conversation — for example, a webhook handler uploads new documents, and a scheduled agent process analyzes them later.

### Extended Prompt Caching

Prompt caching is a technique where the API caches the processed form of prompt prefixes, avoiding the cost of re-processing identical content across requests. Standard prompt caching has a 5-minute TTL (time-to-live) — the cache is valid for 5 minutes after the last request that used it.

Extended prompt caching increases the TTL to **1 hour** — a 12x increase. This has dramatic cost implications for agentic workloads.

Consider an agent that processes a queue of tasks, each requiring the same system prompt and skills context. With a 5-minute cache, the agent must complete each task within 5 minutes or pay the full prompt processing cost again. With a 1-hour cache, the agent can process tasks at any pace, taking breaks between tasks, and the prompt prefix remains cached.

The cost reduction can reach **up to 90%** for workloads with large, stable prompt prefixes and frequent requests. The math:

- System prompt + skills context: 50,000 tokens.
- Without caching: 50,000 input tokens processed per request.
- With extended caching: 50,000 tokens processed once per hour, then served from cache for all subsequent requests in that hour at the cached-token price (typically 90% less than uncached).

For agent builders running high-volume workloads — customer support agents, code review bots, document processing pipelines — extended prompt caching can be the difference between a viable and a prohibitively expensive deployment.

### Combining Capabilities

These API capabilities are designed to compose. A sophisticated agent deployment might use:

- **Extended prompt caching** for the system prompt and skills index (Level 1 of progressive disclosure).
- **Files API** for uploading relevant codebase files that persist across the agent's analysis steps.
- **MCP Connector** for real-time access to GitHub, Jira, and Slack.
- **Code execution** for processing MCP results efficiently (the tools-as-code pattern).

Each capability addresses a different bottleneck. Together, they provide the infrastructure for agents that are fast, cost-effective, and richly connected to the systems they need to interact with.

---

## 9.6 Designing for the Agent Runtime

The systems described in this chapter — Skills, MCP, code execution, API capabilities — are not independent features. They are components of a runtime environment designed around a specific insight: **agents are programs, and programs need runtime services**.

A traditional program runs in a runtime that provides memory management, I/O, concurrency, and standard libraries. An agent runs in a runtime that provides:

| Traditional Runtime | Agent Runtime |
|---|---|
| Memory management | Context window management (progressive disclosure, caching) |
| Standard library | Skills (reusable knowledge and tools) |
| System calls | MCP (standardized external service access) |
| Process isolation | Sandboxed code execution |
| File system | Files API (persistent data across sessions) |

This analogy is more than pedagogical. It points to a design principle: **agent capabilities should be designed as runtime services, not as ad-hoc features**. Each service should have a clean interface, composable behavior, and predictable resource costs.

The teams building the most effective agent systems are the ones that think of their agents not as chatbots with tool access, but as programs running in a purpose-built runtime. The skills are the libraries. The MCP servers are the system services. The code execution environment is the subprocess facility. The context window is the working memory, managed carefully through progressive disclosure and caching.

This shift in mental model — from "AI assistant" to "program with a sophisticated runtime" — is what separates teams that use agents for demos from teams that use agents in production.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Skills architecture | Directory-based bundles of instructions discovered and loaded progressively by agents |
| Progressive disclosure | Three-level loading: index, SKILL.md, referenced files — context is loaded on demand, not upfront |
| Skill development | Start with evaluation, structure for scale, prefer code over prose, iterate collaboratively |
| MCP | Standardized protocol wrapping external services as tools, resources, and prompts |
| Tool definition overload | Loading all MCP tool definitions upfront consumes substantial context at scale |
| Intermediate result redundancy | Raw data flowing through model context creates massive token overhead |
| Tools as code APIs | Agents write code that calls MCP tools; data is processed in execution before returning to context |
| Token reduction | Code-execution pattern achieves ~98.7% reduction in tested scenarios (150K to 2K tokens) |
| Code execution tool | Sandboxed Python with persistence, isolation, and output capture |
| MCP Connector | Remote MCP server URLs specified directly in API requests — no local infrastructure |
| Files API | Upload once, reference repeatedly across requests |
| Extended prompt caching | 1-hour TTL (12x standard), up to 90% cost reduction for high-volume workloads |
| Agent runtime | Think of agent infrastructure as a purpose-built runtime: skills as libraries, MCP as system calls, code execution as subprocess |
