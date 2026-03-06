---
name: security-review
description: DAML and Canton security review — authorization vulnerabilities, privacy leaks, contention attacks, and secure contract design patterns.
---

# Security Review for DAML & Canton

Security audit patterns for DAML smart contracts and Canton deployments.

## When to Activate

- Reviewing DAML contracts for security issues
- Auditing Canton deployment configurations
- Assessing authorization models
- Checking for privacy violations

## DAML Contract Security Checklist

### 1. Authorization
- [ ] Every choice has the correct controller
- [ ] Signatories are minimal (only parties who must authorize)
- [ ] No choice allows unauthorized parties to act
- [ ] Propose-accept pattern used for multi-signatory contracts
- [ ] No authority escalation paths exist

### 2. Privacy
- [ ] Observer sets are minimal
- [ ] Sensitive data not exposed to unnecessary parties
- [ ] No unintended divulgence paths
- [ ] Separate contracts for different visibility requirements

### 3. Data Validation
- [ ] `ensure` clauses validate all invariants
- [ ] Choice bodies validate arguments with `assert`
- [ ] Numeric values checked for range (no negative quantities)
- [ ] Text fields validated (no empty strings where inappropriate)

### 4. Contract Keys
- [ ] Keys use appropriate business identifiers
- [ ] Maintainers are correct signatories
- [ ] Key contention considered and mitigated

### 5. State Machine
- [ ] Valid state transitions enforced
- [ ] Terminal states cannot be advanced
- [ ] No orphaned contracts possible

## Common Vulnerabilities

### V1: Missing Authorization Check

```daml
-- VULNERABLE: Anyone can call this
choice WithdrawAll : ()
  with actor : Party
  controller actor    -- No check that actor is authorized!
  do ...

-- FIXED: Verify actor is authorized
choice WithdrawAll : ()
  with actor : Party
  controller actor
  do
    assert (actor == owner)  -- Explicit check
```

### V2: Over-Broad Observer Set

```daml
-- VULNERABLE: Competitor sees pricing
template Quote with
    seller : Party
    buyer : Party
    competitor : Party  -- Why is competitor here?
    price : Decimal
  where
    signatory seller
    observer buyer, competitor  -- Privacy leak!

-- FIXED: Separate contracts
template Quote with
    seller : Party
    buyer : Party
    price : Decimal
  where
    signatory seller
    observer buyer  -- Only buyer sees price
```

### V3: Missing Ensure Clause

```daml
-- VULNERABLE: Can create zero-quantity assets
template Asset with
    quantity : Decimal
  where
    signatory issuer

-- FIXED
template Asset with
    quantity : Decimal
  where
    signatory issuer
    ensure quantity > 0.0
```

### V4: Unintended Authority Delegation

```daml
-- VULNERABLE: Operator can do anything with bank's authority
template OperatorRole with
    bank : Party
    operator : Party
  where
    signatory bank
    observer operator

    nonconsuming choice DoAnything : ()
      with action : Update ()
      controller operator
      do action  -- Operator runs arbitrary code with bank's authority!

-- FIXED: Enumerate allowed actions
    nonconsuming choice TransferAsset : ContractId Asset
      with assetCid : ContractId Asset; newOwner : Party
      controller operator
      do exercise assetCid Transfer with newOwner
```

### V5: Time-of-Check-Time-of-Use (TOCTOU)

```daml
-- VULNERABLE in multi-step flows (rare in DAML due to atomicity)
-- But can happen across separate transactions:
-- 1. Check balance in TX1
-- 2. Debit in TX2 (balance may have changed!)

-- FIXED: Check and act in same transaction
choice SafeTransfer : ContractId Asset
  with newOwner : Party; minBalance : Decimal
  controller owner
  do
    assert (quantity >= minBalance)  -- Check
    create this with owner = newOwner  -- Act (same TX)
```

## Canton Deployment Security

### Infrastructure
- [ ] TLS enabled on all APIs
- [ ] Admin API not publicly accessible
- [ ] JWT authentication on Ledger API
- [ ] Database credentials in environment variables
- [ ] Network segmentation between components

### Authentication
- [ ] JWT tokens have appropriate expiry
- [ ] Token scopes are minimal (actAs/readAs)
- [ ] Admin tokens separated from user tokens
- [ ] JWKS endpoint uses HTTPS

### Monitoring
- [ ] Failed authentication attempts logged
- [ ] Command rejections monitored
- [ ] ACS size growth tracked
- [ ] Unusual party allocation detected

## Security Testing

```daml
-- Test that unauthorized exercise fails
testSecurityUnauthorizedExercise : Script ()
testSecurityUnauthorizedExercise = script do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with issuer = bank; owner = alice; quantity = 100.0

  -- Bob should NOT be able to transfer Alice's asset
  submitMustFail bob do
    exerciseCmd assetCid Transfer with newOwner = bob

  -- Alice should NOT be able to create assets pretending to be bank
  submitMustFail alice do
    createCmd Asset with issuer = bank; owner = alice; quantity = 999.0
```

## Best Practices

1. **Principle of least authority** — Minimal signatories and controllers
2. **Principle of least visibility** — Minimal observers
3. **Validate all inputs** — ensure clauses + assert in choices
4. **Enumerate, don't generalize** — Specific choices, not generic action runners
5. **Test all authorization paths** — Verify every choice rejects unauthorized parties
6. **Separate concerns** — Different contracts for different sensitivity levels
