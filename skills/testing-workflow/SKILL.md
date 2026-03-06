---
name: testing-workflow
description: TDD workflow for DAML — test-first development, DAML Script patterns, property testing, integration testing, and CI setup.
---

# DAML Testing Workflow

Test-driven development for DAML contracts using DAML Script, with patterns for unit, integration, and property testing.

## When to Activate

- Writing tests for new DAML templates
- Setting up TDD workflows for Canton projects
- Building CI pipelines for DAML code
- Debugging test failures

## TDD Cycle for DAML

```
1. Write a DAML Script test (RED)
2. Write the template/choice to make it pass (GREEN)
3. Refactor while tests pass (REFACTOR)
4. Repeat
```

## Test Structure

```daml
module Test.Asset where

import Daml.Script
import DA.Assert ((===))
import Main (Asset(..), Transfer(..), Split(..))

-- Test naming: test<Feature><Scenario>
testTransferUpdatesOwner : Script ()
testTransferUpdatesOwner = script do
  -- ARRANGE
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank; owner = alice
      description = "Gold"; quantity = 100.0

  -- ACT
  newCid <- submit alice do
    exerciseCmd assetCid Transfer with newOwner = bob

  -- ASSERT
  Some asset <- queryContractId bob newCid
  asset.owner === bob
  asset.quantity === 100.0

testUnauthorizedTransferFails : Script ()
testUnauthorizedTransferFails = script do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank; owner = alice
      description = "Gold"; quantity = 100.0

  submitMustFail bob do
    exerciseCmd assetCid Transfer with newOwner = bob

testSplitPreservesTotal : Script ()
testSplitPreservesTotal = script do
  alice <- allocateParty "Alice"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank; owner = alice
      description = "Gold"; quantity = 100.0

  (cid1, cid2) <- submit alice do
    exerciseCmd assetCid Split with splitQuantity = 30.0

  Some a1 <- queryContractId alice cid1
  Some a2 <- queryContractId alice cid2
  a1.quantity === 30.0
  a2.quantity === 70.0

testInvalidQuantityFails : Script ()
testInvalidQuantityFails = script do
  bank <- allocateParty "Bank"
  submitMustFail bank do
    createCmd Asset with
      issuer = bank; owner = bank
      description = "Gold"; quantity = -1.0  -- Violates ensure
```

## Running Tests

```bash
# Run all tests
daml test

# With trace output
daml test --show-trace

# Specific file
daml test --files test/Test/Asset.daml

# Against running ledger
daml test --ledger-host localhost --ledger-port 6865
```

## Test Patterns

### Setup Helpers

```daml
data TestParties = TestParties with
  alice : Party
  bob : Party
  bank : Party

allocateTestParties : Script TestParties
allocateTestParties = do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"
  pure TestParties with ..

createTestAsset : Party -> Party -> Decimal -> Script (ContractId Asset)
createTestAsset bank owner qty =
  submit bank do
    createCmd Asset with
      issuer = bank; owner; description = "Test"; quantity = qty
```

### Visibility Testing

```daml
testObserverCanSeeContract : Script ()
testObserverCanSeeContract = script do
  TestParties{..} <- allocateTestParties

  assetCid <- submit bank do
    createCmd Asset with
      issuer = bank; owner = alice
      description = "Gold"; quantity = 100.0

  -- Alice (observer) can see it
  Some _ <- queryContractId alice assetCid

  -- Bob (not a stakeholder) cannot see it
  None <- queryContractId bob assetCid

  pure ()
```

### Workflow Testing

```daml
testFullWorkflow : Script ()
testFullWorkflow = script do
  TestParties{..} <- allocateTestParties

  -- 1. Bank issues asset to Alice
  assetCid <- createTestAsset bank alice 100.0

  -- 2. Alice transfers to Bob
  newCid <- submit alice do
    exerciseCmd assetCid Transfer with newOwner = bob

  -- 3. Bob splits
  (part1, part2) <- submit bob do
    exerciseCmd newCid Split with splitQuantity = 40.0

  -- 4. Verify final state
  assets <- query @Asset bob
  length assets === 2
```

### Time-Dependent Testing

```daml
testExpiryAfterDeadline : Script ()
testExpiryAfterDeadline = script do
  alice <- allocateParty "Alice"
  let deadline = time (date 2024 Dec 31) (hours 23)

  offerCid <- submit alice do
    createCmd TimedOffer with
      offerer = alice; deadline; terms = "..."

  -- Before deadline: can still accept
  setTime (time (date 2024 Dec 30) (hours 12))
  -- ... exercise Accept would succeed

  -- After deadline: expired
  setTime (time (date 2025 Jan 1) (hours 0))
  submitMustFail alice do
    exerciseCmd offerCid Accept
```

## Test Organization

```
test/
  Test/
    Asset.daml           -- Tests for Asset template
    Transfer.daml        -- Tests for transfer workflows
    Settlement.daml      -- Tests for settlement
    Authorization.daml   -- Authorization edge cases
    Integration.daml     -- End-to-end scenarios
```

## CI Pipeline

```yaml
# .github/workflows/daml-test.yml
name: DAML Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install DAML SDK
        run: curl -sSL https://get.daml.com/ | sh

      - name: Build
        run: daml build

      - name: Test
        run: daml test --show-trace

      - name: Check formatting
        run: daml damlc lint --diff
```

## Best Practices

1. **One assertion per test** — Clear failure messages
2. **Test both success and failure** — Happy path + `submitMustFail`
3. **Use setup helpers** — Reduce boilerplate, improve readability
4. **Test authorization exhaustively** — Who can and cannot exercise each choice
5. **Test contract keys** — Creation, lookup, uniqueness violations
6. **Test visibility** — Verify who can and cannot see contracts
7. **Name tests descriptively** — `testTransferToSelfFails` not `test3`
8. **Run tests in CI** — Every commit should pass `daml test`
