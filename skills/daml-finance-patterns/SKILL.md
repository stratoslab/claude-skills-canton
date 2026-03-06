---
name: daml-finance-patterns
description: DAML Finance library patterns — holdings, accounts, instruments, settlement, lifecycle events, and tokenization patterns.
---

# DAML Finance Patterns

The DAML Finance library provides purpose-built abstractions for financial asset tokenization, custody, and settlement on Canton.

## When to Activate

- Building financial applications on Canton
- Tokenizing assets (securities, bonds, funds)
- Implementing custody and account models
- Building settlement workflows
- Modeling financial instrument lifecycles

## Core Concepts

### Holdings
A **Holding** represents ownership of a quantity of an instrument in an account.

```
Holding = {
  instrument: InstrumentKey,   -- What is held (bond, token, etc.)
  account: AccountKey,          -- Where it is held (custodian + owner)
  amount: Decimal               -- How much
}
```

### Accounts
An **Account** represents a custody relationship between a custodian and an owner.

```
Account = {
  custodian: Party,    -- Who holds the assets
  owner: Party,        -- Who owns the assets
  id: Id               -- Account identifier
}
```

### Instruments
An **Instrument** describes a financial product (token, bond, equity, etc.).

```
Instrument = {
  issuer: Party,
  depository: Party,
  id: Id,
  version: Text
}
```

## Key Types

```daml
-- From Daml.Finance.Interface.Types
data AccountKey = AccountKey
  with
    custodian : Party
    owner : Party
    id : Id
  deriving (Eq, Show, Ord)

data InstrumentKey = InstrumentKey
  with
    issuer : Party
    depository : Party
    id : Id
    version : Text
  deriving (Eq, Show, Ord)

data Id = Id with unpack : Text
  deriving (Eq, Show, Ord)
```

## Interface Hierarchy

```
Disclosure.I          -- Privacy control (who can see)
  └── Lockable.I      -- Lock/unlock for settlement
       └── Holding.I   -- Asset ownership
            └── Fungible.I    -- Split/merge operations
            └── Transferable.I -- Transfer between accounts
```

## Holdings: Create and Transfer

### Creating a Holding (via Account Credit)

```daml
-- Exercise the Credit choice on the account
holdingCid <- exercise accountCid Account.Credit with
  quantity = Instrument.Quantity with
    unit = instrumentKey
    amount = 1000.0
```

### Transferring a Holding

```daml
-- Transfer from Alice's account to Bob's account
newHoldingCid <- exercise holdingCid Transferable.Transfer with
  actors = Set.singleton alice
  newOwnerAccount = bobAccountKey
```

### Splitting and Merging (Fungible)

```daml
-- Split a holding
[part1, part2] <- exercise holdingCid Fungible.Split with
  amounts = [300.0, 700.0]

-- Merge holdings
merged <- exercise holding1Cid Fungible.Merge with
  fungibleCids = [holding2Cid, holding3Cid]
```

## Accounts: Setup

```daml
-- Create an account factory
factoryCid <- submit custodian do
  createCmd Account.Factory with
    provider = custodian
    observers = Map.empty

-- Open an account
accountCid <- exercise factoryCid Account.Factory.Create with
  account = AccountKey with
    custodian
    owner = alice
    id = Id "alice-main"
  holdingFactory = holdingFactoryCid
  controllers = Account.Controllers with
    outgoing = Set.singleton alice
    incoming = Set.singleton alice
  observers = Map.empty
```

## Settlement: Delivery vs Payment

```daml
-- Settlement involves:
-- 1. Create settlement instructions
-- 2. Allocate holdings to instructions
-- 3. Execute settlement atomically

-- Step 1: Create settlement batch
batchCid <- exercise settlementFactoryCid Settlement.Instruct with
  instructor = exchange
  settlers = Set.fromList [alice, bob]
  id = Id "dvp-001"
  description = "Bond purchase"
  contextId = None
  routedSteps = [
    RoutedStep with
      sender = alice
      receiver = bob
      quantity = Instrument.Quantity with
        unit = cashInstrument
        amount = 10000.0,
    RoutedStep with
      sender = bob
      receiver = alice
      quantity = Instrument.Quantity with
        unit = bondInstrument
        amount = 100.0
  ]

-- Step 2: Allocate holdings
exercise instructionCid Instruction.Allocate with
  actors = Set.singleton alice
  allocation = Instruction.Pledge cashHoldingCid

-- Step 3: Execute when all allocated
exercise batchCid Batch.Settle with
  actors = Set.fromList [alice, bob]
```

## Lifecycle Events

```daml
-- Create a lifecycle rule
ruleCid <- submit issuer do
  createCmd Distribution.Rule with
    providers = Set.singleton issuer
    observers = Map.empty

-- Create a distribution event (e.g., dividend)
eventCid <- submit issuer do
  createCmd Distribution.Event with
    providers = Set.singleton issuer
    id = Id "div-2024-q1"
    description = "Q1 Dividend"
    effectiveTime = time (date 2024 Mar 15) (hours 0)
    instrument = bondInstrumentKey
    newInstrument = bondInstrumentKeyV2
    perUnitDistribution = [
      Instrument.Quantity with
        unit = cashInstrument
        amount = 0.05  -- $0.05 per unit
    ]

-- Apply lifecycle event to holdings
exercise ruleCid Distribution.Evolve with
  eventCid
  instrument = bondInstrumentKey
  holdingCids = [holdingCid1, holdingCid2]
```

## Style Conventions (from DAML Finance)

```daml
-- Type aliases for interfaces
type I = Holding        -- Interface
type V = View           -- View type

-- Qualified imports
import Daml.Finance.Interface.Holding.V4 qualified as Holding

-- Use `pure` instead of `return`
pure $ view this

-- Short-hand notation for constructors
Foo with a; b; c

-- Flexible controllers pattern
choice Transfer : ContractId Holding.I
  with
    actors : Set Party    -- Callers specify who is acting
  controller actors
  do ...
```

## Best Practices

1. **Use DAML Finance interfaces** — Don't reinvent holding/account/instrument
2. **Follow the factory pattern** — Use factories to create holdings and accounts
3. **Use settlement for atomic swaps** — Never transfer assets without settlement
4. **Version instruments** — Lifecycle events create new instrument versions
5. **Use the disclosure interface** — Control visibility explicitly
6. **Lock holdings during settlement** — Prevent double-allocation
