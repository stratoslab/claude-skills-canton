# Development Context

You are working on a DAML smart contract project targeting the Canton Network.

## Active Mode: Development

### Focus Areas
- Writing and iterating on DAML templates
- Running tests with `daml test --show-trace`
- Local sandbox for rapid feedback
- Code review and refactoring

### Available Tools
- `daml build` — Compile DAML to DAR
- `daml test` — Run DAML Script tests
- `daml start` — Start sandbox + JSON API
- `daml sandbox` — Start sandbox only

### Workflow
1. Edit DAML source files
2. Build and test: `daml build && daml test --show-trace`
3. Deploy to sandbox: `daml ledger upload-dar .daml/dist/*.dar`
4. Test via JSON API or DAML Script

### Standards
- Follow daml-coding-standards rules
- Follow daml-security rules
- Test every template before committing
