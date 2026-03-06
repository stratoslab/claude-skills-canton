---
name: contract-keys
description: Contract key design — defining keys, maintainers, uniqueness constraints, key operations, and patterns for stable contract lookups.
---

# Contract Keys

Contract keys provide stable, human-readable lookups for contracts. Unlike contract IDs (which are opaque and change on recreate), keys use business identifiers.

## When to Activate

- Designing contract lookup patterns
- Implementing idempotent operations
- Building key-based workflows
- Troubleshooting key uniqueness violations

## Defining Keys

```daml
template Account
  with
    bank : Party
    owner : Party
    accountNumber : Text
    balance : Decimal
  where
    signatory bank
    observer owner

    key (bank, accountNumber) : (Party, Text)
    maintainer key._1  -- bank is the maintainer
```

### Key Constraints

1. **Must include at least one party** — The maintainer
2. **Maintainer must be a signatory** — Only signatories can maintain key uniqueness
3. **At most one active contract per key** — Enforced by the ledger
4. **Key types must be serializable** — No `ContractId` in keys

### Common Key Shapes

```daml
-- Simple party key (singleton per party)
key owner : Party
maintainer key

-- Party + identifier
key (issuer, isin) : (Party, Text)
maintainer key._1

-- Compound key
key (bank, accountType, accountNumber) : (Party, Text, Text)
maintainer key._1

-- Using a custom record
data AccountKey = AccountKey with
    bank : Party
    number : Text
  deriving (Eq, Show, Ord)

key AccountKey with bank; number = accountNumber : AccountKey
maintainer key.bank
```

## Key Operations

### fetchByKey
Fetches the active contract with the given key. Fails if no contract exists.

```daml
(cid, account) <- fetchByKey @Account (bank, "ACC-001")
```

### lookupByKey
Returns `Some contractId` if found, `None` otherwise.

```daml
mbAccount <- lookupByKey @Account (bank, "ACC-001")
case mbAccount of
  Some cid -> exercise cid Deposit with amount = 100.0
  None -> abort "Account not found"
```

### exerciseByKey
Exercise a choice directly on the contract identified by key.

```daml
exerciseByKey @Account (bank, "ACC-001") Deposit with amount = 100.0
```

### Visibility Rules for Key Operations

| Operation | Requirement |
|-----------|-------------|
| `fetchByKey` | Submitter must be a **stakeholder** |
| `lookupByKey` | Submitter must be a **maintainer** |
| `exerciseByKey` | Submitter must be a **stakeholder** + **controller** |

## Design Patterns

### Idempotent Creation

```daml
ensureAccount : Party -> Text -> Script (ContractId Account)
ensureAccount bank accNum = do
  existing <- queryContractKey @Account (bank, accNum)
  case existing of
    Some (cid, _) -> pure cid
    None -> submit bank do
      createCmd Account with
        bank; owner = bank; accountNumber = accNum; balance = 0.0
```

### Singleton Pattern (One per Party)

```daml
template UserProfile
  with
    user : Party
    name : Text
    email : Text
  where
    signatory user
    key user : Party
    maintainer key

    choice UpdateProfile : ContractId UserProfile
      with newName : Text; newEmail : Text
      controller user
      do create this with name = newName; email = newEmail
```

### Key Migration (Changing Key Values)

Since keys are tied to contract data, changing a key value requires archive + create:

```daml
choice Rename : ContractId Account
  with
    newNumber : Text
  controller bank
  do
    -- Archives this contract (old key) and creates new one (new key)
    create this with accountNumber = newNumber
```

## Contention and Keys

Keys create **contention points**. Two concurrent transactions trying to:
- Create contracts with the same key → one will be rejected
- Exercise the same key → one will be rejected (if consuming)

### Reducing Key Contention

```daml
-- BAD: Hot key — all transactions touch the same contract
template GlobalCounter with
    admin : Party
    count : Int
  where
    key admin : Party
    maintainer key

-- BETTER: Partition by some dimension
template PartitionedCounter with
    admin : Party
    partition : Text
    count : Int
  where
    key (admin, partition) : (Party, Text)
    maintainer key._1
```

## Cross-Domain Key Considerations

In Canton with multiple synchronizers, key uniqueness is enforced **per synchronizer**. A contract with key `(Bank, "ACC-001")` on Domain A does not prevent the same key on Domain B.

Design accordingly if your application spans multiple domains.

## Best Practices

1. **Use business identifiers as keys** — Account numbers, ISINs, not synthetic IDs
2. **Keep keys stable** — Avoid keys that change frequently
3. **Minimize key contention** — Partition hot keys
4. **Use lookupByKey for conditional logic** — Check existence before creating
5. **Document key semantics** — What does the key represent in business terms?
6. **Test key uniqueness** — Write tests that verify duplicate creation fails

## Common Pitfalls

- **Maintainer not a signatory** — Compiler error
- **No party in key** — Keys must include at least one party
- **ContractId in key** — Not allowed (not serializable as key)
- **lookupByKey without maintainer authority** — Submitter must be a maintainer
- **Assuming cross-domain uniqueness** — Keys are per-synchronizer in Canton
