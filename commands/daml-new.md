---
description: Scaffold a new DAML project with daml.yaml, source directory, and initial template.
---

# New DAML Project

Create a new DAML project with the correct structure and configuration.

## Steps

1. Ask for the project name and a brief description of the domain
2. Run `daml new <project-name>` to scaffold
3. Customize `daml.yaml` with appropriate settings:
   - Set the correct SDK version
   - Add `daml-script-lts` to dependencies for testing
   - Set `start-navigator: false` (deprecated)
4. Create initial template in `daml/Main.daml` based on the domain description
5. Create a test file in `daml/Test.daml` with basic tests
6. Run `daml build` to verify compilation
7. Run `daml test` to verify tests pass

## Template

```yaml
sdk-version: 2.10.0
name: {{project-name}}
version: 1.0.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script-lts
build-options:
  - --target=2.1
sandbox-options:
  - --wall-clock-time
start-navigator: false
```
