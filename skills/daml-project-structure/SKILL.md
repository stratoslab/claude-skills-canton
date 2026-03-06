---
name: daml-project-structure
description: DAML project layout — daml.yaml configuration, module organization, dependencies, data-dependencies, and build options.
---

# DAML Project Structure

Every DAML project is defined by a `daml.yaml` file at its root. Understanding project structure is essential for building, testing, and deploying Canton applications.

## When to Activate

- Creating a new DAML project
- Configuring dependencies and SDK versions
- Organizing modules across packages
- Setting up multi-package builds
- Troubleshooting build errors

## Project Layout

```
my-project/
  daml.yaml              # Project configuration
  daml/                   # DAML source directory
    Main.daml             # Primary module
    Types.daml            # Shared data types
    Workflow/
      Transfer.daml       # Workflow-specific templates
      Settlement.daml
  test/
    TestMain.daml         # Test scripts
  .daml/                  # Build artifacts (gitignored)
    dist/
      my-project-1.0.0.dar  # Compiled DAR
```

## daml.yaml Configuration

```yaml
sdk-version: 2.10.0
name: my-project
version: 1.0.0
source: daml

# Standard library dependencies
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script-lts      # For DAML Script testing

# Pre-compiled DAR dependencies (cross-package references)
data-dependencies:
  - ../shared-types/.daml/dist/shared-types-1.0.0.dar
  - .lib/daml-finance/daml-finance-1.6.0.dar

# Compiler options
build-options:
  - --target=2.1          # LF version target
  - -Wno-upgrade-interfaces

# Sandbox configuration
sandbox-options:
  - --wall-clock-time

# Navigator (deprecated but still available)
start-navigator: false
```

## Key Configuration Fields

### sdk-version
The DAML SDK version to use. Must match your installed SDK.
```yaml
sdk-version: 2.10.0
```

### source
Directory containing DAML source files. Defaults to `daml/`.
```yaml
source: daml
# Or for tests:
source: test
```

### dependencies vs data-dependencies

**dependencies** — Source-level dependencies (daml-prim, daml-stdlib). You get full access to all types and can pattern match on them.

**data-dependencies** — DAR-level dependencies (compiled packages). You can use types and templates but cannot pattern match on their constructors. Used for cross-package references.

```yaml
dependencies:
  - daml-prim
  - daml-stdlib

data-dependencies:
  - ./lib/other-package.dar
```

### build-options
Passed directly to the DAML compiler.

```yaml
build-options:
  - --target=2.1           # Target LF version
  - --ghc-option=-Werror   # Treat warnings as errors
  - --include=src/main/daml  # Additional source directories
  - -Wno-upgrade-interfaces  # Suppress interface upgrade warnings
```

## Module Organization

### Module Naming
```daml
module MyProject.Workflow.Transfer where
-- File: daml/MyProject/Workflow/Transfer.daml
```

### Module Hierarchy Best Practice
```
daml/
  Main.daml                    -- module Main (re-exports)
  Types.daml                   -- module Types (shared data types)
  Workflow/
    Issuance.daml              -- module Workflow.Issuance
    Transfer.daml              -- module Workflow.Transfer
    Settlement.daml            -- module Workflow.Settlement
  Interface/
    Asset.daml                 -- module Interface.Asset
    Holding.daml               -- module Interface.Holding
  Role/
    Operator.daml              -- module Role.Operator
    Custodian.daml             -- module Role.Custodian
```

### Imports
```daml
-- Qualified import (preferred for interfaces)
import Daml.Finance.Interface.Holding.V4 qualified as Holding

-- Explicit imports
import DA.List (head, sortOn)
import DA.Optional (fromSome)
import DA.Map qualified as Map (fromList, lookup)
```

## Multi-Package Projects

For larger projects, split into multiple DARs:

```
project-root/
  common/
    daml.yaml              # name: common
    daml/
      Common/Types.daml
  workflows/
    daml.yaml              # name: workflows, data-dependencies: [../common/.daml/dist/common-1.0.0.dar]
    daml/
      Workflow/Transfer.daml
  app/
    daml.yaml              # name: app, data-dependencies: [../workflows/..., ../common/...]
    daml/
      Main.daml
```

Build order matters: `common` -> `workflows` -> `app`.

## CLI Commands

```bash
# Create new project
daml new my-project
daml new my-project --template=skeleton

# Initialize daml.yaml in existing directory
daml init

# Build (produces .dar in .daml/dist/)
daml build

# Clean build artifacts
daml clean

# Run tests
daml test

# Start everything (sandbox + navigator + JSON API)
daml start
```

## Best Practices

1. **Pin sdk-version** — Always specify exact version, not ranges
2. **Separate test source** — Keep test scripts in a `test/` directory
3. **Use data-dependencies for shared packages** — Enables independent versioning
4. **Gitignore .daml/** — Build artifacts should not be committed
5. **One concept per module** — Keep modules focused and cohesive
6. **Use qualified imports** — Especially for interface modules to avoid name clashes

## Common Pitfalls

- **SDK version mismatch** — daml.yaml version must match installed SDK
- **Circular data-dependencies** — Package A cannot depend on B if B depends on A
- **Missing daml-prim/daml-stdlib** — These are always required in dependencies
- **Wrong source directory** — Ensure `source:` points to the right directory
- **LF version incompatibility** — data-dependencies must target compatible LF versions
