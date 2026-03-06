# Claude Skills for Canton Network & DAML

This repository contains Claude Code skills, commands, agents, and rules for developing on the Canton Network with DAML smart contracts.

## Repository Structure
- `skills/` — 27 domain workflow definitions covering DAML, Canton, and Ledger API
- `commands/` — 10 slash commands for common development tasks
- `agents/` — 6 specialized subagents for architecture, review, ops, debugging, security, and testing
- `rules/` — 3 always-follow guideline sets (coding standards, config standards, security)
- `examples/` — 3 example CLAUDE.md configurations for different project types
- `contexts/` — 3 dynamic prompt contexts (dev, review, ops)

## Key Conventions
- DAML code: 2-space indent, 100 char lines, qualified imports for interfaces
- Canton config: HOCON format, env vars for secrets, separate DBs per node
- Security: minimal signatories/observers, propose-accept for multi-party, validate all inputs
- Testing: DAML Script tests for all templates, both success and failure paths
