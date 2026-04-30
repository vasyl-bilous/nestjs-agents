# nestjs-agents

> **Production-grade Claude Code subagents specialized for NestJS development.**
> Code review, testing, security, architecture — with `git diff` scoping built-in.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Compatible-7C3AED)](https://docs.claude.com/en/docs/claude-code/sub-agents)

## Why this exists

Generic AI coding agents don't know about NestJS-specific patterns:
- **Decorators** (`@Injectable`, `@Controller`, `@UseGuards`)
- **DI scopes** (singleton vs request — and the race-condition trap)
- **Layer responsibilities** (Controller → Service → Repository — never mix them)
- **Testing modules** (`Test.createTestingModule` with `getRepositoryToken`)
- **Transactional outbox** patterns for Stripe / RabbitMQ events
- **Validation pipelines** with `class-validator` DTOs

This collection gives Claude Code that context — and **scopes every review/test action to `git diff`**, so agents act on what you just wrote, not the entire codebase.

## What's inside

| Agent | Purpose | Read-only? | Triggers |
|---|---|---|---|
| **`nestjs-code-reviewer`** | Reviews recent changes for bugs, security, NestJS anti-patterns | ✅ Yes | "review my changes", "check this code" |
| **`nestjs-test-writer`** | Writes Jest unit / integration tests for changed components | ❌ Writes test files | "add tests", "cover with tests" |
| **`nestjs-security-auditor`** | Maps changes to OWASP Top 10 + NestJS-specific traps | ✅ Yes | "security audit", "audit auth flow" |
| **`nestjs-architect`** | Architecture consultant — DDD, hexagonal, microservices, transactional outbox | ✅ Yes | "design this feature", "should I split into microservices" |
| **`nestjs-repository-expert`** | Builds entities, repositories, migrations (TypeORM/Prisma) | ❌ Writes code | "create entity", "add migration", "optimize query" |
| **`nestjs-service-expert`** | Implements service layer — business logic, SOLID, error handling | ❌ Writes code | "create service", "implement use case" |
| **`nestjs-controller-expert`** | Creates controllers, DTOs, endpoints, Swagger docs | ❌ Writes code | "add endpoint", "create controller", "add DTO" |

## Install

### Global (recommended — available in all projects)

```bash
mkdir -p ~/.claude/agents

# Clone or download just the agents
curl -L https://github.com/vasyl-bilous/nestjs-agents/archive/main.tar.gz | \
  tar -xz --strip-components=2 -C ~/.claude/agents \
  nestjs-agents-main/agents

# Verify
ls ~/.claude/agents/nestjs-*.md
```

### Per-project

```bash
cd your-nestjs-project
mkdir -p .claude/agents

curl -L https://github.com/vasyl-bilous/nestjs-agents/archive/main.tar.gz | \
  tar -xz --strip-components=2 -C .claude/agents \
  nestjs-agents-main/agents
```

### Pick specific agents only

```bash
mkdir -p ~/.claude/agents
curl -o ~/.claude/agents/nestjs-code-reviewer.md \
  https://raw.githubusercontent.com/vasyl-bilous/nestjs-agents/main/agents/nestjs-code-reviewer.md
curl -o ~/.claude/agents/nestjs-test-writer.md \
  https://raw.githubusercontent.com/vasyl-bilous/nestjs-agents/main/agents/nestjs-test-writer.md
```

After installing, **restart Claude Code** to pick up the new agents. Verify with:

```
/agents
```

## Workflow examples

### 1. Implement → review → test (typical feature)

```
> implement createOrder method in OrderService with Stripe integration

(main agent writes the code)

> use nestjs-code-reviewer to review what I just wrote

(code-reviewer reads `git diff`, finds: missing idempotency on Stripe charge,
 transaction wrapper missing — returns findings with file:line)

> fix those critical findings

(main agent applies fixes)

> use nestjs-test-writer to add unit tests

(test-writer reads `git diff`, writes OrderService.spec.ts with mocked Stripe,
 covers happy path + error path + edge cases. Runs `pnpm test` to verify.)
```

### 2. Architecture decision before coding

```
> nestjs-architect: I'm building an order processing service that needs to call
  Stripe and send confirmation emails. Should I use a transaction across all of
  this or async events?

(architect responds with concrete recommendation:
 - Use Transactional Outbox pattern
 - Write Order + OutboxEvent in DB transaction
 - Worker reads outbox, calls Stripe with idempotency key
 - Separate worker handles email
 - Trade-offs: eventual consistency vs guaranteed delivery
 - Concrete next step: scaffold OutboxEvent entity)
```

### 3. Security audit before deploy

```
> use nestjs-security-auditor to audit my recent auth changes

(auditor reads `git diff`, finds:
 🔴 A2 Cryptographic — JWT secret hardcoded in module
 🔴 A4 Insecure Design — no rate limiting on /auth/login
 🟡 A7 Identification — generic 'wrong password' message enables enumeration
 → returns findings mapped to OWASP categories with fix suggestions)
```

### 4. Build new feature from scratch

```
> nestjs-architect: design module structure for "Subscription" feature

> nestjs-repository-expert: create Subscription entity + repository based on
  the architect's plan

> nestjs-service-expert: implement SubscriptionService with the business rules
  the architect described

> nestjs-controller-expert: add CRUD endpoints for /subscriptions with Swagger

> nestjs-test-writer: cover what I just built with tests

> nestjs-code-reviewer: review everything before I commit
```

## Design principles applied to all agents

Every agent in this collection follows these rules:

### 1. `git diff` is the scope

Every action-taking agent (reviewer, auditor, test-writer) **first runs `git diff` to find changed files** and operates only on those. This prevents:
- Polluting your main agent's context with unrelated files
- Reviewing thousands of lines when you only changed 10
- "Drift" between what you intended and what the agent did

### 2. Read-only where possible

Reviewers and auditors have **no Write/Edit tools** — they cannot modify your code. They produce findings only, with `file:line` references and suggested fixes.

This is a deliberate guardrail:
- Reviews stay neutral (you decide what to fix)
- Auditors can't accidentally break code while "fixing" something
- Your changes remain reproducible

### 3. NestJS-specific context, not generic

Each agent knows about:
- Decorators, DI scopes, modules, providers
- Guards, pipes, interceptors, filters
- DTOs with `class-validator`, Entities with TypeORM/Prisma
- Testing with `Test.createTestingModule`, `getRepositoryToken`, mocked providers
- NestJS exceptions (`NotFoundException`, `ConflictException`, etc.)

### 4. Specific, file:line references

Findings always look like:

```
🔴 [src/auth/auth.service.ts:42] — Generic error message enables username enumeration
   Risk: attacker can detect valid emails by timing / response difference
   Fix: return same 'Invalid credentials' for both wrong-password and unknown-email
```

Not vague advice like "improve error handling".

## Compatibility

- Claude Code v1.0+
- NestJS v9, v10, v11
- TypeORM and Prisma both supported
- Jest (default) and Vitest (with adaptations)

## Contributing

PRs welcome — especially:
- New agents (e2e-test-writer, performance-auditor, migration-writer)
- NestJS framework version updates (v12 when it drops)
- Additional anti-pattern examples from real projects

Please follow the existing format:
- YAML frontmatter (`name`, `description` with triggers, `tools`, `model`)
- `Step 1: Identify scope` with `git diff`
- `What NOT to do` section
- Concrete output format example

## License

[MIT](LICENSE) — use freely in personal and commercial projects. Attribution appreciated but not required.

---

**Author:** [Vasyl Bilous](https://github.com/vasyl-bilous) — Senior Full-Stack Developer, AI-driven dev practitioner.
Other AI-tooling projects: [`component-library-mcp`](https://github.com/vasyl-bilous/component-library-mcp), [`nestjs-dev-logs-mcp`](https://github.com/vasyl-bilous/nestjs-dev-logs-mcp).
