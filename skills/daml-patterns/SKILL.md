---
name: daml-patterns
description: Advanced DAML design patterns — propose-accept, delegation, role contracts, state machines, the initiate-accept pattern, and authorization flow design.
---

# Advanced DAML Design Patterns

Proven patterns for modeling multi-party workflows, authorization flows, and complex business logic in DAML.

## When to Activate

- Designing multi-party authorization workflows
- Modeling real-world business processes
- Building role-based access control
- Implementing state machines on the ledger
- Solving authorization challenges

## Pattern 1: Propose-Accept

The most fundamental multi-party pattern. Used whenever a contract requires multiple signatories who cannot simultaneously submit.

```daml
template TradeProposal
  with
    buyer : Party
    seller : Party
    asset : Text
    price : Decimal
  where
    signatory buyer
    observer seller

    choice Accept : ContractId Trade
      controller seller
      do
        create Trade with ..

    choice Reject : ()
      controller seller
      do
        return ()

    choice Withdraw : ()
      controller buyer
      do
        return ()

template Trade
  with
    buyer : Party
    seller : Party
    asset : Text
    price : Decimal
  where
    signatory buyer, seller  -- Both authorized via propose-accept flow
```

## Pattern 2: Role Contracts

Long-lived contracts that grant a party operational capabilities. The role contract acts as a capability token.

```daml
template OperatorRole
  with
    operator : Party
    admin : Party
  where
    signatory admin
    observer operator

    nonconsuming choice CreateAsset : ContractId Asset
      with
        owner : Party
        description : Text
        quantity : Decimal
      controller operator
      do
        create Asset with
          issuer = admin
          owner
          description
          quantity

    choice RevokeRole : ()
      controller admin
      do return ()
```

## Pattern 3: Delegation

Grant another party the right to act on your behalf through a delegation contract.

```daml
template VotingProxy
  with
    voter : Party
    delegate : Party
    election : Text
  where
    signatory voter
    observer delegate

    choice CastVote : ContractId Vote
      with
        candidate : Text
      controller delegate
      do
        create Vote with
          voter
          candidate
          election
```

## Pattern 4: State Machine

Model sequential states by archiving the current state contract and creating the next one.

```daml
data OrderState = Pending | Approved | Shipped | Delivered
  deriving (Eq, Show, Enum)

template Order
  with
    buyer : Party
    seller : Party
    item : Text
    state : OrderState
  where
    signatory buyer, seller

    choice Advance : ContractId Order
      controller (case state of
        Pending -> seller
        Approved -> seller
        Shipped -> buyer
        Delivered -> error "Terminal state")
      do
        let nextState = case state of
              Pending -> Approved
              Approved -> Shipped
              Shipped -> Delivered
              Delivered -> error "Cannot advance from Delivered"
        create this with state = nextState
```

## Pattern 5: Controller Delegation via Flexible Controllers

```daml
template SharedAccount
  with
    bank : Party
    holders : [Party]
  where
    signatory bank
    observer holders

    nonconsuming choice Withdraw : ContractId Receipt
      with
        actor : Party
        amount : Decimal
      controller actor
      do
        assert (actor `elem` holders)
        create Receipt with
          account = self
          party = actor
          amount
```

## Pattern 6: Master-Subcontract (Composition)

Break complex agreements into a master contract with linked sub-contracts.

```daml
template LoanAgreement
  with
    lender : Party
    borrower : Party
    principal : Decimal
    rate : Decimal
  where
    signatory lender, borrower

    nonconsuming choice CreateRepaymentSchedule : [ContractId RepaymentInstallment]
      controller lender
      do
        let installments = calculateSchedule principal rate
        mapA (\(date, amount) -> create RepaymentInstallment with
          lender; borrower; dueDate = date; amount) installments
```

## Pattern 7: Privacy via Intermediary

When two parties should not know each other, use an intermediary.

```daml
-- Broker sees both sides, buyer and seller don't see each other
template BrokeredTrade
  with
    broker : Party
    buyOrder : ContractId BuyOrder
    sellOrder : ContractId SellOrder
  where
    signatory broker
    -- buyer/seller only see their own sub-transactions
```

## Pattern 8: Idempotent Create with Contract Keys

```daml
template Singleton
  with
    owner : Party
    data : Text
  where
    signatory owner

    key owner : Party
    maintainer key

    choice UpdateData : ContractId Singleton
      with
        newData : Text
      controller owner
      do
        create this with data = newData

-- Usage: lookupByKey first, only create if None
ensureSingleton : Party -> Text -> Script (ContractId Singleton)
ensureSingleton owner data = do
  existing <- queryContractKey @Singleton owner
  case existing of
    Some (cid, _) -> pure cid
    None -> submit owner do createCmd Singleton with owner; data
```

## Pattern 9: Authorization Accumulation

Nested exercises accumulate authorization from all controllers in the call chain.

```daml
template MultiSigRequest
  with
    requester : Party
    approvers : [Party]
    approved : [Party]
    payload : Text
  where
    signatory requester, approved
    observer approvers

    choice Approve : Either (ContractId MultiSigRequest) (ContractId MultiSigAgreement)
      with
        approver : Party
      controller approver
      do
        assert (approver `elem` approvers)
        let newApproved = approver :: approved
        let remaining = filter (/= approver) approvers
        if null remaining
          then do
            cid <- create MultiSigAgreement with signers = requester :: newApproved; payload
            return (Right cid)
          else do
            cid <- create this with approved = newApproved; approvers = remaining
            return (Left cid)
```

## Anti-Patterns to Avoid

1. **God template** — One template with everything; split into focused templates
2. **Missing observers** — Forgetting to add parties who need visibility
3. **Mutable state illusion** — DAML contracts are immutable; use archive-and-recreate
4. **Over-signing** — Making too many parties signatories when observers suffice
5. **Tight coupling** — Templates that directly reference other templates' internals; use interfaces instead

## Decision Guide

| Need | Pattern |
|------|---------|
| Multi-party agreement | Propose-Accept |
| Granting capabilities | Role Contract |
| Acting on behalf of | Delegation |
| Sequential states | State Machine |
| Read-only access | Non-consuming choice |
| Unique lookup | Contract Key + Idempotent Create |
| Privacy separation | Intermediary |
| Extensible contracts | Interfaces |
