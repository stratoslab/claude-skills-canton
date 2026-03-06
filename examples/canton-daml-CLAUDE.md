# Canton DAML Project

## Project Overview
This is a Canton Network application using DAML smart contracts.

## Tech Stack
- DAML (smart contracts)
- Canton (distributed ledger)
- Java / TypeScript (client applications)
- Postgres (persistent storage)
- Docker (deployment)

## Skills Reference
Reference: ~/claude-skills-canton/skills/

## Key Skills for This Project
- daml-templates — Template authoring patterns
- daml-choices — Choice design
- daml-patterns — Propose-accept, delegation, state machines
- canton-local-dev — Local development setup
- testing-workflow — TDD for DAML
- ledger-api-basics — Client integration

## Rules
- Follow DAML coding standards (2-space indent, 100 char lines, qualified imports)
- Use propose-accept for all multi-signatory contracts
- Minimal signatories and observers (privacy by design)
- Validate all data in ensure clauses
- Test all templates with DAML Script before deployment
- Never hardcode credentials in Canton config files
- Use contract keys with business identifiers

## Development Workflow
1. Write DAML templates in `daml/` directory
2. Write tests in `daml/Test/` directory
3. Build: `daml build`
4. Test: `daml test --show-trace`
5. Start sandbox: `daml start`
6. Deploy DAR: `daml ledger upload-dar .daml/dist/*.dar`
7. Allocate parties: `daml ledger allocate-parties Alice Bob`

## Project Structure
```
daml/               # DAML source files
  Main.daml          # Primary templates
  Test/              # Test scripts
daml.yaml            # Project configuration
canton/              # Canton configuration (if full Canton)
  config.conf        # Node topology
  bootstrap.canton   # Bootstrap script
```

## Canton Configuration
- Sandbox for development: `daml start` (port 6865)
- JSON API for frontend: port 7575
- Full Canton for multi-participant testing
