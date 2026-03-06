---
name: canton-privacy
description: Canton privacy model — sub-transaction privacy, visibility rules, divulgence, projection, and privacy-aware contract design.
---

# Canton Privacy Model

Canton provides sub-transaction privacy — each party sees only the parts of a transaction relevant to them. This is fundamentally different from most blockchains where all data is visible to all participants.

## When to Activate

- Designing multi-party workflows with confidentiality requirements
- Understanding who can see what on the ledger
- Debugging visibility issues
- Building privacy-preserving applications
- Compliance with data privacy regulations (GDPR)

## Core Principle: Need-to-Know

A party only sees parts of a transaction if they are a **stakeholder** of an affected contract.

## Visibility Rules

### Who Sees What

| Action | Visible To |
|--------|-----------|
| Create | Signatories + Observers of created contract |
| Consuming Exercise | Signatories + Observers of target contract + Controller + Choice Observers |
| Non-consuming Exercise | Controller + Choice Observers |
| Fetch | Signatories + Observers of fetched contract + Acting parties |

### Stakeholder Roles

```daml
template ConfidentialDeal
  with
    bank : Party       -- signatory: SEES everything about this contract
    client : Party     -- signatory: SEES everything about this contract
    auditor : Party    -- observer: SEES the contract, but didn't authorize creation
    amount : Decimal
  where
    signatory bank, client
    observer auditor
```

## Sub-Transaction Privacy

In a single transaction, different parties see different **views**.

```
Transaction:
  ├── Exercise Transfer on Asset (Alice -> Bob)     ← Alice, Bob, Bank see this
  │   ├── Archive old Asset                          ← Alice, Bank see this
  │   ├── Create new Asset (owner=Bob)               ← Bob, Bank see this
  │   └── Create AuditEntry                          ← Bank, Auditor see this
  └── Exercise UpdateBalance on Account              ← Only Bank sees this
```

- Alice sees: The exercise and the archive of her old contract
- Bob sees: The exercise and creation of his new contract
- Bank sees: Everything (signatory on all contracts)
- Auditor sees: Only the AuditEntry creation

## Projection

A party's **projection** is the subset of the transaction tree visible to them.

```daml
-- Alice's projection:
  Exercise Transfer
    Archive Asset(owner=Alice)

-- Bob's projection:
  Exercise Transfer
    Create Asset(owner=Bob)

-- Bank's projection (full tree):
  Exercise Transfer
    Archive Asset(owner=Alice)
    Create Asset(owner=Bob)
    Create AuditEntry
  Exercise UpdateBalance
```

## Divulgence

When a contract is **fetched** inside an exercise that a party can see, that party learns about the contract even though they're not a stakeholder.

```daml
-- Alice can see the Fetch of Bob's contract if it happens
-- inside an exercise visible to Alice
choice ProcessBatch : ()
  controller admin
  do
    -- If Alice is an observer of this exercise,
    -- she learns about Bob's contract via divulgence
    bobContract <- fetch bobContractId
    ...
```

**Important**: Divulgence is being phased out in favor of **explicit disclosure** in newer DAML versions. Design new contracts to avoid reliance on divulgence.

## Privacy-Aware Design Patterns

### Pattern 1: Intermediary for Privacy

```daml
-- Buyer and Seller don't see each other
template BrokeredTrade
  with
    broker : Party     -- sees both sides
    buyerCid : ContractId BuyOrder    -- buyer details private
    sellerCid : ContractId SellOrder  -- seller details private
  where
    signatory broker
    -- Buyer only sees their BuyOrder
    -- Seller only sees their SellOrder
    -- Broker sees both + the BrokeredTrade
```

### Pattern 2: Minimal Observer Sets

```daml
-- BAD: Everyone sees everything
template BadDesign with
    issuer : Party
    owner : Party
    regulator : Party
    auditor : Party
    amount : Decimal
  where
    signatory issuer
    observer owner, regulator, auditor  -- All see the amount!

-- GOOD: Separate contracts for different visibility
template Asset with
    issuer : Party
    owner : Party
    amount : Decimal
  where
    signatory issuer
    observer owner

template AuditRecord with
    issuer : Party
    auditor : Party
    assetRef : Text  -- Reference, not the full data
  where
    signatory issuer
    observer auditor
```

### Pattern 3: Choice Observers for Selective Visibility

```daml
template Auction
  with
    auctioneer : Party
    item : Text
  where
    signatory auctioneer

    nonconsuming choice PlaceBid : ContractId Bid
      with
        bidder : Party
        amount : Decimal
      observer []  -- No additional observers; only auctioneer and bidder see this
      controller bidder
      do
        create Bid with ..
```

### Pattern 4: Privacy-Preserving Aggregation

```daml
-- Instead of one contract with all data:
-- Use separate contracts per party, aggregate only via a trusted party

template PortfolioSummary
  with
    custodian : Party
    owner : Party
    totalValue : Decimal  -- Owner sees their total
  where
    signatory custodian
    observer owner

-- Individual holdings are separate contracts
-- Other owners cannot see each other's holdings
```

## GDPR Compliance

Canton supports the **right to be forgotten** through pruning:

```scala
// Remove historical data up to an offset
participant1.pruning.prune(offset)
```

- Pruning removes transaction history but preserves active contracts
- After pruning, archived contracts and old transactions are deleted
- ACS (active contracts) is not affected by pruning

## Debugging Visibility

### Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| Party can't see contract | Not a signatory or observer |
| Party sees too much | Added as observer unnecessarily |
| Choice exercise invisible | Party not a stakeholder of target contract |
| Unexpected contract knowledge | Divulgence from a fetch in a visible exercise |

### Investigation Steps

1. Check who are signatories and observers of the contract
2. Check the transaction tree — which exercises are parents?
3. Check if the contract was fetched inside a visible exercise (divulgence)
4. Use transaction tree inspection to see each party's projection

## Best Practices

1. **Minimize observer sets** — Only add parties who truly need visibility
2. **Use separate contracts for different audiences** — Don't mix confidential and public data
3. **Avoid divulgence** — Use explicit disclosure patterns instead
4. **Design for projection** — Think about what each party's view looks like
5. **Test visibility** — Write DAML Script tests that verify parties can/cannot see contracts
6. **Plan for pruning** — Ensure your application works after historical data is pruned
