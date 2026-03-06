---
name: contract-lifecycle
description: Contract lifecycle — create, exercise, archive, fetch patterns, and the functional update model (archive + recreate).
---

# Contract Lifecycle

DAML contracts are immutable. Understanding the create → active → archived lifecycle and the functional update pattern is fundamental.

## When to Activate

- Managing contract state transitions
- Implementing update/modify patterns
- Understanding archive semantics
- Designing contract versioning strategies

## Lifecycle States

```
 CREATE          ACTIVE              ARCHIVED
   │               │                    │
   ▼               ▼                    ▼
┌──────┐    ┌────────────┐    ┌──────────────┐
│ New  │───▶│  Can be    │───▶│ Cannot be    │
│      │    │  fetched,  │    │  fetched or  │
│      │    │  exercised │    │  exercised   │
└──────┘    └────────────┘    └──────────────┘
```

## Creating Contracts

```daml
-- In DAML Script
cid <- submit bank do
  createCmd Asset with
    issuer = bank
    owner = alice
    description = "Gold"
    quantity = 100.0

-- In a choice body
choice Issue : ContractId Asset
  with
    owner : Party
    desc : Text
    qty : Decimal
  controller issuer
  do
    create Asset with
      issuer
      owner
      description = desc
      quantity = qty
```

## Archiving Contracts

Contracts are archived by **consuming choices**. Every consuming exercise archives the target contract.

```daml
-- Explicit archive (built-in choice on every template)
submit owner do exerciseCmd assetCid Archive

-- Implicit archive via consuming choice
choice Transfer : ContractId Asset
  with newOwner : Party
  controller owner
  do
    -- 'this' contract is automatically archived
    create this with owner = newOwner
```

## The Functional Update Pattern

Since contracts are immutable, "updating" means **archive the old + create the new**.

```daml
choice UpdateQuantity : ContractId Asset
  with
    newQuantity : Decimal
  controller owner
  do
    -- 'this' is archived (consuming choice)
    -- A new contract is created with updated data
    create this with quantity = newQuantity
```

This is equivalent to:
```
1. Archive Asset(issuer=Bank, owner=Alice, quantity=100)
2. Create  Asset(issuer=Bank, owner=Alice, quantity=200)
```

## Fetching Contracts

### By Contract ID

```daml
-- In choice body
asset <- fetch assetCid          -- Returns the contract payload
(cid, asset) <- fetchByKey @Asset (bank, "Gold")

-- In DAML Script
Some asset <- queryContractId alice assetCid
```

### By Key

```daml
-- Strict (fails if not found)
(cid, asset) <- fetchByKey @Asset (bank, "Gold")

-- Optional (returns None if not found)
result <- lookupByKey @Asset (bank, "Gold")
case result of
  Some cid -> ...
  None -> ...
```

## Contract ID Tracking

Contract IDs change every time a contract is recreated. Callers must track the new ID.

```daml
-- Choice returns the new contract ID
choice Transfer : ContractId Asset
  with newOwner : Party
  controller owner
  do
    newCid <- create this with owner = newOwner
    return newCid  -- Caller gets the new ID

-- Caller tracks the new ID
newAssetCid <- submit alice do
  exerciseCmd oldAssetCid Transfer with newOwner = bob
-- oldAssetCid is now archived; use newAssetCid going forward
```

## Multi-Step Lifecycle

```daml
data Status = Draft | Pending | Active | Completed | Cancelled
  deriving (Eq, Show)

template WorkOrder
  with
    creator : Party
    assignee : Party
    description : Text
    status : Status
  where
    signatory creator
    observer assignee

    choice Submit : ContractId WorkOrder
      controller creator
      do
        assert (status == Draft)
        create this with status = Pending

    choice Approve : ContractId WorkOrder
      controller assignee
      do
        assert (status == Pending)
        create this with status = Active

    choice Complete : ContractId WorkOrder
      controller assignee
      do
        assert (status == Active)
        create this with status = Completed

    choice Cancel : ContractId WorkOrder
      controller creator
      do
        assert (status /= Completed)
        create this with status = Cancelled
```

## Versioning and Upgrades

When template definitions change, you need a migration strategy:

```daml
-- V1
template AssetV1 with
    issuer : Party
    owner : Party
    quantity : Decimal
  where ...

-- V2 (added currency field)
template AssetV2 with
    issuer : Party
    owner : Party
    quantity : Decimal
    currency : Text
  where ...

-- Migration choice on V1
    choice MigrateToV2 : ContractId AssetV2
      controller issuer
      do
        create AssetV2 with
          issuer; owner; quantity
          currency = "USD"  -- Default value
```

## Best Practices

1. **Always return new contract IDs from choices** — Callers need to track them
2. **Use the functional update pattern** — `create this with field = newValue`
3. **Design clear state transitions** — Document valid state changes
4. **Test archival** — Verify archived contracts cannot be exercised
5. **Handle contract-not-found** — Use `lookupByKey` for graceful handling
6. **Plan for migrations** — Design upgrade paths before deploying

## Common Pitfalls

- **Using old contract ID after exercise** — The contract was archived; use the new ID
- **Trying to update in place** — DAML contracts are immutable
- **Forgetting about observers on recreate** — New contract may have different visibility
- **Double-spend** — Exercising an already-archived contract fails (this is a feature)
