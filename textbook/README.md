# The Surrounding System

### A Harness Engineering Primer

**From theory to implementation: everything you need to become the "harness guy" on every team.**

Built from primary sources including OpenAI's *Harness Engineering* (Ryan Lopopolo, Feb 2026) and Anthropic's engineering blog series on building effective agents, context engineering, tool design, evaluations, skills, and long-running agent harnesses.

---

## Table of Contents

### Part I: Foundations

| Ch | Title | Focus |
|----|-------|-------|
| [00](00-introduction.md) | **What Is Harness Engineering?** | The paradigm shift, the OpenAI experiment, why this is the most important engineering skill going forward |
| [01](01-agentic-systems-foundations.md) | **Foundations — Agentic Systems** | Agents vs workflows, the augmented LLM, five workflow patterns, the autonomy spectrum |

### Part II: Core Disciplines

| Ch | Title | Focus |
|----|-------|-------|
| [02](02-the-agent-loop.md) | **The Agent Loop** | Gather context, take action, verify work, repeat — the heartbeat of every agent system |
| [03](03-context-engineering.md) | **Context Engineering** | The attention budget, context rot, progressive disclosure, compaction, sub-agents |
| [04](04-tool-design.md) | **Designing Tools for Agents** | ACI design, tool consolidation, format design, namespacing, token efficiency |
| [05](05-architecture-and-enforcement.md) | **Architecture and Mechanical Enforcement** | Layered architecture, enforced constraints, error messages as teaching, encoding taste |

### Part III: Advanced Patterns

| Ch | Title | Focus |
|----|-------|-------|
| [06](06-long-running-agents.md) | **Long-Running Agents** | Multi-context-window work, initializer/coding agent split, progress tracking |
| [07](07-feedback-loops-and-verification.md) | **Feedback Loops and Verification** | Agent legibility, seven legibility metrics, evals, pass@k/pass^k |
| [08](08-technical-debt-and-quality.md) | **Garbage Collection of Technical Debt** | Pattern debt, golden principles, merge philosophy, the human element |
| [09](09-skills-and-mcp.md) | **Skills, MCP, and the Agent Toolkit** | Skills architecture, MCP scaling, tools-as-code-APIs, API capabilities |

### Part IV: Practice

| Ch | Title | Focus |
|----|-------|-------|
| [10](10-prompt-engineering-for-harnesses.md) | **Prompt Engineering for Harnesses** | System prompts, anti-patterns, state management, writing AGENTS.md |
| [11](11-building-your-first-harness.md) | **Building Your First Harness** | Step-by-step guide, two worked examples, reusable checklist |

---

## How to Read This

**On the plane (10 hours):** Read chapters 00-09 linearly. Take notes in `../notes/scratch.md`. Flag concepts for the knowledge base.

**After landing:** Chapters 10-11 are implementation-focused. Best read with a laptop open, ready to build.

**Estimated reading time:** ~3-4 hours for a thorough first pass. The remaining flight time is for re-reading, note-taking, and thinking.

---

## Sources

- [Harness Engineering — OpenAI](https://openai.com/index/harness-engineering/) (Ryan Lopopolo, Feb 2026)
- [Building Effective Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents) (Dec 2024)
- [Effective Harnesses for Long-Running Agents — Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) (Nov 2025)
- [Building Agents with the Claude Agent SDK — Anthropic](https://claude.com/blog/building-agents-with-the-claude-agent-sdk) (Sep 2025)
- [Writing Effective Tools for AI Agents — Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents) (Sep 2025)
- [Effective Context Engineering — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (Sep 2025)
- [Demystifying Evals for AI Agents — Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2025)
- [Agent Skills — Anthropic](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills) (2025)
- [Code Execution with MCP — Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp) (2025)
