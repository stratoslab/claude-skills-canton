---
description: Always-follow DAML coding standards for Canton projects.
globs: ["**/*.daml"]
---

# DAML Coding Standards

## Formatting
- 2 spaces indentation
- 100 character line limit
- Use qualified imports for interface modules
- Explicit imports, alphabetically ordered
- Use `pure` instead of `return`

## Naming
- CamelCase for types, templates, choices
- camelCase for variables, functions
- Descriptive names: `transferAsset` not `doIt`
- Choices named as actions: `Transfer`, `Accept`, `Archive`

## Templates
- Minimal signatory sets (only parties who must authorize)
- Minimal observer sets (need-to-know basis)
- Always include `ensure` clause for data invariants
- Use contract keys with business identifiers
- Return created contract IDs from consuming choices

## Choices
- Consuming by default; `nonconsuming` only when contract must survive
- Validate arguments with `assert` or `assertMsg`
- Use propose-accept for multi-signatory contracts
- Never use generic/arbitrary action choices (security risk)

## Testing
- Write DAML Script tests for all templates
- Test both success and failure paths (submit + submitMustFail)
- Test authorization: verify who can and cannot exercise each choice
- Test contract keys: creation, lookup, uniqueness
- Use `===` from DA.Assert for clear failure messages

## Style (DAML Finance Convention)
- Type alias `type I = InterfaceName` for interfaces
- Type alias `type V = View` for viewtypes
- Use `;` for short-hand record construction: `Foo with a; b; c`
- Flexible controllers: `actor : Party` as first choice argument
