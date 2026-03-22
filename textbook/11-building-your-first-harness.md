# Chapter 11: Building Your First Harness — A Practical Guide

The previous chapters gave you the theory. This chapter gives you the practice. We will walk through building a complete agent harness from scratch, step by step, with real file contents, real configurations, and real code examples. By the end of this chapter, you will have a repeatable process for setting up a harness in any domain.

We will build the harness generically first, then apply it to two concrete examples: a full-stack web application and a research agent. The generic process is the skeleton; the examples show how to put flesh on the bones.

---

## Planning Your Harness

Before writing a single file, you need to answer four questions. These answers drive every subsequent decision.

### 1. What Is the Task Domain?

The task domain determines everything: what tools the agent needs, what verification looks like, what failure modes to watch for, and how to measure success.

Common task domains:

| Domain | Primary Tools | Primary Verification | Key Failure Mode |
|--------|--------------|---------------------|-----------------|
| Full-stack web development | Code editor, terminal, browser | Tests, visual review, type checking | Architectural violations, UI drift |
| Backend API development | Code editor, terminal, HTTP client | Tests, contract validation | Data integrity, performance regression |
| Data pipeline engineering | Code editor, terminal, database client | Data validation, pipeline tests | Silent data corruption |
| Research and analysis | Web search, document reader, note-taking | Source verification, logical consistency | Hallucination, source fabrication |
| Customer support | Knowledge base, ticket system, CRM | Resolution rate, customer satisfaction | Incorrect information, policy violations |
| DevOps / Infrastructure | Terraform, kubectl, cloud CLIs | Infrastructure tests, drift detection | Configuration errors, security gaps |

Identify your domain. If it spans multiple domains (e.g., "full-stack web with infrastructure"), pick the primary one and treat the secondary domain as an extension.

### 2. Define the Agent Loop

Every agent system follows a loop. Define yours explicitly:

```
1. Read the task specification
2. Plan the approach (break into subtasks if needed)
3. For each subtask:
   a. Implement the change
   b. Verify the change (run tests, lint, typecheck)
   c. Fix any issues found during verification
   d. Commit the working change
4. Run the full verification suite
5. Report completion
```

Your agent loop might be simpler or more complex, but it must be explicit. The loop becomes part of your system prompt — the agent needs to know its own workflow.

### 3. Identify Required Tools

Start with the minimum set. You can always add tools later, but every unnecessary tool is a distraction. List the tools and, for each one, write a one-sentence justification:

```
- File read/write: Required to read and modify source code
- Terminal execution: Required to run build, test, and lint commands
- Git operations: Required to commit changes and manage branches
- Web browser (Playwright): Required to visually verify UI changes
```

If you cannot write a one-sentence justification for a tool, you probably do not need it yet.

### 4. Define Success Criteria

What does "done" look like? Define it in measurable terms:

```
A task is complete when:
1. All acceptance criteria from the task specification are met
2. All tests pass (unit, integration, and e2e)
3. Linter reports zero errors
4. Type checker reports zero errors
5. The change is committed with a descriptive message
6. The progress file is updated with a summary of what was done
```

These success criteria become part of the agent's instructions, so they need to be precise and verifiable — the agent must be able to check each criterion programmatically.

---

## Step 1: Repository Structure

The repository structure is the skeleton of your harness. It tells both human engineers and agents where things live and how they relate to each other.

### Create the Directory Structure

```bash
mkdir -p docs/design-docs docs/exec-plans docs/product-specs docs/references
touch AGENTS.md ARCHITECTURE.md
touch docs/DESIGN.md docs/FRONTEND.md docs/SECURITY.md
```

The resulting structure:

```
project-root/
  AGENTS.md                    # Agent entry point (~100 lines)
  ARCHITECTURE.md              # Full architecture description
  docs/
    DESIGN.md                  # Design principles and patterns
    FRONTEND.md                # Frontend-specific guidance
    SECURITY.md                # Security requirements
    design-docs/               # One doc per major feature
    exec-plans/                # Project execution plans
    product-specs/             # Product requirements
    references/                # External documentation
  src/                         # Source code (structure varies by project)
  tests/                       # Test code (mirrors src/ structure)
```

### Write the Initial AGENTS.md

Start with this template and customize it for your project:

```markdown
# AGENTS.md
<!-- Last verified: 2026-03-22 -->

## Project Overview
[Project name] is a [brief description] built with [technology stack].
Source code is in `src/`. Tests are in `tests/`.

## Key Commands
- `[build command]` — Build the project
- `[test command]` — Run all tests
- `[lint command]` — Run linter and formatter
- `[typecheck command]` — Run type checker (if applicable)
- `[dev command]` — Start development server (if applicable)

## Architecture
See ARCHITECTURE.md for details. Key rules:
- [Rule 1: e.g., "Layered architecture: handlers -> services -> repositories"]
- [Rule 2: e.g., "Dependencies flow downward only"]
- [Rule 3: e.g., "All external calls go through adapter interfaces"]

## Common Tasks
### [Most common task, e.g., "Adding a new API endpoint"]
1. [Step 1]
2. [Step 2]
3. [Step 3]
4. Run `[verification command]`

### [Second most common task]
1. [Step 1]
2. [Step 2]
3. Run `[verification command]`

## Documentation
- `ARCHITECTURE.md` — System architecture and dependency rules
- `docs/DESIGN.md` — Design principles and patterns
- `docs/SECURITY.md` — Security requirements
- `docs/design-docs/` — Feature design documents

## Constraints
- [Constraint 1: e.g., "Never modify applied migrations"]
- [Constraint 2: e.g., "Never bypass authentication"]
- [Constraint 3: e.g., "Never log PII or secrets"]
- [Constraint 4: e.g., "Always run tests before committing"]
```

Keep it under 100 lines. Every line should earn its place.

### Write ARCHITECTURE.md

The architecture document is where you describe the system in depth. Unlike `AGENTS.md`, this file can be long — several hundred lines if needed. It should cover:

```markdown
# Architecture

## System Overview
TaskFlow is a three-tier web application serving project management
functionality through a REST API.

## Layered Architecture

```
┌─────────────────────────────────────────┐
│           API Layer (Routers)            │
│        src/api/                         │
│    Handles HTTP, validation, auth       │
├─────────────────────────────────────────┤
│         Service Layer                    │
│        src/services/                    │
│    Business logic, orchestration        │
├─────────────────────────────────────────┤
│        Repository Layer                  │
│        src/repositories/                │
│    Database access, query building      │
├─────────────────────────────────────────┤
│          Model Layer                     │
│        src/models/                      │
│    Data models, schema definitions      │
└─────────────────────────────────────────┘
```

## Dependency Rules
Dependencies flow DOWNWARD only:
- Routers may import from: services, models
- Services may import from: repositories, models
- Repositories may import from: models
- Models may not import from any other layer

Violations of these rules will be caught by the architectural linter
(see `scripts/check_architecture.py`).

## Module Responsibilities
### API Layer (`src/api/`)
- HTTP request/response handling
- Input validation using Pydantic models
- Authentication and authorization checks
- Error formatting

[... continue for each layer ...]

## Data Flow
### Read Path
1. Client sends HTTP request
2. Router validates input and checks auth
3. Router calls service method
4. Service applies business rules
5. Service calls repository method
6. Repository executes database query
7. Response flows back up through the layers

### Write Path
[... similar description ...]

## Key Design Decisions
### Why FastAPI over Django
[Rationale with tradeoffs]

### Why Repository Pattern
[Rationale with tradeoffs]
```

This document serves as the agent's architectural memory. When an agent is deciding where to put new code, it reads this document to understand the system's structure and rules.

---

## Step 2: Mechanical Enforcement

Documentation tells agents what to do. Mechanical enforcement *makes* them do it. This is the most important principle in harness engineering: **every rule that can be enforced mechanically, should be.**

### Architectural Linting

Write a script that validates your dependency rules:

```python
#!/usr/bin/env python3
"""scripts/check_architecture.py - Verify architectural dependency rules."""

import ast
import sys
from pathlib import Path

# Define the allowed import relationships
LAYER_ORDER = ["api", "services", "repositories", "models"]

ALLOWED_IMPORTS = {
    "api": {"services", "models"},
    "services": {"repositories", "models"},
    "repositories": {"models"},
    "models": set(),  # models may not import from other layers
}


def get_layer(filepath: Path) -> str | None:
    """Determine which architectural layer a file belongs to."""
    parts = filepath.parts
    if "src" not in parts:
        return None
    src_index = parts.index("src")
    if src_index + 1 < len(parts):
        layer = parts[src_index + 1]
        if layer in LAYER_ORDER:
            return layer
    return None


def get_imports(filepath: Path) -> list[str]:
    """Extract all import targets from a Python file."""
    try:
        tree = ast.parse(filepath.read_text())
    except SyntaxError:
        return []

    imports = []
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                imports.append(alias.name)
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                imports.append(node.module)
    return imports


def check_file(filepath: Path) -> list[str]:
    """Check a single file for architectural violations."""
    source_layer = get_layer(filepath)
    if source_layer is None:
        return []

    violations = []
    for imp in get_imports(filepath):
        # Check if the import references another layer
        for layer in LAYER_ORDER:
            if f"src.{layer}" in imp or imp.startswith(layer):
                if layer not in ALLOWED_IMPORTS[source_layer]:
                    violations.append(
                        f"{filepath}:{source_layer} imports from "
                        f"{layer}, but {source_layer} -> {layer} "
                        f"is not allowed. Allowed targets: "
                        f"{ALLOWED_IMPORTS[source_layer]}"
                    )
    return violations


def main() -> int:
    """Check all Python files in src/ for violations."""
    src_dir = Path("src")
    if not src_dir.exists():
        print("No src/ directory found.")
        return 1

    all_violations = []
    for filepath in src_dir.rglob("*.py"):
        all_violations.extend(check_file(filepath))

    if all_violations:
        print("ARCHITECTURAL VIOLATIONS FOUND:\n")
        for v in all_violations:
            print(f"  ✗ {v}")
        print(f"\n{len(all_violations)} violation(s) found.")
        print(
            "\nTo fix: move the import to the correct layer, or use "
            "dependency injection to invert the dependency."
        )
        return 1

    print("✓ No architectural violations found.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Notice the error message at the bottom: "To fix: move the import to the correct layer, or use dependency injection to invert the dependency." **Remediation-focused error messages** are critical. An error message that only says "violation found" forces the agent to figure out how to fix it. An error message that suggests a fix lets the agent act immediately.

### Structural Tests

Write tests that verify your architecture holds:

```python
# tests/test_architecture.py
"""Structural tests for architectural invariants."""

import importlib
import pkgutil
from pathlib import Path

import pytest


def get_all_modules(package_path: str) -> list[str]:
    """Get all module names under a package path."""
    modules = []
    package = importlib.import_module(package_path)
    for importer, modname, ispkg in pkgutil.walk_packages(
        package.__path__, prefix=package.__name__ + "."
    ):
        modules.append(modname)
    return modules


class TestArchitecture:
    """Verify architectural constraints are maintained."""

    def test_models_have_no_layer_imports(self):
        """Models should not import from any other layer."""
        model_files = Path("src/models").rglob("*.py")
        for filepath in model_files:
            content = filepath.read_text()
            for layer in ["api", "services", "repositories"]:
                assert f"from src.{layer}" not in content, (
                    f"{filepath} imports from {layer}. "
                    f"Models must not depend on other layers."
                )

    def test_repositories_only_import_models(self):
        """Repositories should only import from models."""
        repo_files = Path("src/repositories").rglob("*.py")
        for filepath in repo_files:
            content = filepath.read_text()
            for layer in ["api", "services"]:
                assert f"from src.{layer}" not in content, (
                    f"{filepath} imports from {layer}. "
                    f"Repositories may only import from models."
                )

    def test_every_router_has_tests(self):
        """Every router module should have a corresponding test module."""
        router_files = set(
            f.stem for f in Path("src/api").glob("*.py")
            if f.stem != "__init__"
        )
        test_files = set(
            f.stem.replace("test_", "")
            for f in Path("tests/api").glob("test_*.py")
        )
        missing = router_files - test_files
        assert not missing, (
            f"Routers without tests: {missing}. "
            f"Create test files in tests/api/ for each router."
        )

    def test_no_direct_sql_in_services(self):
        """Services should not contain raw SQL — use repositories instead."""
        service_files = Path("src/services").rglob("*.py")
        sql_keywords = ["SELECT ", "INSERT ", "UPDATE ", "DELETE ", "DROP "]
        for filepath in service_files:
            content = filepath.read_text()
            for keyword in sql_keywords:
                # Allow SQL keywords in comments and strings used for logging
                lines = content.split("\n")
                for i, line in enumerate(lines, 1):
                    stripped = line.strip()
                    if stripped.startswith("#"):
                        continue
                    if keyword in line.upper() and "execute" in line.lower():
                        pytest.fail(
                            f"{filepath}:{i} contains raw SQL. "
                            f"Move database queries to a repository method."
                        )
```

These tests do double duty: they enforce architecture for human developers and for agents. When an agent runs `make test` and sees a structural test fail with a remediation-focused message, it knows exactly what to fix and how to fix it.

### CI Configuration

Wire everything into CI so violations can never be merged:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff mypy
      - run: ruff check src/ tests/
      - run: ruff format --check src/ tests/

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt mypy
      - run: mypy src/ --strict

  architecture:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python scripts/check_architecture.py

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest tests/ -v --tb=short
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
```

The CI configuration has four separate jobs: lint, typecheck, architecture, and test. Each can fail independently, and each provides targeted feedback. When the architecture check fails, the agent sees architecture-specific error messages, not a wall of unrelated test output.

### Remediation-Focused Error Messages

This principle is worth emphasizing with one more example. Compare these two error messages:

```
# Bad: diagnosis without remediation
ERROR: Circular dependency detected between auth_service and user_service.

# Good: diagnosis with remediation
ERROR: Circular dependency detected between auth_service and user_service.

  auth_service imports user_service (src/services/auth_service.py:7)
  user_service imports auth_service (src/services/user_service.py:12)

To fix this circular dependency, consider one of these approaches:
  1. Extract the shared functionality into a new service (e.g., identity_service)
  2. Use dependency injection: pass the needed function as a parameter
  3. Move the shared logic to a utility module in src/utils/

See docs/DESIGN.md#circular-dependencies for examples of each approach.
```

The good version tells the agent *where* the problem is (specific files and line numbers), *what* the problem is (circular dependency), and *how to fix it* (three concrete approaches with a link to examples). An agent reading this message can take action immediately.

---

## Step 3: Tool Design

Tools are how agents interact with the world. Bad tools create confusion; good tools create capability.

### Identify the Minimal Set

Start by listing every action your agent needs to take, then group them into the smallest set of tools that covers all actions:

```
Actions needed:
- Read source files         → File read tool
- Write/modify source files → File edit tool
- Run shell commands        → Terminal tool
- Search for code patterns  → Code search tool (grep)
- Find files by name        → File search tool (glob)
- Manage git state          → Git operations (via terminal)
- View UI in browser        → Browser tool (Playwright MCP)
```

Seven tools for a full-stack development agent. Not three, not twenty — seven. Each tool has a clear, non-overlapping purpose.

### Design Tool Descriptions Like Onboarding Docs

Write each tool's description as if you were explaining it to a new hire on their first day:

```json
{
  "name": "run_terminal_command",
  "description": "Execute a shell command and return its output (stdout and stderr). Use this for running tests (pytest, jest), linting (ruff, eslint), building (make, npm run build), and any other CLI operations. Commands run in the project root directory. Long-running commands (servers, watchers) should be run with the background flag. Commands that modify files (rm, mv) should be used carefully — prefer the file edit tool for modifying source code.",
  "parameters": {
    "command": {
      "type": "string",
      "description": "The shell command to execute. Use absolute paths for clarity."
    },
    "timeout_ms": {
      "type": "integer",
      "description": "Maximum execution time in milliseconds. Default: 120000 (2 minutes). Increase for slow operations like full test suites."
    },
    "background": {
      "type": "boolean",
      "description": "Run in background if true. Use for servers, watchers, and other long-running processes. You'll be notified when the command completes."
    }
  }
}
```

Notice how the description explains not just *what* the tool does, but *when* to use it (tests, linting, building), *how* it behaves (runs in project root), and *what to watch out for* (prefer file edit for source modifications). This is the level of detail a new hire needs. It is the level of detail an agent needs too.

### No Overlap Between Tools

Every action should have exactly one obvious tool. If an agent has both a "write file" tool and an "edit file" tool, it needs to decide which one to use for every modification. This decision point is a source of errors.

Define clear boundaries:

```
- "File read": Use to read any file. Never modifies files.
- "File edit": Use to modify existing files. Sends only the diff.
  Preferred for all source code changes.
- "File write": Use to create new files or completely rewrite existing
  files. Use file edit for partial modifications.
```

### Test Tool Usage with Simple Eval Loops

Before using tools in complex workflows, verify that the agent uses them correctly with simple test cases:

```python
# eval/test_tool_usage.py
"""Simple eval cases to verify correct tool usage."""

EVAL_CASES = [
    {
        "task": "Read the contents of src/models/user.py",
        "expected_tool": "file_read",
        "expected_args_contain": {"file_path": "src/models/user.py"},
    },
    {
        "task": "Add a 'phone_number' field to the User model",
        "expected_tool": "file_edit",
        "expected_args_contain": {"file_path": "src/models/user.py"},
        "should_not_use": "file_write",  # Should edit, not rewrite
    },
    {
        "task": "Run the test suite",
        "expected_tool": "run_terminal_command",
        "expected_args_contain": {"command": "pytest"},
    },
    {
        "task": "Find all files that import the User model",
        "expected_tool": "code_search",
        "expected_args_contain": {"pattern": "import.*User"},
    },
]
```

These evals catch tool design problems early. If the agent consistently uses the wrong tool for a task, the tool descriptions need clarification.

---

## Step 4: Feedback Loops

Feedback loops are the mechanism by which agents self-correct. Without feedback, agents drift. With good feedback, they converge on correct solutions.

### Linting and Formatting: The First Feedback Loop

Linting is the cheapest, fastest feedback loop. Set it up first and make it mandatory:

```makefile
# Makefile
.PHONY: lint format typecheck test

lint:
	ruff check src/ tests/ --output-format=concise
	ruff format --check src/ tests/

format:
	ruff format src/ tests/
	ruff check --fix src/ tests/

typecheck:
	mypy src/ --strict --pretty

test:
	pytest tests/ -v --tb=short

check: lint typecheck test
	@echo "All checks passed."
```

The `check` target runs everything in order: lint first (fastest, catches formatting issues), typecheck second (medium speed, catches type errors), tests last (slowest, catches logic errors). This ordering is deliberate — fail fast on cheap checks before running expensive ones.

Instruct the agent to run `make lint` after every file modification:

```
After every code change, run `make lint`. Fix any issues before proceeding.
Do not accumulate lint errors — fix them immediately. Lint errors compound:
a formatting issue in one file can cause confusing errors when that file is
imported elsewhere.
```

### Type Checking

Static type checking catches a class of errors that linting misses. For typed languages (TypeScript, Python with type hints, Rust, Go), type checking is the second feedback loop:

```
After implementing changes, run `make typecheck`. Type errors often reveal
design problems:
- If you are fighting the type system, your design may need rethinking
- If you need many type: ignore comments, your interfaces may need adjustment
- Missing type annotations should be added, not suppressed
```

### Visual Verification

For UI work, visual verification catches the gap between "code that runs without errors" and "UI that looks correct." Set up a browser automation tool like Playwright MCP:

```
After implementing any visual change:
1. Run `make dev` to start the development server
2. Use the browser tool to navigate to the affected page
3. Take a screenshot and compare it to the design spec
4. Check for:
   - Layout correctness (spacing, alignment, responsive behavior)
   - Color accuracy (compare hex values to the design system)
   - Typography (font family, size, weight, line height)
   - Interactive states (hover, focus, active, disabled)
5. If the implementation does not match the spec, iterate
```

### Create a Basic Eval Suite

An eval suite is a collection of tasks with known correct outcomes. It is how you measure whether your harness is actually working.

Start with 20-50 tasks drawn from real scenarios:

```python
# eval/suite.py
"""Evaluation suite for the development agent harness."""

EVAL_TASKS = [
    # Category: Simple modifications
    {
        "id": "simple-001",
        "description": "Add a 'created_at' timestamp field to the Project model",
        "setup": "git checkout eval-baseline",
        "verification": [
            "grep -q 'created_at' src/models/project.py",
            "make test",
            "make typecheck",
        ],
        "expected_files_changed": [
            "src/models/project.py",
            "tests/models/test_project.py",
        ],
    },
    {
        "id": "simple-002",
        "description": "Add input validation to the create_user endpoint: "
                        "username must be 3-50 characters, alphanumeric only",
        "setup": "git checkout eval-baseline",
        "verification": [
            "make test",
            "make lint",
            # Test that validation actually works
            "python -c \"from src.api.users import CreateUserRequest; "
            "import pytest; "
            "pytest.raises(ValueError, CreateUserRequest, username='ab')\"",
        ],
    },
    # Category: Multi-file changes
    {
        "id": "multi-001",
        "description": "Add a new 'tags' feature: projects can have tags, "
                        "tags are stored in a separate table with a many-to-many "
                        "relationship, add CRUD endpoints for tags",
        "setup": "git checkout eval-baseline",
        "verification": [
            "make check",  # lint + typecheck + test
            "python scripts/check_architecture.py",
            # Verify all layers were created
            "test -f src/models/tag.py",
            "test -f src/repositories/tag_repository.py",
            "test -f src/services/tag_service.py",
            "test -f src/api/tags.py",
            "test -f tests/api/test_tags.py",
        ],
    },
    # Category: Bug fixes
    {
        "id": "bug-001",
        "description": "Users report that updating a project name to an empty "
                        "string succeeds but causes errors later. Fix this.",
        "setup": "git checkout eval-bug-001",  # branch with the bug
        "verification": [
            "make test",
            # Verify the fix handles the edge case
            "pytest tests/api/test_projects.py::test_update_empty_name -v",
        ],
    },
    # ... 16-46 more tasks covering your domain's scenarios
]
```

**Draw tasks from real scenarios.** Do not invent artificial tasks. Go through your team's recent pull requests and extract the task descriptions. Real tasks expose real failure modes; artificial tasks test artificial capabilities.

**Target 70% pass rate initially.** A new harness that passes 70% of eval tasks on the first run is doing well. Below 50% suggests fundamental issues with tool design or system prompts. Above 90% suggests your eval tasks are too easy — add harder ones.

---

## Step 5: Long-Running Agent Support

Complex tasks span multiple context windows. Long-running agent support is the infrastructure that makes this possible.

### Create an Initializer Prompt

The initializer prompt is the first message in every context window. It loads state from previous sessions and orients the agent:

```xml
<initializer>
You are a senior engineer working on the {{project_name}} project.

<project_context>
{{contents of AGENTS.md}}
</project_context>

<current_state>
<progress>
{{contents of progress.txt, or "No previous progress — this is a new task."}}
</progress>

<tasks>
{{contents of tasks.json}}
</tasks>

<recent_changes>
{{output of: git log --oneline -20}}
</recent_changes>
</current_state>

<instructions>
1. Read the progress file and task list to understand the current state
2. Run `make test` to verify all existing work is functional
3. Identify the next pending task from tasks.json
4. Implement the task following the architecture in ARCHITECTURE.md
5. After completing the task:
   a. Run `make check` (lint + typecheck + test)
   b. Commit the changes with a descriptive message
   c. Update tasks.json to mark the task as complete
   d. Update progress.txt with a summary of what you did
6. If you encounter a blocker, document it in progress.txt and move to
   the next task if possible
</instructions>
</initializer>
```

The template variables (`{{...}}`) are filled in by your harness runner — a script that reads the current state files and injects their contents into the prompt.

### Create a Coding Agent Prompt

The coding agent prompt is the system prompt that persists throughout the session. It sets the agent's role, capabilities, and behavioral constraints:

```xml
<system>
<role>
You are a senior software engineer with expertise in {{tech_stack}}.
You are methodical: you read code before modifying it, you run tests
after every change, and you commit working code frequently.
</role>

<workflow>
For each task:
1. Read the relevant source files to understand the current implementation
2. Plan your changes before making them
3. Implement changes incrementally — one logical change at a time
4. After each change, run `make lint` to catch formatting issues
5. After completing the task, run `make check` for full verification
6. Commit with a conventional commit message: type(scope): description
</workflow>

<architecture>
{{contents of ARCHITECTURE.md}}
</architecture>

<constraints>
- Follow the dependency rules in the architecture document strictly
- Never modify migration files that have been applied
- Always add tests for new functionality
- Keep changes focused — do not refactor unrelated code
- If you are unsure about a design decision, document the options in
  progress.txt and proceed with the simplest option
</constraints>

<state_management>
- `tasks.json` — Track task completion. Update status after each task.
- `progress.txt` — Append notes about what you did, decisions made,
  and any issues encountered. Never delete previous entries.
- Git commits — Commit after each completed task. Use conventional commits.
</state_management>
</system>
```

### Set Up the Progress File and Feature List

Create the initial state files:

```bash
# progress.txt
cat > progress.txt << 'EOF'
# Progress Log

## Session 1 — Initial Setup
- Project initialized with base harness structure
- AGENTS.md and ARCHITECTURE.md created
- CI pipeline configured
- Eval suite scaffolded

Next steps: Begin implementing features from tasks.json
EOF
```

```bash
# tasks.json
cat > tasks.json << 'EOF'
{
  "project": "TaskFlow",
  "tasks": [
    {
      "id": 1,
      "description": "Set up the database models for User and Project",
      "status": "pending",
      "priority": "high",
      "depends_on": [],
      "acceptance_criteria": [
        "User model with id, username, email, created_at fields",
        "Project model with id, name, description, owner_id, created_at",
        "Foreign key relationship between Project.owner_id and User.id",
        "Alembic migration generated and applied",
        "Unit tests for model creation and relationships"
      ]
    },
    {
      "id": 2,
      "description": "Implement CRUD endpoints for Users",
      "status": "pending",
      "priority": "high",
      "depends_on": [1],
      "acceptance_criteria": [
        "POST /users — create a user",
        "GET /users/{id} — get user by ID",
        "PUT /users/{id} — update user",
        "DELETE /users/{id} — soft delete user",
        "Input validation on all endpoints",
        "Integration tests for each endpoint"
      ]
    }
  ]
}
EOF
```

### Configure Git Workflows for Clean Handoffs

Each agent session should leave the repository in a clean state for the next session:

```
At the end of your session (or when you have completed all available tasks):
1. Ensure all changes are committed — no uncommitted work
2. Run `make check` one final time to verify everything passes
3. Update progress.txt with a session summary including:
   - What tasks were completed
   - Any decisions or tradeoffs made
   - What the next session should start with
4. Update tasks.json with current task statuses
5. Commit the state file updates: `git commit -m "chore: update progress and task state"`
```

This protocol ensures that when a new context window starts and reads the state files, it gets an accurate picture of the project's status.

---

## Step 6: Garbage Collection

Over time, harness configurations accumulate cruft: prompting instructions that address long-fixed issues, overly specific rules that should have been generalized, documentation that has drifted from reality. Garbage collection is the practice of actively cleaning this up.

### Define Your Golden Principles

Before you can clean, you need to know what "clean" looks like. Define 3-5 golden principles that guide your harness:

```markdown
## Golden Principles

1. **Mechanical enforcement over documentation**: If a rule can be enforced
   by a linter, test, or CI check, it should be. Documentation is for
   context and judgment calls.

2. **Minimalism**: Every instruction, tool, and constraint must justify its
   existence. When in doubt, remove it.

3. **Remediation over diagnosis**: Error messages should tell the agent how
   to fix the problem, not just what the problem is.

4. **Verification before progress**: The agent must verify its work before
   moving to the next task. Building on unverified work is forbidden.

5. **State in files, not in memory**: All state that needs to survive across
   context windows must be persisted to files.
```

### Set Up Recurring Cleanup Tasks

Schedule periodic reviews of your harness components:

**Weekly**: Review agent transcripts from the past week. Look for:
- Tasks where the agent got stuck or looped
- Instructions that the agent misinterpreted
- Tools that the agent used incorrectly

**Monthly**: Review and prune the `AGENTS.md` file. Look for:
- Instructions that address issues that no longer occur
- Commands that have changed
- Architectural rules that have been superseded

**Quarterly**: Review the eval suite. Look for:
- Tasks that are now trivially easy (remove or replace with harder variants)
- Missing coverage for new features or task types
- Eval tasks that test for obsolete patterns

### Track Quality Metrics Per Domain

Measure what matters:

```python
# eval/metrics.py
"""Track harness quality metrics over time."""

METRICS = {
    "eval_pass_rate": {
        "description": "Percentage of eval tasks the agent completes correctly",
        "target": 0.85,
        "measurement": "Run full eval suite, count passes / total",
    },
    "avg_task_turns": {
        "description": "Average number of agent turns to complete a task",
        "target": 15,
        "measurement": "Count tool calls per eval task, average across suite",
    },
    "architecture_violations": {
        "description": "Number of architecture violations in agent-written code",
        "target": 0,
        "measurement": "Run architecture linter on agent output",
    },
    "human_intervention_rate": {
        "description": "Percentage of tasks requiring human intervention",
        "target": 0.10,
        "measurement": "Track how often humans need to correct agent work",
    },
    "context_window_efficiency": {
        "description": "Percentage of context window used for productive work "
                        "vs overhead (loading state, re-reading files, etc.)",
        "target": 0.70,
        "measurement": "Analyze token usage in agent transcripts",
    },
}
```

These metrics tell you whether your harness is improving. A rising eval pass rate means your prompts and tools are getting better. A falling average task turns means the agent is becoming more efficient. An increasing human intervention rate is an alarm bell.

---

## Step 7: Iterate

Building a harness is not a one-time activity. It is a continuous improvement process driven by observation, diagnosis, and targeted fixes.

### Read Transcripts of Agent Runs

This is the single highest-value activity in harness maintenance. Reading transcripts reveals:

- **What the agent thinks it is being told to do** (often different from what you intended)
- **Where it hesitates** (multiple tool calls trying different approaches)
- **Where it fails** (wrong tool selection, incorrect reasoning, hallucination)
- **Where it succeeds effortlessly** (these paths are working — do not touch them)

Set aside time to read at least 2-3 full transcripts per week. Not summaries — full transcripts. The detail matters.

### Identify Where Agents Get Stuck

Common stuck patterns:

| Pattern | Symptom | Fix |
|---------|---------|-----|
| Looping | Agent repeats the same action 3+ times | Add explicit instructions for what to do when an approach fails |
| Wrong tool | Agent uses file write when it should use file edit | Clarify tool descriptions, add negative examples |
| Missing context | Agent does not know about a relevant file or convention | Add a pointer in AGENTS.md |
| Conflicting instructions | Agent oscillates between two behaviors | Remove one of the conflicting rules |
| Over-planning | Agent spends many turns planning before acting | Add "prefer action over analysis" guidance |

### Encode Fixes Systematically

When you identify a failure mode, you have three options for fixing it. Use the most mechanical option available:

1. **Linter rule** (most mechanical): If the failure produces code that can be statically detected, write a linter rule. Example: "agent creates files outside the src/ directory" becomes a linter rule that checks file locations.

2. **Better tool description** (medium mechanical): If the failure is a tool misuse, update the tool description. Example: "agent uses file_write for small edits" becomes a tool description clarification.

3. **Documentation update** (least mechanical): If the failure requires judgment, update AGENTS.md or a referenced doc. Example: "agent does not know about our caching conventions" becomes a section in docs/DESIGN.md.

Always prefer option 1 over 2, and option 2 over 3. Mechanical enforcement is more reliable than documentation because it provides immediate, unambiguous feedback.

### Measure Improvement Through Evals

After making a change, re-run your eval suite and compare results:

```bash
# Run evals and save results with timestamp
python eval/run.py --output eval/results/$(date +%Y%m%d_%H%M%S).json

# Compare with previous run
python eval/compare.py eval/results/latest.json eval/results/previous.json
```

```
Eval Comparison: 2026-03-22 vs 2026-03-15

Overall pass rate: 78% → 83% (+5%)

Improved:
  multi-001: FAIL → PASS (architecture linter now catches layer violations)
  bug-003:   FAIL → PASS (better error message guides fix)

Regressed:
  simple-004: PASS → FAIL (new lint rule is too strict, investigate)

Unchanged failures:
  multi-003: FAIL (agent still struggles with multi-table migrations)
```

This feedback loop — observe, diagnose, fix, measure — is the engine of harness improvement. Each cycle makes the harness slightly more effective, and these improvements compound over time.

---

## Example: Building a Harness for a Full-Stack Web App

Let us apply everything above to a concrete scenario. We are building a harness for TaskFlow, a project management web application with a FastAPI backend and a React frontend.

### Planning

**Domain**: Full-stack web development
**Agent loop**: Read task, plan, implement backend, implement frontend, verify (tests + visual), commit
**Tools**: File read, file edit, file write, terminal, code search, file search, Playwright MCP
**Success criteria**: Tests pass, linter clean, typecheck clean, visual match with design spec

### Repository Structure

```
taskflow/
  AGENTS.md
  ARCHITECTURE.md
  Makefile
  docs/
    DESIGN.md
    FRONTEND.md
    SECURITY.md
    design-docs/
      user-authentication.md
      project-management.md
    references/
      api-spec.yaml
  backend/
    src/
      api/
      services/
      repositories/
      models/
    tests/
    requirements.txt
    pyproject.toml
  frontend/
    src/
      components/
      pages/
      hooks/
      services/
      types/
    tests/
    package.json
    tsconfig.json
  scripts/
    check_architecture.py
  progress.txt
  tasks.json
```

### AGENTS.md

```markdown
# AGENTS.md — TaskFlow
<!-- Last verified: 2026-03-22 -->

## Overview
TaskFlow is a project management app. Backend: Python 3.12, FastAPI,
PostgreSQL, SQLAlchemy. Frontend: React 18, TypeScript, Tailwind CSS.

## Key Commands
- `make dev-backend` — Start backend dev server (port 8000)
- `make dev-frontend` — Start frontend dev server (port 3000)
- `make test` — Run all tests (backend + frontend)
- `make test-backend` — Run backend tests only
- `make test-frontend` — Run frontend tests only
- `make lint` — Lint all code
- `make typecheck` — Type check all code
- `make check` — Run lint + typecheck + test

## Architecture
See ARCHITECTURE.md. Key rules:
- Backend: layered (api -> services -> repositories -> models)
- Frontend: pages use hooks, hooks use services, services call API
- Backend and frontend share types via `shared/types/`
- No direct DB access from services — always go through repositories
- No business logic in API layer or frontend components

## Adding a Backend Endpoint
1. Add/update model in `backend/src/models/`
2. Add/update repository in `backend/src/repositories/`
3. Add/update service in `backend/src/services/`
4. Add/update router in `backend/src/api/`
5. Add tests in `backend/tests/` mirroring source structure
6. Run `make check`

## Adding a Frontend Page
1. Create page component in `frontend/src/pages/`
2. Create sub-components in `frontend/src/components/`
3. Create hooks for data fetching in `frontend/src/hooks/`
4. Add API service methods in `frontend/src/services/`
5. Add route in `frontend/src/App.tsx`
6. Run `make check`
7. Visually verify with Playwright

## Docs
- `ARCHITECTURE.md` — Full architecture
- `docs/DESIGN.md` — Design patterns
- `docs/FRONTEND.md` — Frontend conventions
- `docs/SECURITY.md` — Auth, authz, data handling

## Constraints
- Never modify applied migrations
- Never bypass auth checks
- Never log PII or secrets
- Always run tests before committing
- Always visually verify UI changes
```

### Makefile

```makefile
.PHONY: dev-backend dev-frontend test lint typecheck check

dev-backend:
	cd backend && uvicorn src.main:app --reload --port 8000

dev-frontend:
	cd frontend && npm run dev

test-backend:
	cd backend && pytest tests/ -v --tb=short

test-frontend:
	cd frontend && npm test -- --watchAll=false

test: test-backend test-frontend

lint-backend:
	cd backend && ruff check src/ tests/ && ruff format --check src/ tests/

lint-frontend:
	cd frontend && npx eslint src/ --max-warnings=0

lint: lint-backend lint-frontend

typecheck-backend:
	cd backend && mypy src/ --strict

typecheck-frontend:
	cd frontend && npx tsc --noEmit

typecheck: typecheck-backend typecheck-frontend

check: lint typecheck test
	@echo "All checks passed."
```

### Architectural Linter (Backend)

The Python architecture checker from Step 2 handles the backend. For the frontend, add an ESLint rule:

```javascript
// frontend/.eslintrc.js
module.exports = {
  // ... other config
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['../../backend/*', '../../../backend/*'],
          message: 'Frontend cannot import from backend. Use shared/types/ for shared types.'
        },
        {
          group: ['../pages/*'],
          message: 'Components cannot import from pages. Extract shared logic to hooks/ or utils/.'
        }
      ]
    }]
  }
};
```

### Sample Eval Task

```python
{
    "id": "fullstack-001",
    "description": "Add a 'due date' field to projects. Backend: add the field "
                    "to the model, update CRUD endpoints, add validation that "
                    "due date must be in the future. Frontend: add a date picker "
                    "to the project creation form, display due date on the "
                    "project detail page.",
    "setup": "git checkout eval-baseline",
    "verification": [
        "make check",
        "python scripts/check_architecture.py",
        "grep -q 'due_date' backend/src/models/project.py",
        "grep -q 'due_date' frontend/src/types/project.ts",
        "cd frontend && npx playwright test tests/e2e/project-due-date.spec.ts",
    ],
    "expected_layers_touched": {
        "backend": ["models", "repositories", "services", "api"],
        "frontend": ["types", "components", "pages", "services"],
    },
}
```

---

## Example: Building a Harness for a Research Agent

Not all harnesses are about code. Let us build one for a research agent that investigates topics, synthesizes findings, and produces structured reports.

### Planning

**Domain**: Research and analysis
**Agent loop**: Receive question, search for sources, read and evaluate sources, synthesize findings, write report, verify claims
**Tools**: Web search, document reader, note-taking (file write), citation manager
**Success criteria**: All claims sourced, no fabricated citations, logical consistency, meets formatting requirements

### Repository Structure

```
research-harness/
  AGENTS.md
  docs/
    STYLE_GUIDE.md
    METHODOLOGY.md
    references/
  output/
    reports/
    notes/
  templates/
    report-template.md
    source-evaluation.md
  sources.json
  progress.txt
  tasks.json
```

### AGENTS.md

```markdown
# AGENTS.md — Research Agent
<!-- Last verified: 2026-03-22 -->

## Overview
Research agent that investigates topics and produces sourced reports.
All output goes in `output/`. Sources tracked in `sources.json`.

## Workflow
1. Receive research question
2. Break question into sub-questions
3. For each sub-question:
   a. Search for 3-5 relevant sources
   b. Read and evaluate each source (use templates/source-evaluation.md)
   c. Record findings in output/notes/
   d. Add sources to sources.json
4. Synthesize findings into a report using templates/report-template.md
5. Verify all claims have citations
6. Save report to output/reports/

## Source Evaluation
See docs/METHODOLOGY.md for source evaluation criteria. Key rules:
- Every factual claim must cite a specific source
- Sources must be evaluated for credibility before use
- Prefer primary sources over secondary sources
- Prefer recent sources (last 2 years) unless historical context needed
- Never fabricate or assume sources — if you cannot find a source, say so

## Output Format
Use templates/report-template.md for all reports. Key sections:
- Executive Summary (3-5 sentences)
- Methodology (how sources were found and evaluated)
- Findings (organized by sub-question)
- Limitations (what was not covered and why)
- Sources (full citation list with URLs)

## Constraints
- NEVER fabricate citations or sources
- NEVER present unverified claims as facts
- ALWAYS note confidence level (high/medium/low) for each finding
- ALWAYS disclose when sources conflict
- Mark speculative analysis clearly as "Analysis" vs "Finding"
```

### Source Tracking (sources.json)

```json
{
  "sources": [
    {
      "id": "src-001",
      "title": "Example Research Paper",
      "url": "https://example.com/paper",
      "type": "academic_paper",
      "credibility": "high",
      "date_accessed": "2026-03-22",
      "publication_date": "2025-11-15",
      "evaluation": "Peer-reviewed, published in reputable journal, methodology sound",
      "used_in_claims": ["finding-1", "finding-3"]
    }
  ]
}
```

### Source Evaluation Template

```markdown
# Source Evaluation

## Source
- **Title**: [title]
- **URL**: [url]
- **Type**: [academic_paper | news_article | industry_report | blog_post | official_documentation]
- **Publication Date**: [date]

## Credibility Assessment
- **Author credentials**: [assessment]
- **Publication reputation**: [assessment]
- **Methodology** (if applicable): [assessment]
- **Potential biases**: [assessment]
- **Overall credibility**: [high | medium | low]

## Key Claims Extracted
1. [claim] — [quote from source]
2. [claim] — [quote from source]

## Relevance to Research Question
[How this source addresses the research question]
```

### Report Template

```markdown
# [Report Title]

*Research Date: [date]*
*Confidence Level: [high | medium | low]*

## Executive Summary
[3-5 sentence summary of key findings]

## Methodology
- **Research question**: [the original question]
- **Sub-questions investigated**: [list]
- **Sources consulted**: [count] sources across [types]
- **Date range**: Sources from [earliest] to [latest]

## Findings

### [Sub-question 1]
**Confidence: [high/medium/low]**

[Finding text with inline citations like [src-001]]

### [Sub-question 2]
...

## Conflicting Evidence
[Document where sources disagree and why]

## Limitations
- [What was not investigated and why]
- [Known gaps in available sources]
- [Potential biases in the source selection]

## Sources
[Full citation list, generated from sources.json]
```

### Anti-Hallucination Verification

The research domain's biggest risk is hallucination — the agent inventing sources or claims. Build verification into the harness:

```python
# scripts/verify_citations.py
"""Verify that all citations in a report correspond to real sources."""

import json
import re
import sys
from pathlib import Path


def verify_report(report_path: str, sources_path: str = "sources.json") -> int:
    report = Path(report_path).read_text()
    sources = json.loads(Path(sources_path).read_text())

    # Extract all citation references from the report
    citations = set(re.findall(r'\[src-(\d+)\]', report))

    # Get all source IDs
    source_ids = {s["id"].replace("src-", "") for s in sources["sources"]}

    # Check for citations that reference non-existent sources
    missing = citations - source_ids
    if missing:
        print(f"ERROR: Report references non-existent sources: "
              f"{['src-' + m for m in missing]}")
        print("Fix: Add these sources to sources.json or remove the citations.")
        return 1

    # Check for uncited claims (heuristic: sentences with factual language
    # but no citation)
    lines = report.split('\n')
    uncited_claims = []
    factual_indicators = [
        'according to', 'research shows', 'studies indicate',
        'data suggests', 'evidence demonstrates', 'statistics show',
        'reports indicate', 'surveys found'
    ]
    for i, line in enumerate(lines, 1):
        line_lower = line.lower()
        if any(indicator in line_lower for indicator in factual_indicators):
            if not re.search(r'\[src-\d+\]', line):
                uncited_claims.append((i, line.strip()))

    if uncited_claims:
        print("WARNING: Possible uncited factual claims:")
        for line_num, text in uncited_claims:
            print(f"  Line {line_num}: {text}")
        print("Fix: Add citations or rephrase as analysis/opinion.")

    if not missing:
        print("All citations verified.")
    return 0


if __name__ == "__main__":
    sys.exit(verify_report(sys.argv[1]))
```

This script is a mechanical enforcement tool for the research domain, analogous to the architectural linter for code. It catches the most dangerous failure mode — fabricated citations — automatically.

---

## Checklist

Use this checklist when setting up any new harness. It covers all seven steps and applies to any domain.

### Planning
- [ ] Task domain identified
- [ ] Agent loop defined and documented
- [ ] Minimal tool set identified with justifications
- [ ] Success criteria defined in measurable terms

### Repository Structure
- [ ] `AGENTS.md` created (~100 lines, table of contents style)
- [ ] `ARCHITECTURE.md` created (comprehensive system description)
- [ ] `docs/` directory created with appropriate subdirectories
- [ ] `docs/DESIGN.md` — design principles and patterns
- [ ] `docs/SECURITY.md` — security requirements (if applicable)
- [ ] Domain-specific docs created (e.g., `docs/FRONTEND.md`)
- [ ] Templates directory created (if applicable)

### Mechanical Enforcement
- [ ] Linting configured and running in CI
- [ ] Type checking configured (if applicable) and running in CI
- [ ] Architectural linter written and running in CI
- [ ] Structural tests written for dependency validation
- [ ] All error messages include remediation guidance
- [ ] CI blocks merges on any violation

### Tool Design
- [ ] Each tool has a clear, non-overlapping purpose
- [ ] Tool descriptions are written at new-hire level of detail
- [ ] Tool descriptions include when-to-use and when-not-to-use guidance
- [ ] Basic tool usage verified through simple eval cases

### Feedback Loops
- [ ] Linting runs after every file modification
- [ ] Type checking runs after implementation changes
- [ ] Full test suite runs before commits
- [ ] Visual verification set up (if applicable)
- [ ] Eval suite created with 20-50 tasks from real scenarios
- [ ] Eval pass rate measured and baselined

### Long-Running Agent Support
- [ ] Initializer prompt template created
- [ ] Coding/working agent system prompt created
- [ ] `progress.txt` initialized
- [ ] `tasks.json` (or equivalent) initialized
- [ ] Git workflow documented for clean session handoffs
- [ ] State file update instructions included in agent prompts

### Garbage Collection
- [ ] Golden principles defined (3-5)
- [ ] Weekly transcript review scheduled
- [ ] Monthly AGENTS.md review scheduled
- [ ] Quarterly eval suite review scheduled
- [ ] Quality metrics defined and tracking set up

### Iteration
- [ ] Process for reading agent transcripts established
- [ ] Common stuck patterns documented
- [ ] Fix encoding prioritized: linter rule > tool description > documentation
- [ ] Eval comparison tooling set up for measuring improvement

---

## Summary

Building a harness is an engineering discipline, not a creative exercise. The steps are systematic: plan, structure, enforce, design tools, create feedback loops, support long-running work, clean up, and iterate. Each step builds on the previous ones, and skipping a step creates a gap that will haunt you later.

The two examples in this chapter — a full-stack web app and a research agent — demonstrate that the same framework applies across radically different domains. The tools change, the verification methods change, the failure modes change, but the structure is the same. That is the power of the harness pattern: it gives you a repeatable process for making agents effective in any domain.

Start with the checklist. Build the minimum viable harness. Run agents against it. Read the transcripts. Fix what breaks. Measure the improvement. Repeat. The harness will never be "done" — it will evolve with your project, your agents, and your understanding. That is not a flaw; it is the nature of the work.
