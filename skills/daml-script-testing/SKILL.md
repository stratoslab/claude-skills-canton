---
name: daml-script-testing
description: DAML Script for testing contracts — party allocation, contract creation, exercise, assertions, and test scenarios.
---

# DAML Script Testing

DAML Script is the primary tool for testing DAML contracts. It runs against the ledger (sandbox or Canton) and provides a rich API for simulating multi-party workflows.

## When to Activate

- Writing tests for DAML templates
- Simulating multi-party workflows
- Validating authorization rules
- Setting up test data on sandbox
- Debugging contract behavior

## Basic Test Structure

```daml
module Test where

import Daml.Script
import Main (Asset(..), Transfer(..))

testAssetCreation : Script ()
testAssetCreation = script do
  -- Allocate parties
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"

  -- Create a contract
  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank
      owner = alice
      description = "Gold"
      quantity = 100.0

  -- Exercise a choice
  newAssetCid <- submit alice do
    exerciseCmd assetCid Transfer with
      newOwner = bob

  -- Query and assert
  Some asset <- queryContractId bob newAssetCid
  assert (asset.owner == bob)
  assert (asset.quantity == 100.0)

  return ()
```

## Party Allocation

```daml
-- Simple allocation
alice <- allocateParty "Alice"

-- With party ID hint
bob <- allocatePartyWithHint "Bob" (PartyIdHint "bob-1")

-- Allocate on specific participant
charlie <- allocatePartyOn "Charlie" (ParticipantName "participant1")
```

## Command Submission

### submit — Must succeed
```daml
cid <- submit alice do
  createCmd Asset with ..
```

### submitMustFail — Must fail
```daml
submitMustFail bob do
  createCmd Asset with
    issuer = bob
    owner = alice
    quantity = -1.0  -- violates ensure clause
```

### submitMulti — Multiple acting/reading parties
```daml
submitMulti [alice, bob] [bank] do
  createCmd JointAccount with
    holders = [alice, bob]
    bank = bank
```

### submitTree — Get full transaction tree
```daml
tree <- submitTree alice do
  exerciseCmd proposalCid Accept
-- Inspect tree for created/archived contracts
```

## Query Functions

```daml
-- Query all contracts of a type visible to a party
assets <- query @Asset alice

-- Query by contract ID
Some asset <- queryContractId alice assetCid

-- Query by contract key
Some (cid, asset) <- queryContractKey @Asset alice (bank, "Gold")

-- Query interface
iAssets <- queryInterface @IAsset alice
```

## Assertions and Debugging

```daml
-- Basic assertion
assert (quantity > 0.0)

-- Assert with message
assertMsg "Quantity must be positive" (quantity > 0.0)

-- Debug output
debug ("Asset created: " <> show assetCid)

-- Trace (returns the value, prints as side effect)
let result = trace "Computing..." (1 + 2)
```

## Time Management

```daml
-- Get current ledger time
now <- getTime

-- Set time (only in static time mode)
setTime (time (date 2024 Jan 15) (hours 10))

-- Advance time
passTime (days 30)
```

## Testing Patterns

### Test Authorization Failure
```daml
testUnauthorizedTransfer : Script ()
testUnauthorizedTransfer = script do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank; owner = alice
      description = "Gold"; quantity = 100.0

  -- Bob cannot transfer Alice's asset
  submitMustFail bob do
    exerciseCmd assetCid Transfer with newOwner = bob
```

### Test Propose-Accept Workflow
```daml
testProposeAccept : Script ()
testProposeAccept = script do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  proposalCid <- submit alice do
    createCmd Proposal with
      from = alice; to = bob; terms = "..."

  -- Bob accepts
  agreementCid <- submit bob do
    exerciseCmd proposalCid Accept

  Some agreement <- queryContractId bob agreementCid
  assert (agreement.from == alice)
```

### Test Contract Key Uniqueness
```daml
testKeyUniqueness : Script ()
testKeyUniqueness = script do
  bank <- allocateParty "Bank"

  submit bank do
    createCmd Asset with
      issuer = bank; owner = bank
      description = "Gold"; quantity = 100.0

  -- Same key should fail
  submitMustFail bank do
    createCmd Asset with
      issuer = bank; owner = bank
      description = "Gold"; quantity = 200.0
```

## Running Tests

```bash
# Run all DAML Script tests
daml test

# Run specific test
daml test --files daml/Test.daml

# Run with verbose output
daml test --show-trace

# Run against sandbox
daml test --ledger-host localhost --ledger-port 6865
```

## Best Practices

1. **One assertion per test** when practical — makes failures clear
2. **Name tests descriptively** — `testUnauthorizedTransferFails`, not `test1`
3. **Test both happy and unhappy paths** — Use `submitMustFail`
4. **Allocate fresh parties per test** — Avoid state leakage
5. **Use `debug` for troubleshooting** — Remove before committing
6. **Test key operations** — Create, lookup, exercise-by-key, uniqueness
7. **Test time-dependent logic** — Use `setTime` / `passTime`

## Common Pitfalls

- **Using `submit` when it should fail** — Use `submitMustFail`
- **Forgetting party visibility** — Parties can only see contracts they're signatory/observer on
- **Static vs wall-clock time** — `setTime` only works in static time mode
- **Not checking return values** — Always verify the state after exercising choices
