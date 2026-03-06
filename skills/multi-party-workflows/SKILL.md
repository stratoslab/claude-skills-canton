---
name: multi-party-workflows
description: Multi-party authorization workflows — propose-accept, multi-signatory patterns, authority accumulation, and cross-participant coordination.
---

# Multi-Party Workflows

DAML's authorization model requires explicit consent from all signatories. Multi-party workflows coordinate this consent across organizational boundaries.

## When to Activate

- Designing workflows requiring multiple parties' consent
- Implementing trade, settlement, or approval flows
- Building cross-organization processes
- Solving authorization challenges in multi-party contracts

## Authorization Model Recap

- **Create** requires authorization from **all signatories**
- **Exercise** requires authorization from **all controllers**
- Authority **flows through exercises** — nested creates inherit parent authority
- A single command submission can only provide one party's authorization

## Pattern: Two-Party Propose-Accept

The simplest multi-party pattern.

```daml
template ServiceAgreement
  with
    provider : Party
    client : Party
    terms : Text
    fee : Decimal
  where
    signatory provider, client  -- Both must agree

-- Since both must sign, use proposal workflow:
template ServiceProposal
  with
    provider : Party
    client : Party
    terms : Text
    fee : Decimal
  where
    signatory provider          -- Only provider signs the proposal
    observer client             -- Client can see and act on it

    choice AcceptService : ContractId ServiceAgreement
      controller client         -- Client exercises = adds their authority
      do
        create ServiceAgreement with ..  -- Now both authorities present

    choice RejectService : ()
      controller client
      do return ()

    choice WithdrawProposal : ()
      controller provider
      do return ()
```

## Pattern: N-Party Sequential Approval

```daml
template MultiApproval
  with
    initiator : Party
    pendingApprovers : [Party]
    completedApprovers : [Party]
    payload : Text
  where
    signatory initiator :: completedApprovers
    observer pendingApprovers

    choice Approve : Either (ContractId MultiApproval) (ContractId Approved)
      with approver : Party
      controller approver
      do
        assert (approver `elem` pendingApprovers)
        let remaining = filter (/= approver) pendingApprovers
            approved = approver :: completedApprovers
        if null remaining
          then Right <$> create Approved with
            signers = initiator :: approved; payload
          else Left <$> create this with
            pendingApprovers = remaining
            completedApprovers = approved

template Approved
  with
    signers : [Party]
    payload : Text
  where
    signatory signers
```

## Pattern: Delivery-vs-Payment (DvP)

Atomic swap of two assets between two parties.

```daml
template DvPProposal
  with
    buyer : Party
    seller : Party
    paymentAmount : Decimal
    assetDescription : Text
  where
    signatory buyer
    observer seller

    choice SettleDvP : (ContractId Asset, ContractId Payment)
      with
        paymentCid : ContractId Payment
        assetCid : ContractId Asset
      controller seller
      do
        -- Transfer payment from buyer to seller
        newPaymentCid <- exercise paymentCid Transfer with newOwner = seller
        -- Transfer asset from seller to buyer
        newAssetCid <- exercise assetCid Transfer with newOwner = buyer
        return (newAssetCid, newPaymentCid)
```

## Pattern: Role-Based Delegation

```daml
template CustodianRole
  with
    custodian : Party
    regulator : Party
  where
    signatory regulator
    observer custodian

    nonconsuming choice OpenAccount : ContractId Account
      with
        owner : Party
      controller custodian
      do
        -- Custodian acts with regulator's authority for account creation
        create Account with
          custodian
          owner
          regulator
          balance = 0.0

template Account
  with
    custodian : Party
    owner : Party
    regulator : Party
    balance : Decimal
  where
    signatory custodian, regulator  -- Both signed via role delegation
    observer owner
```

## Cross-Participant Coordination

When parties are on different Canton participants:

```
Participant 1 (Alice)              Participant 2 (Bob)
    │                                    │
    │  1. Alice creates Proposal         │
    │     (visible to Bob via observer)  │
    │                                    │
    │  2. Bob sees Proposal              │
    │     (delivered via synchronizer)   │
    │                                    │
    │  3. Bob exercises Accept           │
    │     (authorized by Bob)            │
    │                                    │
    │  4. Both see resulting Agreement   │
    │     (atomic, via Canton protocol)  │
```

Canton's synchronization protocol ensures atomicity even when parties are on different participants.

## Authority Accumulation in Nested Exercises

```daml
-- Each exercise in the chain adds its controller's authority
template Escrow
  with
    escrowAgent : Party
    buyer : Party
    seller : Party
    amount : Decimal
  where
    signatory escrowAgent
    observer buyer, seller

    choice Release : ContractId Payment
      controller buyer          -- buyer's authority added
      do
        -- Inside here: escrowAgent + buyer authority available
        exercise sellerPaymentRight ConfirmRelease
        -- Now: escrowAgent + buyer + seller authority available
        create Payment with payer = escrowAgent; payee = seller; amount
```

## Testing Multi-Party Workflows

```daml
testProposalAccept : Script ()
testProposalAccept = script do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  -- Alice creates proposal
  proposalCid <- submit alice do
    createCmd ServiceProposal with
      provider = alice; client = bob
      terms = "Consulting"; fee = 1000.0

  -- Bob accepts
  agreementCid <- submit bob do
    exerciseCmd proposalCid AcceptService

  -- Both can see the agreement
  Some agreement <- queryContractId alice agreementCid
  assert (agreement.provider == alice)
  assert (agreement.client == bob)

  -- Verify multi-signatory (neither can unilaterally archive)
  submitMustFail alice do
    exerciseCmd agreementCid Archive
```

## Best Practices

1. **Use propose-accept for multi-signatory** — The only safe way to gather authorization
2. **Keep proposal contracts lightweight** — They're temporary
3. **Always include Withdraw/Reject choices** — Let parties back out
4. **Test cross-party visibility** — Verify each party sees what they should
5. **Design for failure** — Include timeout/expiry mechanisms for proposals
6. **Avoid deep nesting** — Hard to reason about authority; keep chains shallow
