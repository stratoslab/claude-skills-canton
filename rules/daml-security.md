---
description: Security rules that must always be followed when writing DAML contracts.
globs: ["**/*.daml"]
---

# DAML Security Rules

## Authorization
- NEVER create choices with generic `Update ()` action parameters — this allows arbitrary code execution
- ALWAYS verify the actor is authorized in flexible controller patterns
- NEVER allow a non-signatory party to unilaterally create multi-signatory contracts
- ALWAYS use propose-accept for gathering multi-party authorization

## Privacy
- NEVER add parties as observers unless they have a business need to see the contract
- NEVER put sensitive data (prices, amounts, personal info) in contracts visible to unauthorized parties
- ALWAYS separate contracts by confidentiality level when different parties need different views

## Validation
- ALWAYS validate numeric values (positive quantities, valid ranges) in ensure clauses
- ALWAYS validate choice arguments before acting on them
- NEVER trust external input without validation

## Contract Keys
- ALWAYS use business identifiers, not synthetic IDs
- ALWAYS ensure maintainers are signatories
- Be aware that key uniqueness is per-synchronizer in Canton
