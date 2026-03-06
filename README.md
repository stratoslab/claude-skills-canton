# Claude Skills for Canton Network & DAML

A comprehensive Claude Code skill set for building on the **Canton Network** and writing **DAML smart contracts**. Inspired by [everything-claude-code](https://github.com/affaan-m/everything-claude-code), tailored specifically for Canton/DAML developers.

**49 files | 27 skills | 10 commands | 6 agents | 3 rule sets | 6,000+ lines of Canton/DAML knowledge**

---

## Table of Contents

- [What This Is](#what-this-is)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [How Claude Code Uses Each Component](#how-claude-code-uses-each-component)
- [All Skills (27)](#all-skills-27)
- [All Commands (10)](#all-commands-10)
- [All Agents (6)](#all-agents-6)
- [All Rules (3)](#all-rules-3)
- [Example CLAUDE.md Configs (3)](#example-claudemd-configs-3)
- [Contexts (3)](#contexts-3)
- [How to Use in Your Project](#how-to-use-in-your-project)
- [Typical Developer Workflow](#typical-developer-workflow)
- [Repository Structure](#repository-structure)
- [Contributing](#contributing)
- [Links](#links)

---

## What This Is

This repo turns Claude Code into a Canton Network expert. It contains:

- **Skills** — Deep reference docs that Claude reads when working on specific topics (DAML templates, Canton architecture, Ledger API, etc.)
- **Commands** — Slash commands (`/daml-build`, `/canton-start`, etc.) for common developer actions
- **Agents** — Specialized subagents Claude can delegate to (architect, reviewer, ops, debugger, security auditor, test runner)
- **Rules** — Always-on coding standards and security guidelines Claude follows for every DAML/Canton file
- **Examples** — Ready-to-use `CLAUDE.md` configurations for different project types
- **Contexts** — Mode-switching prompts (development, review, operations)

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/stratoslab/claude-skills-canton.git
cd claude-skills-canton

# 2. Install rules (always-active guidelines for Claude Code)
mkdir -p ~/.claude/rules
cp rules/*.md ~/.claude/rules/

# 3. In your Canton/DAML project, create a CLAUDE.md
#    (see "How to Use in Your Project" below)
```

That's it. Claude Code will now use these skills, rules, and agents when working on your Canton/DAML projects.

---

## Installation

### Option 1: Full Install (Recommended)

```bash
# Clone to your home directory (or anywhere you prefer)
git clone https://github.com/stratoslab/claude-skills-canton.git ~/claude-skills-canton

# Install rules globally (Claude Code auto-loads files from ~/.claude/rules/)
mkdir -p ~/.claude/rules
cp ~/claude-skills-canton/rules/*.md ~/.claude/rules/
```

**What this does:**
- Rules in `~/.claude/rules/` are **automatically loaded** by Claude Code in every session
- Skills, commands, agents, and examples are available when you reference them from your project's `CLAUDE.md`

### Option 2: Project-Level Only

If you only want the skills for a specific project:

```bash
# Inside your DAML project directory
git clone https://github.com/stratoslab/claude-skills-canton.git .claude-skills

# Reference in your project's CLAUDE.md (see examples below)
```

### Option 3: Cherry-Pick Individual Skills

Copy only the skills you need:

```bash
# Example: only DAML template and testing skills
mkdir -p my-project/.claude/skills
cp -r ~/claude-skills-canton/skills/daml-templates my-project/.claude/skills/
cp -r ~/claude-skills-canton/skills/daml-script-testing my-project/.claude/skills/
cp -r ~/claude-skills-canton/skills/testing-workflow my-project/.claude/skills/
```

---

## How Claude Code Uses Each Component

Understanding how each piece works helps you get the most out of this repo.

### Skills (`skills/*/SKILL.md`)

**What they are:** Deep reference documents with code examples, patterns, best practices, and pitfalls for specific topics.

**How Claude uses them:** When Claude encounters a task related to a skill topic (e.g., writing a DAML template), it reads the relevant `SKILL.md` to get expert-level patterns and code examples. Skills are activated by context — Claude reads them when the topic is relevant.

**How to reference them:** In your project's `CLAUDE.md`, point to the skills directory:

```markdown
## Canton Skills
When working on DAML contracts, reference skills from: ~/claude-skills-canton/skills/

Key skills for this project:
- daml-templates: Template authoring patterns
- daml-choices: Choice design
- canton-local-dev: Local development setup
- testing-workflow: TDD approach
```

Or ask Claude directly: *"Read the daml-patterns skill and design a propose-accept workflow for trade settlement."*

### Commands (`commands/*.md`)

**What they are:** Step-by-step instructions for common developer actions, designed to be invoked as slash commands.

**How Claude uses them:** When you type a command name (e.g., "run daml-build"), Claude reads the command file and follows its steps.

**How to use them:**

```
You: "Run daml-build"
Claude: Reads commands/daml-build.md, then runs `daml build`, handles errors, reports DAR path

You: "Run daml-review on my templates"
Claude: Reads commands/daml-review.md, then reviews your DAML code against the checklist

You: "Run canton-start with a multi-node setup"
Claude: Reads commands/canton-start.md, sets up Canton config and starts nodes
```

### Agents (`agents/*.md`)

**What they are:** Specialized Claude subagent definitions with specific roles, tools, and instructions.

**How Claude uses them:** Claude can delegate complex tasks to specialized agents. Each agent has defined capabilities and a focused review process.

**How to invoke them:**

```
You: "Use the daml-architect agent to design a bond issuance data model"
Claude: Delegates to the daml-architect agent, which analyzes requirements,
        designs templates, plans authorization flows, and checks privacy

You: "Run the security-auditor agent on my contracts"
Claude: Delegates to security-auditor, which systematically checks for
        authorization vulnerabilities, privacy leaks, and missing validation
```

### Rules (`rules/*.md`)

**What they are:** Always-follow guidelines that Claude checks automatically when working on matching files.

**How Claude uses them:** Rules are loaded based on file glob patterns. When Claude edits a `.daml` file, it automatically follows `daml-coding-standards.md` and `daml-security.md`. When editing `.conf` files, it follows `canton-config-standards.md`.

**How they activate:** Automatic — no action needed. After copying to `~/.claude/rules/`, they apply to all sessions.

| Rule File | Activates On | What It Enforces |
|-----------|-------------|------------------|
| `daml-coding-standards.md` | `**/*.daml` | 2-space indent, naming, minimal signatories, ensure clauses, testing |
| `daml-security.md` | `**/*.daml` | No generic action choices, privacy validation, input checking |
| `canton-config-standards.md` | `**/*.conf` | HOCON format, env vars for secrets, TLS, monitoring |

### Examples (`examples/*-CLAUDE.md`)

**What they are:** Ready-to-use `CLAUDE.md` project configuration templates for different Canton project types.

**How to use them:** Copy the example that matches your project into your project root as `CLAUDE.md`:

```bash
# For a standard DAML project
cp ~/claude-skills-canton/examples/canton-daml-CLAUDE.md my-project/CLAUDE.md

# For a DeFi / tokenization project using DAML Finance
cp ~/claude-skills-canton/examples/defi-canton-CLAUDE.md my-project/CLAUDE.md

# For an enterprise multi-org Canton deployment
cp ~/claude-skills-canton/examples/enterprise-canton-CLAUDE.md my-project/CLAUDE.md
```

### Contexts (`contexts/*.md`)

**What they are:** Mode-switching prompts that set Claude's focus for a session.

**How to use them:** Reference a context when you want Claude to operate in a specific mode:

```
You: "Switch to review context" (or paste contexts/review.md)
Claude: Enters code review mode — focuses on authorization, privacy, validation

You: "Switch to ops context"
Claude: Enters operations mode — focuses on deployment, monitoring, troubleshooting
```

---

## All Skills (27)

### DAML Smart Contract Language (7 skills)

| # | Skill | Directory | What It Covers |
|---|-------|-----------|---------------|
| 1 | **DAML Templates** | `skills/daml-templates/` | Template structure, `with` clauses, signatories, observers, `ensure` clauses, `agreement` text, data types vs templates |
| 2 | **DAML Choices** | `skills/daml-choices/` | Consuming, non-consuming, pre/post-consuming choices, controllers, authorization rules, exercise patterns (`exercise`, `exerciseByKey`, `createAndExercise`) |
| 3 | **DAML Interfaces** | `skills/daml-interfaces/` | Interface definitions, `viewtype`, `interface instance`, type conversions (`toInterfaceContractId`, `fromInterfaceContractId`), interface inheritance with `requires` |
| 4 | **DAML Script Testing** | `skills/daml-script-testing/` | `script do`, `allocateParty`, `submit`/`submitMustFail`/`submitMulti`/`submitTree`, `query`/`queryContractId`/`queryContractKey`, time management, test patterns |
| 5 | **DAML Project Structure** | `skills/daml-project-structure/` | `daml.yaml` configuration, `dependencies` vs `data-dependencies`, module organization, multi-package builds, CLI commands (`daml new`, `daml init`, `daml build`) |
| 6 | **DAML Design Patterns** | `skills/daml-patterns/` | Propose-accept, delegation, role contracts, state machines, controller delegation, master-subcontract, privacy intermediary, idempotent create, authorization accumulation, anti-patterns |
| 7 | **DAML Finance Patterns** | `skills/daml-finance-patterns/` | Holdings, accounts, instruments, `InstrumentKey`/`AccountKey`, fungible split/merge, settlement (DvP), lifecycle events, disclosure, lockable, DAML Finance style conventions |

### Canton Network Architecture (5 skills)

| # | Skill | Directory | What It Covers |
|---|-------|-----------|---------------|
| 8 | **Canton Architecture** | `skills/canton-architecture/` | Participant nodes, synchronizers (domains), sequencers, mediators, topology manager, identity model (namespaces, unique identifiers), synchronization protocol (7-step commit), HOCON configuration |
| 9 | **Canton Local Dev** | `skills/canton-local-dev/` | `daml sandbox`, `daml start`, full Canton multi-node config, Canton console bootstrap, development iteration cycle, local Postgres setup, Navigator, JSON API server |
| 10 | **Canton Deployment** | `skills/canton-deployment/` | Docker Compose, Postgres storage config, TLS + JWT auth, Prometheus monitoring, health checks, high availability (active-passive, BFT sequencer), operational runbook, security checklist |
| 11 | **Canton Privacy** | `skills/canton-privacy/` | Sub-transaction privacy, visibility rules (who sees what), stakeholder roles, projection, divulgence, privacy-aware design patterns (intermediary, minimal observers, choice observers), GDPR pruning |
| 12 | **Canton Console** | `skills/canton-console/` | Scala REPL commands, node references, health/status, domain connectivity, DAR upload, party allocation, topology management, Ledger API access from console, bootstrap scripts |

### Ledger API & Client Integration (6 skills)

| # | Skill | Directory | What It Covers |
|---|-------|-----------|---------------|
| 13 | **Ledger API Basics** | `skills/ledger-api-basics/` | gRPC services (9 services), JWT authentication, connection setup, key concepts (command ID, workflow ID, offsets), command types (Create, Exercise, ExerciseByKey), error codes, JSON API vs gRPC |
| 14 | **Ledger API Commands** | `skills/ledger-api-commands/` | Synchronous (SubmitAndWait) and async (Submit + CompletionStream) submission, deduplication, error recovery with retries, batch commands, Java + TypeScript + curl examples |
| 15 | **Ledger API Streaming** | `skills/ledger-api-streaming/` | Flat vs tree transactions, ACS + stream bootstrap pattern, offset management, crash recovery, filtering (by party, by template), reconnection handling, Java + TypeScript examples |
| 16 | **Ledger API Queries** | `skills/ledger-api-queries/` | ACS queries (gRPC + JSON API), filtering, contract key lookups, React hooks (`useQuery`, `useStreamQuery`, `useFetchByKey`), building read models, JSON API query operators |
| 17 | **Java Bindings** | `skills/java-bindings/` | Maven setup, `daml codegen java`, `DamlLedgerClient`, command submission with generated classes, RxJava transaction streams, ACS bootstrap, bot pattern, authentication |
| 18 | **TypeScript Bindings** | `skills/typescript-bindings/` | `@daml/types`, `@daml/ledger`, `@daml/react`, `daml codegen js`, JSON API endpoints, React hooks (`DamlLedger` provider, `useQuery`, `useStreamQuery`, `useLedger`), direct fetch/axios, dev JWT tokens |

### Contract & Transaction Model (4 skills)

| # | Skill | Directory | What It Covers |
|---|-------|-----------|---------------|
| 19 | **Contract Keys** | `skills/contract-keys/` | Key definition, maintainers, uniqueness constraints, `fetchByKey`/`lookupByKey`/`exerciseByKey`, visibility rules, idempotent creation, singleton pattern, key migration, contention, cross-domain considerations |
| 20 | **Contract Lifecycle** | `skills/contract-lifecycle/` | Create, active, archived states, functional update pattern (archive + recreate), fetching (by ID, by key), contract ID tracking, multi-step lifecycle (state machines), versioning and upgrades |
| 21 | **Multi-Party Workflows** | `skills/multi-party-workflows/` | Authorization model, propose-accept, N-party sequential approval, DvP, role-based delegation, cross-participant coordination, authority accumulation, testing multi-party flows |
| 22 | **Transaction Model** | `skills/transaction-model/` | Transaction trees vs flat, action types (create, exercise, fetch, lookup), consuming vs non-consuming, rollback nodes (try-catch), atomicity, transaction/event/contract IDs, inspection via Script and API |

### Development & Operations (5 skills)

| # | Skill | Directory | What It Covers |
|---|-------|-----------|---------------|
| 23 | **DAR Management** | `skills/dar-management/` | `daml build`, DAR contents, deployment (CLI, console, gRPC), package versioning, upgrade strategies (side-by-side, interface-based, Daml upgrade mechanism), multi-participant deployment |
| 24 | **Party Management** | `skills/party-management/` | Allocation (CLI, console, Script, API), party ID format (`hint::namespace`), party-to-participant mapping, participant permissions, design patterns (per-user, role-based, service, public party) |
| 25 | **Security Review** | `skills/security-review/` | Contract security checklist (authorization, privacy, validation, keys, state machine), 5 common vulnerability patterns with code examples, Canton deployment security checklist, security testing patterns |
| 26 | **Testing Workflow** | `skills/testing-workflow/` | TDD cycle for DAML, test structure (arrange-act-assert), test helpers, visibility testing, workflow testing, time-dependent testing, test organization, CI pipeline (GitHub Actions) |
| 27 | **Debugging Canton** | `skills/debugging-canton/` | Error code system, log analysis, transaction inspection (Script, API, console), common issues (contract not found, auth failed, timeout, contention, connectivity), performance debugging, ACS monitoring |

---

## All Commands (10)

| # | Command | File | What It Does |
|---|---------|------|-------------|
| 1 | `/daml-new` | `commands/daml-new.md` | Scaffolds a new DAML project — creates directory, `daml.yaml`, initial template, test file, builds and tests |
| 2 | `/daml-build` | `commands/daml-build.md` | Compiles DAML source to DAR, analyzes errors, reports output path |
| 3 | `/daml-test` | `commands/daml-test.md` | Runs `daml test --show-trace`, parses results, suggests fixes for failures |
| 4 | `/canton-start` | `commands/canton-start.md` | Starts local Canton sandbox or multi-node network, verifies health |
| 5 | `/canton-deploy` | `commands/canton-deploy.md` | Uploads DAR to running Canton participant(s), handles auth |
| 6 | `/canton-status` | `commands/canton-status.md` | Checks node health, domain connectivity, loaded packages, allocated parties |
| 7 | `/ledger-query` | `commands/ledger-query.md` | Queries active contracts via JSON API or DAML Script |
| 8 | `/party-create` | `commands/party-create.md` | Allocates a new party on a Canton participant |
| 9 | `/daml-review` | `commands/daml-review.md` | Reviews DAML code against authorization, privacy, validation, design, and testing checklist |
| 10 | `/tx-inspect` | `commands/tx-inspect.md` | Inspects a transaction tree — events, causality, party visibility projections |

---

## All Agents (6)

| # | Agent | File | Model | Tools | Role |
|---|-------|------|-------|-------|------|
| 1 | **daml-architect** | `agents/daml-architect.md` | opus | Read, Grep, Glob | Designs multi-party DAML data models, authorization flows, privacy-aware contract structures, interface hierarchies |
| 2 | **daml-reviewer** | `agents/daml-reviewer.md` | opus | Read, Grep, Glob | Reviews DAML templates for correctness, security, privacy, design quality, and test coverage |
| 3 | **canton-ops** | `agents/canton-ops.md` | opus | Read, Grep, Glob, Bash | Canton deployment, node management, DAR deployment, health checks, log analysis, troubleshooting |
| 4 | **ledger-debugger** | `agents/ledger-debugger.md` | opus | Read, Grep, Glob, Bash | Debugs Ledger API failures — classifies errors, investigates root causes, provides fixes |
| 5 | **security-auditor** | `agents/security-auditor.md` | opus | Read, Grep, Glob | Audits DAML contracts for authorization bypass, privacy leaks, missing validation; audits Canton infrastructure security |
| 6 | **test-runner** | `agents/test-runner.md` | sonnet | Read, Grep, Glob, Bash | Executes `daml test`, parses results, diagnoses failures, suggests fixes |

---

## All Rules (3)

Rules are **automatically applied** when Claude Code edits matching files (after installing to `~/.claude/rules/`).

| # | Rule | File | Glob Pattern | What It Enforces |
|---|------|------|-------------|------------------|
| 1 | **DAML Coding Standards** | `rules/daml-coding-standards.md` | `**/*.daml` | 2-space indent, 100-char lines, qualified imports, CamelCase types, camelCase vars, minimal signatories/observers, ensure clauses, `pure` over `return`, DAML Finance conventions |
| 2 | **DAML Security** | `rules/daml-security.md` | `**/*.daml` | No generic action choices, verify flexible controllers, propose-accept for multi-sig, minimal observers, validate numerics, business-identifier keys |
| 3 | **Canton Config Standards** | `rules/canton-config-standards.md` | `**/*.conf` | HOCON format, env vars for secrets, separate DBs per node, TLS in production, Admin API on localhost, Prometheus monitoring, descriptive node names |

---

## Example CLAUDE.md Configs (3)

Copy the one that matches your project type into your project root as `CLAUDE.md`.

| # | Example | File | Use Case |
|---|---------|------|----------|
| 1 | **Standard DAML Project** | `examples/canton-daml-CLAUDE.md` | General DAML + Canton project with sandbox development |
| 2 | **DeFi / Tokenization** | `examples/defi-canton-CLAUDE.md` | DAML Finance library, holdings, settlement, lifecycle events |
| 3 | **Enterprise Multi-Org** | `examples/enterprise-canton-CLAUDE.md` | Multi-participant, multi-synchronizer, privacy between organizations |

---

## Contexts (3)

Contexts set Claude's focus mode for a session.

| # | Context | File | Focus |
|---|---------|------|-------|
| 1 | **Development** | `contexts/dev.md` | Writing DAML, running tests, local sandbox, fast iteration |
| 2 | **Code Review** | `contexts/review.md` | Authorization correctness, privacy, validation, design quality |
| 3 | **Operations** | `contexts/ops.md` | Node health, DAR deployment, party management, troubleshooting |

---

## How to Use in Your Project

### Step 1: Install Rules (One-Time)

```bash
mkdir -p ~/.claude/rules
cp ~/claude-skills-canton/rules/*.md ~/.claude/rules/
```

### Step 2: Create Your Project's CLAUDE.md

In your DAML project root, create a `CLAUDE.md`:

```markdown
# My Canton Project

## Overview
Bond issuance and trading platform on Canton Network.

## Tech Stack
- DAML 2.10.0 (smart contracts)
- Canton (distributed ledger)
- Java (backend automation)
- React + TypeScript (frontend)

## Canton Skills
When working on DAML contracts, reference: ~/claude-skills-canton/skills/

Key skills:
- daml-templates, daml-choices, daml-interfaces
- daml-patterns (propose-accept for all multi-party contracts)
- daml-finance-patterns (holdings, accounts, instruments)
- canton-local-dev (sandbox workflow)
- testing-workflow (TDD for all templates)
- ledger-api-commands, typescript-bindings

## Project Rules
- Every template must have an `ensure` clause
- Every template must have DAML Script tests
- Use propose-accept for all multi-signatory contracts
- Use contract keys with ISIN/account numbers (business identifiers)
- Minimal observer sets (privacy by design)

## Development Workflow
1. Edit DAML in `daml/` directory
2. Build: `daml build`
3. Test: `daml test --show-trace`
4. Sandbox: `daml start` (port 6865 + JSON API on 7575)
```

### Step 3: Use Claude Code

Now when you work with Claude Code in your project:

```
You: "Design a bond issuance workflow with issuer, custodian, and investor parties"
Claude: [Reads daml-architect + daml-patterns + daml-finance-patterns skills]
        [Designs templates with proper signatories, propose-accept, privacy]

You: "Write tests for the BondIssuance template"
Claude: [Reads daml-script-testing + testing-workflow skills]
        [Writes comprehensive DAML Script tests with happy + failure paths]

You: "Review my DAML code for security issues"
Claude: [Delegates to security-auditor agent]
        [Systematic audit with findings rated Critical/High/Medium/Low]

You: "Deploy the DAR to the sandbox"
Claude: [Reads canton-deploy command]
        [Runs daml build, then daml ledger upload-dar]

You: "Debug: my Transfer choice is failing with NOT_FOUND"
Claude: [Reads debugging-canton skill]
        [Checks if contract is still active, verifies party visibility, checks DAR]
```

---

## Typical Developer Workflow

```
                    ┌──────────────────────────────────┐
                    │         DEVELOPMENT LOOP          │
                    └──────────────────────────────────┘

  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ Write DAML  │───>│ daml build  │───>│  daml test   │
  │ templates   │    │ (compile)   │    │ (run tests)  │
  └─────────────┘    └─────────────┘    └──────┬───────┘
        ^                                       │
        │                                       │ tests pass?
        │ fix                                   │
        │                                       v
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ Debug with  │<───│ Submit cmds │<───│ Start sandbox│
  │ tx-inspect  │    │ via API     │    │ + deploy DAR │
  └─────────────┘    └─────────────┘    └──────────────┘

  Skills used:        Skills used:        Skills used:
  - debugging-canton  - ledger-api-cmds   - canton-local-dev
  - transaction-model - java-bindings     - dar-management
                      - ts-bindings       - party-management
```

---

## Repository Structure

```
claude-skills-canton/
├── README.md                          # This file
├── CLAUDE.md                          # Repo-level Claude config
├── .gitignore
│
├── skills/                            # 27 domain knowledge files
│   ├── daml-templates/SKILL.md        # Template authoring
│   ├── daml-choices/SKILL.md          # Choice design
│   ├── daml-interfaces/SKILL.md       # Interfaces
│   ├── daml-script-testing/SKILL.md   # DAML Script testing
│   ├── daml-project-structure/SKILL.md # daml.yaml, modules
│   ├── daml-patterns/SKILL.md         # Design patterns
│   ├── daml-finance-patterns/SKILL.md # DAML Finance library
│   ├── canton-architecture/SKILL.md   # Canton architecture
│   ├── canton-local-dev/SKILL.md      # Local development
│   ├── canton-deployment/SKILL.md     # Production deployment
│   ├── canton-privacy/SKILL.md        # Privacy model
│   ├── canton-console/SKILL.md        # Console operations
│   ├── ledger-api-basics/SKILL.md     # API fundamentals
│   ├── ledger-api-commands/SKILL.md   # Command submission
│   ├── ledger-api-streaming/SKILL.md  # Transaction streams
│   ├── ledger-api-queries/SKILL.md    # ACS queries
│   ├── java-bindings/SKILL.md         # Java client
│   ├── typescript-bindings/SKILL.md   # TypeScript client
│   ├── contract-keys/SKILL.md         # Contract keys
│   ├── contract-lifecycle/SKILL.md    # Contract lifecycle
│   ├── multi-party-workflows/SKILL.md # Multi-party auth
│   ├── transaction-model/SKILL.md     # Transaction trees
│   ├── dar-management/SKILL.md        # DAR build/deploy
│   ├── party-management/SKILL.md      # Party allocation
│   ├── security-review/SKILL.md       # Security audit
│   ├── testing-workflow/SKILL.md      # TDD workflow
│   └── debugging-canton/SKILL.md      # Debugging
│
├── commands/                          # 10 slash commands
│   ├── daml-new.md
│   ├── daml-build.md
│   ├── daml-test.md
│   ├── canton-start.md
│   ├── canton-deploy.md
│   ├── canton-status.md
│   ├── ledger-query.md
│   ├── party-create.md
│   ├── daml-review.md
│   └── tx-inspect.md
│
├── agents/                            # 6 specialized subagents
│   ├── daml-architect.md
│   ├── daml-reviewer.md
│   ├── canton-ops.md
│   ├── ledger-debugger.md
│   ├── security-auditor.md
│   └── test-runner.md
│
├── rules/                             # 3 always-follow rule sets
│   ├── daml-coding-standards.md
│   ├── daml-security.md
│   └── canton-config-standards.md
│
├── examples/                          # 3 example CLAUDE.md configs
│   ├── canton-daml-CLAUDE.md
│   ├── defi-canton-CLAUDE.md
│   └── enterprise-canton-CLAUDE.md
│
└── contexts/                          # 3 prompt contexts
    ├── dev.md
    ├── review.md
    └── ops.md
```

---

## Contributing

PRs welcome. When adding a new skill:

1. Create a directory under `skills/` with a descriptive kebab-case name
2. Add a `SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: my-skill-name
   description: One-line description of what this skill covers.
   ---
   ```
3. Include:
   - "When to Activate" section
   - Code examples with DAML/Canton syntax
   - Best practices
   - Common pitfalls
4. Update this README with the new skill in the appropriate table
5. Run through the daml-review checklist if adding DAML examples

---

## License

MIT

---

## Links

- [DAML Documentation](https://docs.daml.com)
- [Canton Network](https://www.canton.network)
- [Canton GitHub](https://github.com/digital-asset/canton)
- [DAML GitHub](https://github.com/digital-asset/daml)
- [DAML Finance](https://github.com/digital-asset/daml-finance)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code) (inspiration)
