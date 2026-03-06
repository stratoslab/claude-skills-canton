---
name: daml-interfaces
description: DAML interface definitions — viewtypes, interface choices, implementations, and polymorphic contract interaction patterns.
---

# DAML Interfaces

Interfaces enable polymorphic interaction with contracts. Multiple templates can implement the same interface, allowing uniform handling without knowing the concrete type.

## When to Activate

- Designing extensible contract hierarchies
- Building generic workflows that work across template types
- Creating reusable libraries (e.g., DAML Finance)
- Decoupling consumers from specific template implementations

## Interface Definition

```daml
module Interfaces where

data AssetView = AssetView
  with
    owner : Party
    issuer : Party
    quantity : Decimal
  deriving (Eq, Show)

interface IAsset where
  viewtype AssetView

  getOwner : Party
  getIssuer : Party
  getQuantity : Decimal

  choice ITransfer : ContractId IAsset
    with
      newOwner : Party
    controller getOwner this
    do
      abort "must be implemented by template"

  choice ISplit : (ContractId IAsset, ContractId IAsset)
    with
      splitQuantity : Decimal
    controller getOwner this
    do
      abort "must be implemented by template"
```

## Template Implementation

```daml
template Token
  with
    issuer : Party
    owner : Party
    amount : Decimal
    symbol : Text
  where
    signatory issuer
    observer owner

    interface instance IAsset for Token where
      view = AssetView with
        owner
        issuer
        quantity = amount

      getOwner = owner
      getIssuer = issuer
      getQuantity = amount

      ITransfer newOwner = do
        cid <- create this with owner = newOwner
        return (toInterfaceContractId cid)

      ISplit splitQuantity = do
        cid1 <- create this with amount = splitQuantity
        cid2 <- create this with amount = amount - splitQuantity
        return (toInterfaceContractId cid1, toInterfaceContractId cid2)
```

## Using Interfaces

### Exercise via Interface
```daml
-- Exercise on any contract implementing IAsset
exercise (toInterfaceContractId @IAsset tokenCid) ITransfer with newOwner = bob
```

### Query by Interface
```daml
-- Fetch all contracts implementing IAsset
assets <- queryInterface @IAsset alice
```

### Get View
```daml
-- Get the standardized view of any IAsset
let view = view (toInterface @IAsset tokenContract)
-- view.owner, view.issuer, view.quantity
```

### Type Conversion
```daml
-- Template to interface
let iAssetCid : ContractId IAsset = toInterfaceContractId @IAsset tokenCid

-- Interface to template (may fail)
let maybeTokenCid : Optional (ContractId Token) = fromInterfaceContractId @Token iAssetCid

-- Coerce (unsafe, use carefully)
let tokenCid : ContractId Token = coerceInterfaceContractId @Token iAssetCid
```

## Interface Inheritance

```daml
interface IFungible requires IAsset where
  viewtype FungibleView

  choice Merge : ContractId IFungible
    with
      otherCid : ContractId IFungible
    controller (getOwner (toInterface @IAsset this))
    do
      abort "must be implemented"
```

## DAML Finance Interface Pattern

The DAML Finance library uses interfaces extensively:

```
IHolding     -- ownership of assets
IAccount     -- custody accounts
IInstrument  -- financial instruments
ISettlement  -- settlement of transactions
```

## Best Practices

1. **Use viewtype for read access** — Standardize data access across implementations
2. **Keep interfaces minimal** — Only include truly shared behavior
3. **Default implementations can abort** — Templates must override
4. **Prefer interface choices for cross-template workflows** — Enables generic processing
5. **Use `requires` for interface hierarchies** — Build layered abstractions
6. **Document expected semantics** — Interfaces are contracts (in the software sense)

## Common Pitfalls

- **Missing `interface instance`** — Template won't satisfy the interface
- **Forgetting `toInterfaceContractId`** — Type conversion is explicit
- **View mismatch** — Viewtype fields must match what the template provides
- **Circular requires** — Interface A requires B requires A is not allowed
