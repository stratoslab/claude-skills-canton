---
name: daml-templates
description: DAML template authoring — signatories, observers, ensure clauses, key definitions, and agreement text. Core building block for all Canton smart contracts.
---

# DAML Template Authoring

Templates are the fundamental building block of DAML. Every contract on the ledger is an instance of a template.

## When to Activate

- Creating new smart contracts
- Designing data models for Canton applications
- Modeling agreements between parties
- Defining on-ledger state

## Template Structure

```daml
module MyModule where

import Daml.Script
import DA.Text (isInfixOf)

template Asset
  with
    issuer : Party
    owner : Party
    description : Text
    quantity : Decimal
  where
    signatory issuer
    observer owner

    ensure quantity > 0.0

    key (issuer, description) : (Party, Text)
    maintainer key._1

    choice Transfer : ContractId Asset
      with
        newOwner : Party
      controller owner
      do
        create this with owner = newOwner

    choice Split : (ContractId Asset, ContractId Asset)
      with
        splitQuantity : Decimal
      controller owner
      do
        assert (splitQuantity > 0.0 && splitQuantity < quantity)
        cid1 <- create this with quantity = splitQuantity
        cid2 <- create this with quantity = quantity - splitQuantity
        return (cid1, cid2)

    nonconsuming choice GetInfo : (Party, Text, Decimal)
      controller owner
      do
        return (issuer, description, quantity)
```

## Key Concepts

### Signatories
- Parties who **authorize** the creation and existence of the contract
- Must consent to contract creation (their authority is needed)
- At least one signatory is required
- Signatories can always see the contract

```daml
signatory issuer          -- single signatory
signatory issuer, owner   -- multiple signatories (both must authorize)
```

### Observers
- Parties who can **see** the contract but did not authorize it
- Cannot prevent creation or archival
- Useful for read access, notifications

```daml
observer owner                    -- single observer
observer owner, regulator         -- multiple observers
observer if isPublic then [public] else []  -- conditional observers
```

### Ensure Clause
- Boolean predicate that must hold for the contract to be valid
- Checked at creation time
- Use for data invariants

```daml
ensure quantity > 0.0
    && description /= ""
    && issuer /= owner
```

### Contract Keys
- Unique identifier for looking up contracts without ContractId
- Must include at least one signatory party in the key
- Maintainer must be a subset of signatories

```daml
key (issuer, description) : (Party, Text)
maintainer key._1    -- issuer maintains the key
```

### Agreement Text (Optional)
```daml
agreement show issuer <> " issues " <> show quantity <> " of " <> description <> " to " <> show owner
```

## Best Practices

1. **Minimal signatories** — Only parties who truly need to authorize
2. **Validate in ensure** — Catch invalid state at creation, not in choices
3. **Meaningful keys** — Use business identifiers, not synthetic IDs
4. **Keep templates focused** — One concept per template
5. **Use records for complex data** — Extract data types for clarity

```daml
data AssetDetails = AssetDetails
  with
    description : Text
    category : Text
    metadata : [(Text, Text)]
  deriving (Eq, Show)

template Asset
  with
    issuer : Party
    owner : Party
    details : AssetDetails
    quantity : Decimal
  where
    signatory issuer
    observer owner
    ensure quantity > 0.0
```

## Common Pitfalls

- **Missing signatory** — Every template needs at least one
- **Key without maintainer** — Keys require an explicit maintainer
- **Ensure not checked on exercise** — Only checked at creation
- **Circular dependencies** — Templates in the same module can reference each other, but cross-module cycles are not allowed
- **Forgetting `with` keyword** — Template fields and choice arguments both use `with`

## Template vs Data Type

Use templates for **on-ledger state** (agreements, assets, rights).
Use data types for **off-ledger data** (parameters, return values, nested records).
