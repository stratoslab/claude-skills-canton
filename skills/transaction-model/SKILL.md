---
name: transaction-model
description: DAML transaction model — transaction trees, action types, rollback nodes, atomicity, and flat vs tree representations.
---

# Transaction Model

Transactions in DAML are trees of actions. Understanding this structure is essential for debugging, auditing, and building correct applications.

## When to Activate

- Debugging transaction failures
- Understanding transaction causality
- Inspecting transaction trees via Ledger API
- Building audit trails
- Analyzing rollback behavior

## Transaction Structure

A transaction is a **tree of actions**, not a flat list.

```
Transaction
├── Exercise "Transfer" on Asset#1 (consuming)
│   ├── Fetch Asset#1                    (leaf)
│   ├── Archive Asset#1                  (leaf)
│   └── Create Asset#2 (owner=Bob)       (leaf)
└── Create AuditLog#1                    (root-level create)
```

## Action Types

### Create
Produces a new active contract with a fresh contract ID.
```daml
create Asset with issuer = bank; owner = alice; quantity = 100.0
```

### Exercise (Consuming)
Archives the target contract and executes the choice body. Sub-actions are children.
```daml
exercise assetCid Transfer with newOwner = bob
-- Archives assetCid, runs Transfer body (which may create/exercise/fetch)
```

### Exercise (Non-consuming)
Executes the choice body without archiving the target.
```daml
exercise assetCid GetBalance
-- assetCid remains active
```

### Fetch
Reads an active contract. Used for validation — verifies the contract exists and is active.
```daml
asset <- fetch assetCid
```

### Lookup by Key
Checks whether a contract with a given key exists. Returns the contract ID or None.
```daml
result <- lookupByKey @Asset (bank, "Gold")
```

## Flat vs Tree Representation

### Flat Transaction
Only shows **root-level creates and archives** — the net effect.

```json
{
  "transactionId": "tx-001",
  "events": [
    { "type": "archived", "contractId": "#1:0", "templateId": "Main:Asset" },
    { "type": "created", "contractId": "#2:0", "templateId": "Main:Asset",
      "arguments": { "owner": "Bob", ... } }
  ]
}
```

### Transaction Tree
Shows **full causality** — exercises with children.

```json
{
  "transactionId": "tx-001",
  "rootEventIds": ["#ev0"],
  "eventsById": {
    "#ev0": {
      "type": "exercised",
      "contractId": "#1:0",
      "choice": "Transfer",
      "childEventIds": ["#ev1", "#ev2"]
    },
    "#ev1": { "type": "archived", "contractId": "#1:0" },
    "#ev2": { "type": "created", "contractId": "#2:0", "arguments": {...} }
  }
}
```

## Rollback Nodes (Exception Handling)

DAML supports `try-catch` for controlled failure within a transaction.

```daml
choice TryTransfer : Optional (ContractId Asset)
  controller owner
  do
    try do
      newCid <- exercise assetCid Transfer with newOwner = bob
      return (Some newCid)
    catch
      (_ : TransferFailed) -> return None
```

**Rollback semantics:**
- Sub-actions inside the `try` block that failed become **rollback nodes**
- Rollback nodes appear in the transaction tree but their effects are **NOT committed**
- The outer transaction continues (unless it also fails)
- Informees of rollback nodes still see them (for audit purposes)

## Atomicity

- A transaction is **all-or-nothing** — either all actions commit or none do
- This holds even for multi-participant transactions (via Canton commit protocol)
- If any validation fails (authorization, contract not active, ensure clause), the **entire transaction** is rejected

## Transaction ID and Event IDs

```
Transaction ID: "tx-abc123"        -- Globally unique
  Event ID: "#ev1"                  -- Unique within transaction
    Contract ID: "#1:0"             -- Globally unique, assigned at creation
```

## Inspecting Transactions

### DAML Script
```daml
tree <- submitTree alice do
  exerciseCmd assetCid Transfer with newOwner = bob
-- tree contains the full transaction tree
```

### Ledger API (gRPC)
```java
// Get transaction tree by ID
var tree = client.getTransactionsClient()
    .getTransactionById(txId, Set.of("Alice::ns"));

// Walk the tree
for (String rootId : tree.getRootEventIdsList()) {
    var event = tree.getEventsByIdMap().get(rootId);
    if (event.hasExercised()) {
        // Process exercised event and recurse into children
        for (String childId : event.getExercised().getChildEventIdsList()) {
            // ...
        }
    }
}
```

### JSON API
```bash
curl -X POST http://localhost:7575/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"templateIds": ["Main:Asset"]}'
```

## Best Practices

1. **Use flat transactions for state tracking** — Simpler, shows net effect
2. **Use tree transactions for audit/debug** — Full causality chain
3. **Design for atomicity** — Group related operations in a single transaction
4. **Handle rollbacks explicitly** — Use try-catch only when partial failure is acceptable
5. **Log transaction IDs** — Essential for debugging and correlation
