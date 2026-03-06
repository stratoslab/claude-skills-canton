---
description: Review DAML code for best practices, security issues, and common pitfalls.
---

# DAML Code Review

Review DAML source files against the following checklist:

## Authorization
- [ ] Correct signatories (minimal, necessary)
- [ ] Correct controllers for each choice
- [ ] No authority escalation paths
- [ ] Propose-accept for multi-signatory contracts

## Privacy
- [ ] Minimal observer sets
- [ ] No unnecessary data exposure
- [ ] Separate contracts for different visibility needs

## Data Validation
- [ ] Ensure clauses for invariants
- [ ] Assert statements in choice bodies
- [ ] Positive quantity checks
- [ ] Non-empty string checks

## Design
- [ ] Templates are focused (single responsibility)
- [ ] Contract keys use business identifiers
- [ ] Choices return created contract IDs
- [ ] State machine transitions are guarded

## Testing
- [ ] Tests exist for all templates
- [ ] Both happy and failure paths tested
- [ ] Authorization tests present
- [ ] Key uniqueness tests present
