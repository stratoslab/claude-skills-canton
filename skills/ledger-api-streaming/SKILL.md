---
name: ledger-api-streaming
description: Transaction stream subscriptions — flat vs tree transactions, offset management, filtering, crash recovery, and real-time event processing.
---

# Ledger API Transaction Streaming

Transaction streams provide a real-time feed of ledger changes. Essential for building reactive applications, automation bots, and event-driven architectures.

## When to Activate

- Building real-time applications reacting to ledger events
- Implementing event-driven automation
- Synchronizing external systems with ledger state
- Building read models / projections from ledger data

## Two Stream Types

### Flat Transactions
Shows the **net effect** — only `CreatedEvent` and `ArchivedEvent`.

```
Transaction {
  events: [
    ArchivedEvent { contractId: "#1:0", templateId: "Main:Asset" },
    CreatedEvent  { contractId: "#2:0", templateId: "Main:Asset", arguments: {...} }
  ]
}
```

### Transaction Trees
Shows the **full causality chain** — includes `ExercisedEvent` with parent/child relationships.

```
TransactionTree {
  rootEventIds: ["#ev1"],
  eventsById: {
    "#ev1": ExercisedEvent {
      choice: "Transfer",
      childEventIds: ["#ev2", "#ev3"],
    },
    "#ev2": ArchivedEvent { contractId: "#1:0" },
    "#ev3": CreatedEvent  { contractId: "#2:0", ... }
  }
}
```

**Use flat transactions** for simple state tracking. **Use transaction trees** when you need to understand why things happened.

## Subscription Pattern

### The ACS + Stream Bootstrap

```
1. GetActiveContracts()          → Snapshot of current state
   Returns: [CreatedEvent, ...] + final offset

2. GetTransactions(begin=offset) → Continuous stream from snapshot
   Returns: stream of Transaction messages
```

This guarantees a **complete, gap-free view** of the ledger.

### Java Implementation

```java
// Step 1: Get ACS snapshot
AtomicReference<String> latestOffset = new AtomicReference<>();
Map<String, CreatedEvent> activeContracts = new ConcurrentHashMap<>();

client.getActiveContractSetClient()
    .getActiveContracts(filter, true)
    .blockingForEach(response -> {
        for (CreatedEvent ce : response.getCreatedEvents()) {
            activeContracts.put(ce.getContractId(), ce);
        }
        if (!response.getOffset().isEmpty()) {
            latestOffset.set(response.getOffset());
        }
    });

// Step 2: Subscribe to transaction stream from ACS offset
client.getTransactionsClient()
    .getTransactions(
        new LedgerOffset.Absolute(latestOffset.get()),
        filter,
        true)
    .forEach(tx -> {
        for (Event event : tx.getEvents()) {
            if (event instanceof CreatedEvent) {
                activeContracts.put(event.getContractId(), (CreatedEvent) event);
            } else if (event instanceof ArchivedEvent) {
                activeContracts.remove(event.getContractId());
            }
        }
        latestOffset.set(tx.getOffset());
        persistOffset(tx.getOffset()); // For crash recovery
    });
```

### TypeScript Implementation

```typescript
// Using @daml/ledger with JSON API streaming (SSE)
const stream = ledger.streamQuery(Asset, { owner: party });

stream.on('live', (contracts) => {
  console.log('Initial ACS:', contracts);
});

stream.on('change', (contracts) => {
  console.log('Updated contracts:', contracts);
});

stream.on('close', (event) => {
  console.log('Stream closed, reconnecting...');
  // Implement reconnection logic
});
```

## Filtering

### By Party
```java
TransactionFilter filter = new FiltersByParty(Map.of(
    "Alice::ns", NoFilter.instance,     // All templates for Alice
    "Bob::ns", NoFilter.instance        // All templates for Bob
));
```

### By Template
```java
TransactionFilter filter = new FiltersByParty(Map.of(
    "Alice::ns", new InclusiveFilter(Set.of(
        Identifier.of("pkg-id", "Main", "Asset"),
        Identifier.of("pkg-id", "Main", "Trade")
    ))
));
```

## Offset Management

### Starting Points
```java
LedgerOffset.LedgerBegin    // From genesis (replay everything)
LedgerOffset.LedgerEnd      // From now (only future events)
new LedgerOffset.Absolute("000000123")  // Resume from saved offset
```

### Crash Recovery

```java
// On startup: load last persisted offset
String savedOffset = loadOffsetFromDatabase();

LedgerOffset begin;
if (savedOffset != null) {
    begin = new LedgerOffset.Absolute(savedOffset);
} else {
    // First run: bootstrap from ACS
    begin = bootstrapFromACS();
}

// Subscribe with recovery offset
client.getTransactionsClient()
    .getTransactions(begin, filter, true)
    .forEach(tx -> {
        processTransaction(tx);
        persistOffset(tx.getOffset());
    });
```

### Offset Persistence Strategies

| Strategy | Consistency | Performance |
|----------|-------------|-------------|
| Per-transaction commit | Exactly-once | Slower |
| Batch commit (every N) | At-least-once (may replay) | Faster |
| Async commit | At-least-once | Fastest |

## Handling Reconnection

```java
while (true) {
    try {
        subscribeToTransactions(loadOffset());
    } catch (StatusRuntimeException e) {
        if (e.getStatus().getCode() == Status.Code.UNAVAILABLE) {
            Thread.sleep(backoff());
            continue;
        }
        throw e;
    }
}
```

## Best Practices

1. **Always bootstrap from ACS + stream** — Ensures complete view
2. **Persist offsets** — Essential for crash recovery
3. **Use flat transactions for state tracking** — Simpler, lower overhead
4. **Use transaction trees for audit/debugging** — Full causality
5. **Filter by template when possible** — Reduces network bandwidth
6. **Handle stream disconnection** — Implement reconnection with offset resume
7. **Process events idempotently** — In case of replays after crash recovery
