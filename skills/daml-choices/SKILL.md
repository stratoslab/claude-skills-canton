---
name: daml-choices
description: DAML choice design — consuming, non-consuming, pre-consuming, post-consuming choices, controllers, authorization, and exercise patterns.
---

# DAML Choice Design

Choices define the actions that can be performed on a contract. They are the "API" of your smart contract.

## When to Activate

- Adding behavior to templates
- Designing state transitions
- Implementing multi-party authorization flows
- Building workflow steps

## Choice Types

### Consuming Choice (Default)
Archives the contract when exercised. The contract can no longer be used.

```daml
choice Accept : ContractId Agreement
  controller receiver
  do
    create Agreement with ..
```

### Non-consuming Choice
Contract remains active after exercise. Use for queries, reads, notifications.

```daml
nonconsuming choice GetBalance : Decimal
  controller owner
  do
    return balance
```

### Pre-consuming Choice
Archives the contract **before** the choice body executes. Useful when the body may fail and you still want the contract archived.

```daml
preconsuming choice Finalize : ()
  controller admin
  do
    -- contract already archived here
    doSomethingThatMightFail
```

### Post-consuming Choice
Archives the contract **after** the choice body executes. Default behavior made explicit.

```daml
postconsuming choice Complete : ()
  controller owner
  do
    -- contract still active here during body
    return ()
    -- archived after body completes
```

## Controllers

Controllers are the parties authorized to exercise the choice.

```daml
-- Single controller
choice Accept : ()
  controller receiver
  do ...

-- Multiple controllers (ALL must authorize)
choice JointApproval : ()
  controller [partyA, partyB]
  do ...

-- Flexible controller (from choice argument)
choice DelegatedAction : ()
  with
    actor : Party
  controller actor
  do
    assert (actor `elem` delegates)
```

## Choice Arguments

```daml
choice Transfer : ContractId Asset
  with
    newOwner : Party
    reason : Text
  controller owner
  do
    assert (newOwner /= owner)
    create this with owner = newOwner
```

## Exercise Patterns

### Direct Exercise
```daml
-- In DAML Script or choice body
exercise contractId Transfer with newOwner = alice
```

### Exercise by Key
```daml
exerciseByKey @Asset (issuer, "gold") Transfer with newOwner = alice
```

### Fetch and Exercise
```daml
(cid, contract) <- fetchByKey @Asset (issuer, "gold")
exercise cid Transfer with newOwner = alice
```

### Create and Exercise
```daml
result <- createAndExercise
  (Asset with issuer = bank, owner = bank, description = "bond", quantity = 100.0)
  (Transfer with newOwner = alice)
```

## Advanced Patterns

### Propose-Accept Pattern
```daml
template TransferProposal
  with
    asset : Asset
    newOwner : Party
  where
    signatory asset.owner
    observer newOwner

    choice AcceptTransfer : ContractId Asset
      controller newOwner
      do
        create asset with owner = newOwner

    choice RejectTransfer : ContractId Asset
      controller newOwner
      do
        create asset

    choice WithdrawProposal : ContractId Asset
      controller asset.owner
      do
        create asset
```

### Delegation Pattern
```daml
template OperatorRole
  with
    operator : Party
    owner : Party
  where
    signatory owner
    observer operator

    nonconsuming choice OperateOnBehalf : ContractId Asset
      with
        assetCid : ContractId Asset
        newOwner : Party
      controller operator
      do
        exercise assetCid Transfer with newOwner
```

### Choice Returning Multiple Results
```daml
choice Split : (ContractId Asset, ContractId Asset)
  with
    splitAmount : Decimal
  controller owner
  do
    c1 <- create this with quantity = splitAmount
    c2 <- create this with quantity = quantity - splitAmount
    return (c1, c2)
```

## Authorization Rules

1. **Controller can always exercise** — They have the right by definition
2. **Signatories of resulting contracts must authorize** — Either directly or through the choice chain
3. **Authority flows through choices** — Exercising a choice carries the authority of the controller AND all signatories of the contract

```daml
-- issuer's authority flows through Transfer choice
-- because issuer is signatory of Asset
choice Transfer : ContractId Asset
  with newOwner : Party
  controller owner
  do
    -- Can create new Asset with issuer as signatory
    -- because issuer authorized the original contract
    create this with owner = newOwner
```

## Best Practices

1. **Name choices as actions** — `Transfer`, `Accept`, `Archive`, not `TransferChoice`
2. **Consuming by default** — Only use `nonconsuming` when you explicitly need the contract to survive
3. **Validate in choice body** — Use `assert` or `abort` for runtime checks
4. **Return created contract IDs** — Callers need to track new contracts
5. **Keep choice bodies small** — Extract complex logic into helper functions
6. **Use propose-accept for multi-party** — Never require absent parties to sign

## Common Pitfalls

- **Controller not an observer/signatory** — Controller must be able to see the contract to exercise
- **Missing authorization** — Creating a contract requires all signatory authority
- **Consuming when you shouldn't** — Accidentally archiving contracts that should persist
- **Forgetting return type** — Every choice must declare its return type
