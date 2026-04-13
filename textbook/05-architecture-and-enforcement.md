# Chapter 5: Architecture and Mechanical Enforcement

> "Documentation is a suggestion. A linter is a law."

There is a moment in every engineering organization's history when someone says: "We should document our architecture so people follow it." They write a wiki page. They present it at an all-hands. They pin it in Slack. And within six months, the codebase has drifted so far from the documented architecture that the wiki page is not just outdated — it is actively misleading.

With human engineers, this drift is gradual. People read the documentation, understand the intent, but make pragmatic exceptions — a quick shortcut here, a "temporary" workaround there. Each individual deviation is small. The compound effect is architectural decay.

With coding agents, this drift is not gradual. It is immediate and exponential. An agent does not read your wiki page. It does not attend your architecture review. It does not absorb institutional knowledge through hallway conversations. It learns your architecture by *observing your codebase* — and if your codebase contains violations of your intended architecture, the agent will replicate those violations faithfully, at scale, without hesitation.

This chapter is about a single, critical insight: **with agents, architectural constraints are what ALLOW speed without drift.** The constraints are not friction — they are the rails that let the train go fast. Without them, speed and correctness are in opposition. With them, they are aligned.

---

## The Warehouse Analogy

Imagine two warehouses. Both contain the same inventory — thousands of parts needed to assemble products.

**Warehouse A** has been organized by the people who work there over many years. Parts are roughly grouped by type, but there are exceptions. The rarely-used specialty bolts are on the third shelf in aisle 7 because that's where Jim put them in 2019 when the usual shelf was full. The red bins contain electrical components except for bin 14, which has hydraulic fittings because Maria needed space during the summer rush. Everyone who works in this warehouse knows where things are. A new hire spends three months learning the layout through tribal knowledge.

**Warehouse B** has clearly labeled shelves. Every part has a designated location printed on a map. The locations are determined by a systematic scheme — category, subcategory, part number. There are no exceptions. If a part doesn't fit the scheme, the scheme is updated and all affected parts are relocated. A new hire can find any part on their first day using the map.

Now deploy a robot in each warehouse. In Warehouse A, the robot is lost. It follows the labeling system (which is inconsistent), picks from the wrong bins, and assembles defective products. In Warehouse B, the robot is immediately productive. The systematic organization *is* its programming.

Your codebase is a warehouse. Your agents are the robots. The question is not *whether* to organize it — it's *how rigorously* to enforce the organization.

---

## Layered Architecture

### The OpenAI Pattern: Types, Config, Repo, Service, Runtime, UI

One of the most effective architectural patterns for agent-worked codebases is a strict layered architecture with explicit dependency direction. OpenAI has publicly described a variant of this pattern, and it serves as an excellent reference implementation. The layers, from bottom to top:

**1. Types** — The foundation layer. Pure data definitions: interfaces, type aliases, enums, value objects. No logic. No imports from any other layer. This layer answers the question: "What are the nouns in our system?"

```typescript
// types/customer.ts
export interface Customer {
  id: CustomerId;
  email: Email;
  plan: PlanTier;
  createdAt: Timestamp;
}

export type CustomerId = string & { readonly brand: unique symbol };
export type PlanTier = "free" | "pro" | "enterprise";
```

**2. Config** — Static configuration and constants. Feature flags, environment-specific settings, default values, magic numbers with names. May import from Types. No logic, no I/O, no side effects.

```typescript
// config/plans.ts
import { PlanTier } from "../types/customer";

export const PLAN_LIMITS: Record<PlanTier, { maxSeats: number; maxStorage: Bytes }> = {
  free: { maxSeats: 5, maxStorage: bytes("1GB") },
  pro: { maxSeats: 50, maxStorage: bytes("100GB") },
  enterprise: { maxSeats: Infinity, maxStorage: bytes("10TB") },
};
```

**3. Repo (Repository)** — Data access. Database queries, API client wrappers, cache interactions. May import from Types and Config. This layer encapsulates *where* data lives and *how* it's retrieved. It does not contain business logic — it does not decide *what* to do with the data, only how to fetch and store it.

```typescript
// repo/customer-repo.ts
import { Customer, CustomerId } from "../types/customer";

export class CustomerRepo {
  async findById(id: CustomerId): Promise<Customer | null> { /* ... */ }
  async findByEmail(email: Email): Promise<Customer | null> { /* ... */ }
  async save(customer: Customer): Promise<void> { /* ... */ }
}
```

**4. Service** — Business logic. The core of the application. May import from Types, Config, and Repo. This is where rules, validations, workflows, and transformations live. Services orchestrate repository calls and apply business rules.

```typescript
// service/billing-service.ts
import { CustomerRepo } from "../repo/customer-repo";
import { PLAN_LIMITS } from "../config/plans";
import { Customer, PlanTier } from "../types/customer";

export class BillingService {
  constructor(private customerRepo: CustomerRepo) {}

  async upgradePlan(customerId: CustomerId, newPlan: PlanTier): Promise<UpgradeResult> {
    const customer = await this.customerRepo.findById(customerId);
    if (!customer) return { success: false, reason: "customer_not_found" };
    if (customer.plan === newPlan) return { success: false, reason: "already_on_plan" };
    // ... business rules ...
  }
}
```

**5. Runtime** — Application wiring, middleware, background jobs, event handlers. May import from all lower layers. This is where dependency injection happens, where services are instantiated with their dependencies, where HTTP middleware chains are assembled.

**6. UI** — Presentation layer. React components, CLI formatters, API response serializers. May import from Types (for type safety) but should not import from Service or Repo directly — it receives data through props, stores, or API responses.

### Dependency Direction: Code Can Only Depend "Forward"

The critical rule is this: **dependencies flow in one direction only — downward through the layer chain.** A Service may import from Repo, but a Repo must never import from Service. UI may import from Types, but Types must never import from UI.

This is the **Dependency Rule** from Robert Martin's Clean Architecture, applied with a specific layer ordering. The rule has a profound consequence: **any layer can be tested, replaced, or reasoned about independently of the layers above it.** You can swap out the Repo layer (move from PostgreSQL to DynamoDB) without changing any Service code, as long as the Repo interfaces (defined in Types) remain stable.

For agent-worked codebases, this rule provides an even more important benefit: **it constrains the blast radius of agent changes.** If an agent modifies a Service, the change cannot possibly affect how data is stored (Repo) or how types are defined (Types). The layer boundaries create *firewalls* that prevent changes from propagating in unexpected directions.

### Cross-Cutting Concerns Through Explicit Provider Interfaces

Some concerns genuinely cut across layers: logging, authentication, metrics, tracing, feature flags. The temptation is to import these directly wherever needed, creating a web of cross-cutting dependencies that violates the layer model.

The solution is the **Provider pattern** (a variant of the Strategy pattern and Dependency Inversion Principle). Define a Provider interface in the Types layer, implement it in the Runtime layer, and inject it into Services and Repos at construction time:

```typescript
// types/providers.ts
export interface LogProvider {
  info(message: string, context?: Record<string, unknown>): void;
  error(message: string, error?: Error, context?: Record<string, unknown>): void;
}

export interface MetricsProvider {
  increment(metric: string, tags?: Record<string, string>): void;
  histogram(metric: string, value: number, tags?: Record<string, string>): void;
}

export interface AuthProvider {
  getCurrentUser(): Promise<User>;
  hasPermission(user: User, permission: Permission): boolean;
}
```

```typescript
// runtime/providers.ts
import { LogProvider } from "../types/providers";

export class StructuredLogProvider implements LogProvider {
  info(message: string, context?: Record<string, unknown>): void {
    // Writes structured JSON to stdout with OpenTelemetry trace IDs
  }
  // ...
}
```

Services receive providers through constructor injection:

```typescript
// service/billing-service.ts
export class BillingService {
  constructor(
    private customerRepo: CustomerRepo,
    private log: LogProvider,
    private metrics: MetricsProvider,
  ) {}
}
```

This pattern ensures that cross-cutting concerns are:
- **Explicitly declared** (visible in the constructor signature)
- **Consistently implemented** (one implementation, injected everywhere)
- **Testable** (swap in a mock provider for unit tests)
- **Layer-compliant** (the interface is in Types, the implementation is in Runtime)

### Why This Matters Now, Not Later

Traditional advice says: "Don't over-architect. Start simple and add structure when complexity demands it." This advice assumes human engineers who can carry architectural intent in their heads and refactor organically as the system grows. It assumes that architectural knowledge is distributed across the team through code reviews, pair programming, and design discussions.

With agents, none of these assumptions hold. The agent doesn't carry architectural intent. It doesn't participate in design discussions. It infers architecture from the code it can see — and if the code has no clear architecture, the agent's output will have no clear architecture.

**Layered architecture is not premature optimization in an agent-assisted codebase. It is a prerequisite.** Teams that historically postponed architectural structure until they had hundreds of engineers find that, with agents, they need it at ten engineers — or even at two, if those two engineers are each running multiple agents.

The upfront cost of establishing layers, defining boundaries, and writing enforcement rules is modest — days, not weeks. The cost of retrofitting architecture onto a codebase that agents have filled with inconsistent patterns is enormous.

---

## Mechanical Enforcement (Not Documentation!)

The architecture described above is useless if it exists only in documentation. This is the central lesson of this section, and it bears repeating: **enforcement is the only form of architecture that agents respect.**

Documentation is a human coordination mechanism. It works (imperfectly) because humans read prose, internalize intent, and make judgment calls. Agents do not do any of these things. An agent's understanding of your architecture comes from two sources:

1. **The patterns already present in the code** (which it infers through its context window).
2. **The error messages it receives when it violates constraints** (which it uses to correct its behavior).

Neither of these sources involves reading a wiki page. Both of these sources are strengthened by mechanical enforcement.

### Custom Linters (Themselves Generated by Agents!)

Custom lint rules are the workhorse of architectural enforcement. They are fast, deterministic, and provide immediate feedback. And here is the delightful recursion: **you can use agents to write the lint rules that constrain agents.**

Consider the layered architecture above. The fundamental constraint — "code can only depend forward through the layer chain" — can be expressed as a lint rule:

```javascript
// eslint-plugin-architecture/rules/layer-dependency.js
const LAYER_ORDER = ["types", "config", "repo", "service", "runtime", "ui"];

module.exports = {
  meta: {
    type: "problem",
    docs: { description: "Enforce layer dependency direction" },
    messages: {
      invalidImport: "{{currentLayer}} layer cannot import from {{importedLayer}} layer. " +
        "Dependencies must flow downward: types → config → repo → service → runtime → ui. " +
        "Move shared logic to a lower layer or use a Provider interface. " +
        "See docs/ARCHITECTURE.md#layer-dependencies"
    }
  },
  create(context) {
    return {
      ImportDeclaration(node) {
        const currentLayer = getLayerFromPath(context.getFilename());
        const importedLayer = getLayerFromPath(node.source.value);
        if (currentLayer && importedLayer) {
          const currentIndex = LAYER_ORDER.indexOf(currentLayer);
          const importedIndex = LAYER_ORDER.indexOf(importedLayer);
          if (importedIndex > currentIndex) {
            context.report({
              node,
              messageId: "invalidImport",
              data: { currentLayer, importedLayer }
            });
          }
        }
      }
    };
  }
};
```

This rule is straightforward to write — an agent can generate it from a natural language description of the constraint. More importantly, it is **self-documenting through its error message.** When an agent (or a human) violates the constraint, the error message explains not just *what* is wrong but *why* it's wrong and *how* to fix it.

### Structural Tests That Validate Dependency Directions

Lint rules catch violations at the file level. **Structural tests** catch violations at the module and package level, validating higher-order architectural properties.

```typescript
// tests/architecture.test.ts
import { analyzeImports } from "./utils/import-analyzer";

describe("Architecture: Layer Dependencies", () => {
  const imports = analyzeImports("src/");

  test("Types layer has no internal imports", () => {
    const typesImports = imports.filter(i => i.sourceLayer === "types" && i.targetLayer !== null);
    expect(typesImports).toEqual([]);
  });

  test("Repo layer does not import from Service", () => {
    const violations = imports.filter(
      i => i.sourceLayer === "repo" && i.targetLayer === "service"
    );
    expect(violations).toEqual([]);
  });

  test("No circular dependencies between modules", () => {
    const cycles = findCycles(imports);
    expect(cycles).toEqual([]);
  });

  test("All Services receive dependencies through constructor injection", () => {
    const services = findClasses("src/service/");
    for (const service of services) {
      const directImports = service.body.filter(isDirectInstantiation);
      expect(directImports).toEqual([]);
    }
  });
});
```

These tests run in CI alongside unit and integration tests. They are not testing behavior — they are testing *structure*. This is the **ArchUnit** pattern (popularized in the Java ecosystem by the ArchUnit library) applied to any language. The test suite becomes a living, executable specification of your architectural rules.

### CI Jobs That Block PRs Violating Boundaries

The enforcement chain must be complete. Lint rules and structural tests are meaningless if developers (or agents) can merge code that fails them. **Configure CI to block pull requests that violate architectural constraints.**

This means:
- Lint rules run as a required CI check. No merge until clean.
- Structural tests run as a required CI check. No merge until clean.
- These checks run on every PR, not just on a nightly schedule.
- The checks cannot be bypassed without explicit approval from a designated reviewer (an architect or tech lead).

The non-bypassability is important. In a human-only workflow, it's sometimes reasonable to merge a quick fix that violates a lint rule and clean it up later. In an agent-assisted workflow, this creates a precedent in the codebase that the agent will replicate. **Every merged violation teaches every future agent that the violation is acceptable.**

### Why Enforcement Beats Documentation: The Replication Problem

To understand why enforcement is so much more important than documentation in agent-assisted codebases, you need to understand the **replication problem.**

When an agent writes new code, it does so by observing existing code in the repository and generating code that is *stylistically and structurally similar*. This is not a bug — it's a feature. You *want* the agent to follow existing conventions. But it means that every pattern in your codebase — good or bad — will be replicated proportionally to its prevalence.

If your codebase has 100 files following the correct import pattern and 3 files with incorrect imports (legacy exceptions, quick hacks, migration artifacts), the agent will almost always follow the correct pattern. But as the codebase grows — especially with agents writing significant portions of new code — those 3 exceptions don't stay at 3. Each one might be replicated. Now you have 8. Then 15. Then 30. The ratio of correct to incorrect shifts, and eventually the incorrect pattern becomes prevalent enough that the agent treats it as an *alternative convention* rather than an error.

This is **exponential pattern decay**. Documentation cannot prevent it because the agent doesn't read the documentation — it reads the code. Only mechanical enforcement prevents it because enforcement ensures that the incorrect pattern is *never merged in the first place*, keeping the signal-to-noise ratio in the codebase clean.

Think of enforcement as **pattern hygiene** for an immune system that has no natural resistance to bad patterns. Every merged violation is an infection that will spread.

---

## Error Messages as Agent Context

We touched on error messages in Chapter 4 when discussing tool design. Here, we revisit the topic with a different focus: **linter and CI error messages as a mechanism for teaching agents.**

When a linter fails on code an agent has written, the error message is injected into the agent's context as part of the tool response (typically the output of a `run_command` or `run_tests` tool). At that moment, the error message becomes **just-in-time instruction** — a targeted explanation of what the agent did wrong and how to fix it, delivered at exactly the moment the agent needs it.

This is far more effective than up-front instruction (system prompts, documentation) for two reasons:

1. **Relevance.** The error is about the exact code the agent just wrote. There's no question of whether the instruction applies — it obviously does. Compare this to a system prompt that says "always follow the layer dependency rules" — the agent must figure out when and how that instruction applies to the current context.

2. **Token efficiency.** The error message appears only when needed. A comprehensive architecture guide in the system prompt consumes tokens on every single agent invocation, even when the agent isn't touching architectural boundaries. An error message consumes tokens only when the agent has made a mistake.

### Crafting Remediation-Focused Error Messages

The difference between an error message that helps an agent self-correct and one that causes it to thrash is substantial. Here is a taxonomy of error message quality levels:

**Level 0: Cryptic**
```
Error: E0147
```
The agent has no ability to self-correct. It will either retry blindly or give up.

**Level 1: Diagnostic**
```
Error: Invalid import in service layer
```
The agent knows something is wrong with an import but doesn't know what to do about it. It may try removing the import, which could break other things.

**Level 2: Explanatory**
```
Error: Service layer (src/service/billing.ts) cannot import from UI layer (src/ui/components/Button.tsx).
Dependencies must flow downward: types → config → repo → service → runtime → ui.
```
The agent understands the rule and the specific violation. It can likely fix this by restructuring the import.

**Level 3: Remediation-Focused**
```
Error: Service layer cannot import from UI layer.

  Violation: src/service/billing.ts imports from src/ui/components/PriceDisplay.tsx
  Rule: Dependencies must flow downward (types → config → repo → service → runtime → ui)

  How to fix:
  - If you need the PriceDisplay formatting logic in the service layer, extract it
    into a pure function in src/service/formatters/ or src/types/formatting.ts
  - If you need to render prices in the UI, pass formatted data from the service
    layer to the UI layer through props or a store
  - See docs/ARCHITECTURE.md#layer-dependencies for examples
```
The agent understands the rule, the specific violation, *and* the concrete steps to fix it. It also has a reference document it can read for more context if needed. This level of error message is effectively a mini-tutorial delivered at the point of need.

**Always aim for Level 3.** The additional effort to write detailed error messages is trivial compared to the time saved by agents that can self-correct on the first attempt rather than thrashing through multiple failed retries.

### Error Messages as Implicit Documentation

Here is an underappreciated benefit of well-crafted error messages: they **become the canonical documentation** for your architectural rules. When you write a lint rule with a Level 3 error message, you have simultaneously:

1. Defined the rule (the lint logic)
2. Documented the rule (the error message)
3. Enforced the rule (the CI check)
4. Taught the rule (the remediation guidance)

All four in a single artifact. Compare this to the traditional approach: define the rule in a design document, document it on the wiki, enforce it through code review (manually, inconsistently), and teach it through onboarding sessions. The lint-rule-with-good-error-message approach is more reliable, more maintainable, and more effective — for both humans and agents.

---

## Encoding Taste Into the Codebase

Architecture is about structure. But great codebases have something beyond structure — they have *taste*. Taste is the difference between code that is technically correct and code that is maintainable, readable, and expressive. It is the difference between a function that works and a function that *obviously* works.

Taste has traditionally been the domain of senior engineers, transmitted through code review, pair programming, and mentorship. It is subjective, contextual, and hard to articulate. But here is the key insight for agent-assisted development:

**If you can articulate what you don't like about the code, you can encode that articulation as a rule.**

And if you can encode it as a rule, you can enforce it mechanically. And if you can enforce it mechanically, every agent in your organization will follow it, always, without drift.

### From Opinion to Enforcement: The Encoding Pipeline

The process of encoding taste follows a consistent pipeline:

**1. Notice a pattern you dislike.** During code review, you see code that's technically correct but feels wrong. Maybe it's a component that mixes data fetching and rendering. Maybe it's a function that handles too many concerns. Maybe it's an error message that isn't user-friendly.

**2. Articulate why you dislike it.** This is the critical step. "I don't like this code" is useless. "This component violates the single-responsibility principle by combining data fetching with rendering, which makes it untestable and forces re-rendering when the data-fetching logic changes" — that is an encodable rule.

**3. Choose an enforcement mechanism.** Depending on the nature of the rule:
   - **Custom lint rule** for syntactic patterns (import restrictions, naming conventions, banned functions)
   - **Structural test** for architectural properties (dependency directions, module boundaries)
   - **Custom code reviewer** (an agent configured with review criteria) for semantic properties (code clarity, appropriate abstraction level)
   - **Type system constraint** for data flow properties (making invalid states unrepresentable)

**4. Write the enforcement.** Implement the lint rule, test, or reviewer. Generate it with an agent — describe the rule in natural language and let the agent write the enforcement code.

**5. Write the error message.** Include a clear explanation of what was detected, why it's problematic, and how to fix it. This is the remediation guidance that will teach future agents (and engineers) the rule.

### Case Study: The Duplicate Helper Function Problem

Here is a concrete example. Your codebase has a utility function `formatCurrency(amount, currency)` that formats monetary values for display. It's used throughout the codebase. During a code review, you notice that someone has created a second function with almost identical functionality:

```typescript
// src/utils/format.ts (the canonical implementation)
export function formatCurrency(amount: number, currency: string): string {
  // Connected to OpenTelemetry for performance monitoring
  // Handles locale-specific formatting
  // Logs formatting errors to Sentry
  return new Intl.NumberFormat(getLocale(), {
    style: "currency",
    currency,
  }).format(amount);
}

// src/features/checkout/helpers.ts (the duplicate)
function formatPrice(amount: number, curr: string): string {
  // No OpenTelemetry
  // No error logging
  // Hardcoded locale
  return `$${amount.toFixed(2)}`;
}
```

The duplicate is technically functional — it produces formatted prices. But it's missing observability (no OpenTelemetry tracing), error handling (no Sentry logging), and internationalization (hardcoded dollar sign and locale). If an agent sees both functions and needs to format a price, it might use either one. The duplicate creates a pattern that will replicate.

**The enforcement:** Write a custom ESLint rule that detects functions matching the signature pattern of the canonical `formatCurrency` function and bans them from being defined anywhere except `src/utils/format.ts`:

```javascript
// eslint-plugin-custom/rules/no-duplicate-currency-formatter.js
module.exports = {
  meta: {
    type: "problem",
    messages: {
      duplicateFormatter:
        "Do not create custom currency formatting functions. " +
        "Use formatCurrency() from 'src/utils/format.ts' instead. " +
        "The canonical implementation includes OpenTelemetry tracing, " +
        "Sentry error logging, and locale-aware formatting. " +
        "Custom implementations miss these cross-cutting concerns. " +
        "If formatCurrency() doesn't meet your needs, extend it rather than duplicating it."
    }
  },
  create(context) {
    return {
      FunctionDeclaration(node) {
        if (looksLikeCurrencyFormatter(node) && !isInAllowedPath(context.getFilename())) {
          context.report({ node, messageId: "duplicateFormatter" });
        }
      }
    };
  }
};
```

The function `looksLikeCurrencyFormatter` uses heuristics — checking for parameters named `amount`/`price`/`value` and `currency`/`curr`/`code`, return type of string, and body containing number formatting logic. It doesn't need to be perfect; false positives are caught during review, and false negatives are caught over time as the heuristics are refined.

### The Compound Effect: Individual Expertise as Organizational Multiplier

Here is where encoding taste produces its most powerful effect. In a traditional team, an engineer's expertise benefits the code they personally write and the code they personally review. Their impact is linear — bounded by their time.

When that engineer encodes their expertise as rules, their impact becomes multiplicative. The rule runs on every PR, generated by every agent, across every team member. The front-end specialist who encodes React component patterns affects *every* React component written by any agent in the organization.

Consider this scenario:

1. **Day 1:** A front-end architect joins the team. They review existing React code and notice that components are monolithic — mixing state management, data fetching, and rendering in single files.

2. **Day 2:** They articulate their preferred pattern: custom hooks for state and data fetching, pure presentational components for rendering, container components for composition. They write three lint rules:
   - `no-fetch-in-component`: Detects `fetch`/`axios`/`useSWR` calls inside component files (must be in hook files)
   - `max-component-responsibilities`: Detects components that define more than 2 `useState` calls and also render JSX (split into hook + component)
   - `prefer-composition`: Detects components longer than 100 lines (suggest decomposition)

3. **Day 3:** The rules are deployed to CI. Every agent that writes React code — for every engineer on the team — now follows the front-end architect's patterns. Not because the agents read a style guide. Not because the agents attended a workshop. Because the architecture is mechanically enforced, and the error messages teach the correct pattern.

4. **Day 30:** The codebase has 50 new React components, all following the architect's patterns. Every agent-written component uses custom hooks for data fetching, keeps presentational components pure, and decomposes complex UIs into small, testable pieces. The architect's taste has been *multiplied* across the entire team's agent fleet.

This is the **expertise multiplier effect**: each person's encoded taste makes every other person's agents more effective. The senior engineer who writes a lint rule for error handling patterns improves every agent-written error handler. The security engineer who writes a structural test for authentication boundaries prevents every agent from creating unauthenticated endpoints. The database engineer who writes a custom reviewer for query patterns catches every N+1 query an agent generates.

The total effect is greater than the sum of individual contributions. It is a network effect where each new rule increases the value of all existing rules by raising the baseline quality that agents start from.

---

## Repository as Single Source of Truth

All of the enforcement mechanisms described above — lint rules, structural tests, CI checks — operate on the codebase. They can only enforce rules about artifacts that exist in the repository. This leads to a principle that is obvious once stated but profound in its implications:

**Anything the agent can't access in-context doesn't exist.**

If a design decision was made in a Slack thread, the agent doesn't know about it. If a security requirement is documented in a Confluence page that the agent can't access, the requirement doesn't exist for the agent. If a product spec lives in a Google Doc, the agent cannot reference it when making implementation decisions.

The solution is radical but necessary: **make the repository the single source of truth for all decisions that affect the code.** Not a secondary copy. Not a reference. The source.

### Design Decisions: Versioned Markdown in `docs/design-docs/`

Design decisions — the *why* behind architectural choices — belong in the repository, versioned alongside the code they affect. The format is straightforward:

```
docs/
  design-docs/
    001-authentication-strategy.md
    002-database-migration-approach.md
    003-api-versioning-policy.md
    004-event-sourcing-for-billing.md
```

Each design document follows a consistent template:

```markdown
# DD-003: API Versioning Policy

## Status: Accepted
## Date: 2026-02-15
## Participants: Alice Chen, Bob Kumar, Carol Torres

## Context
Our API is consumed by 47 external integrators. Breaking changes cause integration
failures and support burden. We need a versioning strategy that allows evolution
without breakage.

## Decision
We will use URL-path versioning (e.g., /v2/customers) with the following rules:
- New major versions may remove or change existing fields
- Minor changes (new optional fields, new endpoints) do not require a version bump
- We support the current version and one previous version (N and N-1)
- Deprecation notices are added 90 days before a version is sunset

## Alternatives Considered
- Header-based versioning: Rejected because it's harder for integrators to test
  in browsers and harder for us to route at the load balancer level.
- Query parameter versioning: Rejected because it makes caching more complex
  and pollutes analytics.

## Consequences
- Every API endpoint must be organized under a version prefix
- Controllers must be versioned (v1/CustomerController, v2/CustomerController)
- Shared logic goes in version-agnostic service layer, not in controllers
```

This format — inspired by the **Architecture Decision Record (ADR)** pattern — captures not just the decision but the context, alternatives, and consequences. When an agent needs to add a new API endpoint, it can read the design doc and understand *why* endpoints are organized the way they are, not just *how*.

### Execution Plans: `docs/exec-plans/` with Progress Logs

For larger initiatives — migrations, refactors, feature epics — **execution plans** track the intended sequence of work and its progress. These are living documents that agents can read to understand what has been done and what remains.

```markdown
# Exec Plan: Migrate from REST to GraphQL

## Status: In Progress (Phase 2 of 4)
## Owner: Platform Team
## Started: 2026-01-10
## Target Completion: 2026-04-30

## Phases

### Phase 1: Schema Definition [COMPLETED]
- [x] Define GraphQL schema for Customer domain
- [x] Define GraphQL schema for Order domain
- [x] Define GraphQL schema for Product domain
- [x] Set up code generation from schema

### Phase 2: Resolver Implementation [IN PROGRESS]
- [x] Customer resolvers (queries and mutations)
- [x] Order resolvers (queries only)
- [ ] Order resolvers (mutations) ← CURRENT
- [ ] Product resolvers
- [ ] Cross-domain resolvers (e.g., customer.orders)

### Phase 3: Client Migration [NOT STARTED]
- [ ] Update web dashboard to use GraphQL
- [ ] Update mobile app to use GraphQL
- [ ] Update internal tools to use GraphQL

### Phase 4: REST Deprecation [NOT STARTED]
- [ ] Add deprecation headers to REST endpoints
- [ ] Monitor REST usage and contact remaining consumers
- [ ] Remove REST endpoints after 90-day deprecation window

## Progress Log
- 2026-03-20: Completed order query resolvers. Mutations blocked on payment
  service refactor (see DD-012).
- 2026-03-05: Completed customer resolvers. Performance is within 10% of REST
  for equivalent queries.
- 2026-02-01: Schema definition complete. Chose code-first approach with
  TypeGraphQL. See DD-011 for rationale.
```

An agent working on the order mutations can read this plan and understand: it's in Phase 2, customer resolvers are the reference implementation, there's a dependency on the payment service refactor, and performance parity with REST is a success criterion. All without a human needing to explain the context.

### Product Specs: `docs/product-specs/` with Indexed Navigation

Product specifications — the *what* and *for whom* behind features — also belong in the repository. An index file provides navigation:

```markdown
# Product Specifications Index

## Active Specs
- [PS-042: Self-Service Plan Upgrades](./042-self-service-upgrades.md) - In Development
- [PS-041: Usage-Based Billing Dashboard](./041-usage-billing-dashboard.md) - In Development
- [PS-040: Team Management Improvements](./040-team-management.md) - Complete

## Archived Specs
- [PS-039: Legacy Billing Sunset](./039-legacy-billing-sunset.md) - Complete
```

Each spec contains enough detail for an agent to make product-informed implementation decisions: user stories, acceptance criteria, edge cases, and out-of-scope items.

### Technical Debt: Tracked in `docs/exec-plans/tech-debt-tracker.md`

Technical debt that is tracked only in issue trackers (Jira, Linear, GitHub Issues) is invisible to agents. A dedicated debt tracker in the repository makes it visible:

```markdown
# Technical Debt Tracker

## High Priority
| ID | Description | Impact | Remediation | Owner |
|----|-------------|--------|-------------|-------|
| TD-001 | Payment service uses deprecated Stripe API v2 | Will break when Stripe sunsets v2 (2026-06) | Migrate to v3, see exec-plan/stripe-migration.md | @alice |
| TD-002 | Customer search uses LIKE queries instead of full-text search | Slow for >100k customers, no relevance ranking | Implement Elasticsearch indexing | @bob |

## Medium Priority
| ID | Description | Impact | Remediation | Owner |
|----|-------------|--------|-------------|-------|
| TD-003 | Test suite takes 45 minutes | Slow CI feedback loop | Parallelize, remove redundant integration tests | @carol |
```

When an agent is about to modify the payment service, it can consult this tracker and know to use Stripe API v3, not v2. Without the tracker, the agent would infer the API version from existing code (v2) and perpetuate the technical debt.

### Story: The Slack Thread Security Decision

Here is a story that illustrates why repository-as-source-of-truth matters in practice.

A team at a mid-stage startup needed to choose a cryptography library for hashing user passwords. The security engineer evaluated three options: `bcrypt`, `argon2`, and `scrypt`. After analysis, they recommended `argon2` for its memory-hardness properties and resistance to GPU-based attacks. The decision was discussed and approved in a Slack thread in the `#security` channel.

The team started using `argon2` in new code. But the decision was never recorded anywhere except Slack. Six months later:

1. A new engineer joined the team. Their agent was tasked with implementing a password reset feature.
2. The agent searched the codebase and found two patterns: some files used `argon2` (the correct choice), and some legacy files used `bcrypt` (from before the decision).
3. The agent chose `bcrypt` — it appeared in more files (legacy code was more extensive), and `bcrypt` is more commonly discussed in the agent's training data.
4. The PR was merged after a cursory review. The security regression went unnoticed for weeks.

The fix was two-fold:

**1. Reflect the decision into the codebase** by creating a design document:

```markdown
# DD-015: Password Hashing Algorithm

## Status: Accepted
## Date: 2025-09-15

## Decision
Use argon2id for all password hashing. Do not use bcrypt, scrypt, or any other
algorithm. The argon2id variant provides resistance to both GPU and side-channel
attacks.

## Implementation
- Use the `argon2` npm package (v0.31+)
- Configuration: memory cost 64MB, time cost 3, parallelism 4
- All existing bcrypt hashes will be transparently upgraded on next login
```

**2. Add a guardrail** by writing a lint rule:

```javascript
// Ban bcrypt imports
{
  "no-restricted-imports": [{
    "name": "bcrypt",
    "message": "Use argon2 for password hashing (see docs/design-docs/015-password-hashing.md). bcrypt is deprecated per security policy."
  }, {
    "name": "bcryptjs",
    "message": "Use argon2 for password hashing (see docs/design-docs/015-password-hashing.md). bcrypt variants are deprecated per security policy."
  }]
}
```

Now, every agent that tries to import `bcrypt` gets an error message explaining that `argon2` is the correct choice, with a link to the design doc explaining why. The Slack thread's decision has been encoded into the codebase as an enforceable rule. The institutional knowledge is no longer trapped in a channel that agents cannot access.

This story generalizes: **every important decision that lives outside the repository is a latent bug waiting for an agent to make the wrong choice.** The solution is not to prevent agents from making decisions — it's to ensure the repository contains enough context for them to make the *right* decisions.

---

## Bringing It All Together

The architecture and enforcement patterns in this chapter form a coherent system:

1. **Layered architecture** provides a clear structure that agents can observe and replicate.
2. **Custom lint rules** enforce layer boundaries and prevent pattern decay.
3. **Structural tests** validate higher-order architectural properties.
4. **CI checks** ensure no violation reaches the main branch.
5. **Remediation-focused error messages** teach agents (and humans) the rules at the point of violation.
6. **Encoded taste** turns individual expertise into organizational standards.
7. **Repository-as-source-of-truth** ensures agents have access to all context needed for good decisions.

Each layer of this system reinforces the others. The lint rules reference design documents. The design documents explain the architectural decisions. The structural tests validate the architecture. The CI checks enforce the tests. The error messages point to the design documents. It is a closed loop where every component supports every other component.

The result is a codebase where agents can operate at high speed with high confidence. Not because the agents are perfectly intelligent — they are not. But because the codebase is structured so that the *most likely* action an agent takes is the *correct* action, and the *incorrect* actions are caught before they reach production.

This is the paradox of constraints in agent-assisted development: **more constraints produce more freedom.** The team with strict architectural enforcement ships faster than the team with "move fast and break things" because their agents don't break things. The upfront investment in rules, tests, and enforcement pays compound returns as the team scales — each new agent, each new engineer, each new tool operates within a system designed to make correctness the path of least resistance.

In the next chapter, we'll explore how to test and evaluate agent systems — how to know whether your tools, architecture, and enforcement patterns are actually working, and how to identify and fix the gaps.

---

## Annotated Bibliography

**[1]** Lopopolo, Ryan. "Harness Engineering: Leveraging Codex in an Agent-First World." *OpenAI*, February 2026. https://openai.com/index/harness-engineering/

> Describes the layered architecture pattern (Types, Config, Repo, Service, Runtime, UI) that this chapter presents as a reference implementation. Covers the rationale for strict dependency direction and mechanical enforcement in agent-worked codebases, directly informing the chapter's treatment of architectural layers and the warehouse analogy.

**[2]** Anthropic. "Building Effective Agents." *Anthropic Engineering Blog*, December 19, 2024. https://www.anthropic.com/engineering/building-effective-agents

> Provides the foundational agent design principles that motivate this chapter's emphasis on mechanical enforcement over documentation. Covers how agents infer patterns from existing code and why architectural guardrails are essential for maintaining codebase quality at scale.

**[3]** Anthropic. "Effective Context Engineering for AI Agents." *Anthropic Engineering Blog*, September 29, 2025. https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

> Directly relevant to the chapter's discussion of error messages as agent context and the repository as single source of truth. Covers how information injected into an agent's context window shapes its behavior, supporting the chapter's argument that remediation-focused error messages serve as just-in-time instruction.

**[4]** Anthropic. "Prompting Best Practices — Claude 4.6." *Claude Documentation*, 2025. https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices

> Informs the chapter's treatment of how agents interpret system-level instructions and codebase patterns. Relevant to the sections on encoding taste, crafting lint rule error messages, and designing the repository artifacts (design docs, execution plans) that guide agent behavior.

**[5]** Anthropic. "Writing Effective Tools for AI Agents." *Anthropic Engineering Blog*, September 11, 2025. https://www.anthropic.com/engineering/writing-tools-for-agents

> Complements this chapter's focus on enforcement by covering how tool-level error responses guide agent self-correction. The error message design principles from this source directly inform the chapter's taxonomy of error message quality levels (cryptic through remediation-focused).
