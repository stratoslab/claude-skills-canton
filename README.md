# Claude Skills for Canton Network & DAML

A comprehensive Claude Code skill set for building on the **Canton Network** and writing **DAML smart contracts**. Inspired by [everything-claude-code](https://github.com/affaan-m/everything-claude-code), tailored specifically for Canton/DAML developers.

## What This Is

A collection of **skills**, **agents**, **commands**, and **rules** that make Claude Code deeply effective for Canton Network development — from writing DAML templates to deploying DARs, querying the Ledger API, and managing multi-party workflows.

## Repository Structure

```
claude-skills-canton/
├── skills/                  # 27 domain workflow definitions
│   ├── daml-templates/      # Template authoring patterns
│   ├── daml-choices/        # Choice design and exercise patterns
│   ├── daml-interfaces/     # Interface definitions and implementation
│   ├── daml-script-testing/ # DAML Script for testing
│   ├── daml-project-structure/ # daml.yaml, module layout
│   ├── daml-patterns/       # Advanced DAML design patterns
│   ├── daml-finance-patterns/ # DAML Finance library patterns
│   ├── canton-architecture/ # Canton architecture concepts
│   ├── canton-local-dev/    # Sandbox and local Canton setup
│   ├── canton-deployment/   # Production deployment patterns
│   ├── canton-privacy/      # Privacy and sub-transaction visibility
│   ├── canton-console/      # Canton console operations
│   ├── ledger-api-basics/   # Ledger API fundamentals
│   ├── ledger-api-streaming/# Transaction stream subscriptions
│   ├── ledger-api-queries/  # Active contract set queries
│   ├── ledger-api-commands/ # Command submission patterns
│   ├── java-bindings/       # Java client bindings
│   ├── typescript-bindings/ # TypeScript/JavaScript bindings
│   ├── contract-keys/       # Contract key design
│   ├── contract-lifecycle/  # Create, exercise, archive patterns
│   ├── multi-party-workflows/ # Multi-party authorization
│   ├── transaction-model/   # Transaction trees and flat transactions
│   ├── dar-management/      # DAR build, deploy, versioning
│   ├── party-management/    # Party allocation and identity
│   ├── security-review/     # Canton/DAML security patterns
│   ├── testing-workflow/    # TDD for DAML contracts
│   └── debugging-canton/    # Debugging Canton nodes and contracts
├── commands/                # Slash commands for quick actions
├── agents/                  # Specialized subagents
├── rules/                   # Always-follow DAML/Canton guidelines
├── examples/                # Example CLAUDE.md configurations
└── contexts/                # Dynamic prompt contexts
```

## Installation

### Manual Setup

```bash
git clone https://github.com/stratoslab/claude-skills-canton-.git
cd claude-skills-canton-

# Copy rules to your Claude config
cp rules/*.md ~/.claude/rules/

# Reference skills in your project's CLAUDE.md
```

### Project Integration

Add to your project's `CLAUDE.md`:

```markdown
# Canton/DAML Project

## Skills
Reference: ~/claude-skills-canton/skills/

## Rules
- Follow DAML coding standards from canton-rules
- Use Canton privacy model for all multi-party designs
- Test all templates with DAML Script before deployment
```

## Core Skills

### DAML Smart Contracts
| Skill | Description |
|-------|-------------|
| `daml-templates` | Template structure, signatories, observers, ensure clauses |
| `daml-choices` | Consuming/non-consuming choices, controllers, choice bodies |
| `daml-interfaces` | Interface definitions, viewtypes, implementations |
| `daml-patterns` | Propose-accept, delegation, role contracts, state machines |
| `daml-finance-patterns` | Asset modeling, holdings, accounts, instruments |

### Canton Network
| Skill | Description |
|-------|-------------|
| `canton-architecture` | Participants, domains, synchronization protocol |
| `canton-local-dev` | Sandbox setup, local multi-node Canton |
| `canton-deployment` | Docker, production config, domain connectivity |
| `canton-privacy` | Sub-transaction privacy, divulgence, projection |
| `canton-console` | Canton console commands and administration |

### Ledger API & Integration
| Skill | Description |
|-------|-------------|
| `ledger-api-basics` | gRPC services, authentication, connection |
| `ledger-api-commands` | Command submission, completion tracking |
| `ledger-api-streaming` | Transaction streams, offsets, filters |
| `ledger-api-queries` | Active contract set queries and filters |
| `java-bindings` | Java codegen, reactive streams, bot framework |
| `typescript-bindings` | TypeScript codegen, JSON API |

### Development Workflow
| Skill | Description |
|-------|-------------|
| `testing-workflow` | TDD with DAML Script, property-based testing |
| `dar-management` | Build, deploy, upgrade DARs |
| `party-management` | Party allocation, hints, display names |
| `debugging-canton` | Log analysis, contract inspection, error codes |

## Commands

| Command | Description |
|---------|-------------|
| `/daml-new` | Scaffold a new DAML project |
| `/daml-build` | Compile DAML to DAR |
| `/daml-test` | Run DAML Script tests |
| `/canton-start` | Start local Canton sandbox |
| `/canton-deploy` | Deploy DAR to running Canton |
| `/canton-status` | Check Canton node health |
| `/ledger-query` | Query active contracts |
| `/party-create` | Allocate a new party |
| `/daml-review` | Review DAML code for best practices |
| `/tx-inspect` | Inspect transaction tree |

## Agents

| Agent | Description |
|-------|-------------|
| `daml-architect` | Design multi-party DAML data models |
| `daml-reviewer` | Review DAML templates for correctness and style |
| `canton-ops` | Canton deployment and operational tasks |
| `ledger-debugger` | Debug Ledger API issues and transaction failures |
| `security-auditor` | Audit DAML contracts for authorization vulnerabilities |
| `test-runner` | Execute and validate DAML Script tests |

## Typical Developer Workflow

```
write DAML  -->  compile DAR  -->  start sandbox  -->  deploy DAR
    |                                                       |
    v                                                       v
run tests  <--  query contracts  <--  submit commands  <--  create parties
```

## Contributing

PRs welcome. When adding a new skill:

1. Create a directory under `skills/`
2. Add a `SKILL.md` with frontmatter (name, description)
3. Include code examples with DAML syntax
4. Reference relevant Canton docs
5. Update this README

## License

MIT

## Links

- [DAML Documentation](https://docs.daml.com)
- [Canton Network](https://www.canton.network)
- [Canton GitHub](https://github.com/digital-asset/canton)
- [DAML GitHub](https://github.com/digital-asset/daml)
- [DAML Finance](https://github.com/digital-asset/daml-finance)
