---
name: ledger-api-commands
description: Command submission patterns — synchronous and asynchronous submission, deduplication, completion tracking, and error recovery.
---

# Ledger API Command Submission

Commands are the only way to mutate ledger state. Understanding submission patterns, deduplication, and error recovery is critical for reliable applications.

## When to Activate

- Submitting transactions to the ledger
- Implementing reliable command submission with retries
- Handling command deduplication
- Tracking command completion asynchronously
- Building automation bots

## Synchronous Submission (Recommended)

Use `CommandService` for most cases — it submits and waits for the result.

### Java

```java
import com.daml.ledger.api.v1.CommandServiceGrpc;
import com.daml.ledger.api.v1.CommandServiceOuterClass.*;

var commands = Commands.newBuilder()
    .setApplicationId("my-app")
    .setCommandId(UUID.randomUUID().toString())
    .setWorkflowId("settlement-batch")
    .setParty("Alice::namespace")
    .addCommands(Command.newBuilder()
        .setCreate(CreateCommand.newBuilder()
            .setTemplateId(Identifier.newBuilder()
                .setModuleName("Main")
                .setEntityName("Asset")
                .build())
            .setCreateArguments(Record.newBuilder()
                .addFields(RecordField.newBuilder()
                    .setLabel("issuer")
                    .setValue(Value.newBuilder().setParty("Bank::ns")))
                .addFields(RecordField.newBuilder()
                    .setLabel("owner")
                    .setValue(Value.newBuilder().setParty("Alice::ns")))
                .addFields(RecordField.newBuilder()
                    .setLabel("quantity")
                    .setValue(Value.newBuilder().setNumeric("100.0")))
                .build())
            .build())
        .build())
    .build();

// Submit and wait for transaction ID
var response = commandService.submitAndWaitForTransactionId(
    SubmitAndWaitRequest.newBuilder().setCommands(commands).build()
);
String txId = response.getTransactionId();
```

### TypeScript (JSON API)

```typescript
// Create
const contract = await ledger.create(Asset, {
  issuer: 'Bank',
  owner: 'Alice',
  quantity: '100.0',
});

// Exercise
await ledger.exercise(Asset.Transfer, contract.contractId, {
  newOwner: 'Bob',
});

// Create and exercise
const result = await ledger.createAndExercise(Asset, {
  issuer: 'Bank',
  owner: 'Bank',
  quantity: '100.0',
}, Asset.Transfer, {
  newOwner: 'Alice',
});
```

### curl (JSON API)

```bash
# Create
curl -X POST http://localhost:7575/v1/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateId": "Main:Asset",
    "payload": {
      "issuer": "Bank",
      "owner": "Alice",
      "quantity": "100.0"
    }
  }'

# Exercise
curl -X POST http://localhost:7575/v1/exercise \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateId": "Main:Asset",
    "contractId": "#1:0",
    "choice": "Transfer",
    "argument": { "newOwner": "Bob" }
  }'
```

## Asynchronous Submission

For high-throughput scenarios, use `CommandSubmissionService` + `CommandCompletionService`.

```java
// 1. Submit (returns immediately)
commandSubmissionService.submit(
    SubmitRequest.newBuilder().setCommands(commands).build()
);

// 2. Track completion
var completionStream = commandCompletionService.completionStream(
    CompletionStreamRequest.newBuilder()
        .setApplicationId("my-app")
        .addParties("Alice::ns")
        .setOffset(LedgerOffset.newBuilder()
            .setBoundary(LedgerOffset.LedgerBoundary.LEDGER_END))
        .build()
);

completionStream.forEach(response -> {
    for (var completion : response.getCompletionsList()) {
        if (completion.getStatus().getCode() == 0) {
            System.out.println("Success: " + completion.getTransactionId());
        } else {
            System.out.println("Failed: " + completion.getStatus().getMessage());
        }
    }
});
```

## Command Deduplication

The ledger deduplicates commands based on `(command_id, application_id, act_as)`.

```java
// Same command_id = deduplicated (safe to retry)
var commands = Commands.newBuilder()
    .setCommandId("idempotent-transfer-001")  // Stable ID
    .setApplicationId("my-app")
    .setParty("Alice::ns")
    .setDeduplicationPeriod(Duration.newBuilder().setSeconds(300))
    .addCommands(...)
    .build();
```

**Rules:**
- Within the dedup period, resubmitting the same `command_id` returns `ALREADY_EXISTS`
- After the dedup period, the same `command_id` is treated as a new command
- Use deterministic `command_id` for idempotent operations

## Error Recovery Pattern

```java
CompletableFuture<String> submitWithRetry(Commands commands, int maxRetries) {
    for (int attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            var response = commandService.submitAndWaitForTransactionId(
                SubmitAndWaitRequest.newBuilder()
                    .setCommands(commands).build());
            return CompletableFuture.completedFuture(response.getTransactionId());
        } catch (StatusRuntimeException e) {
            switch (e.getStatus().getCode()) {
                case ALREADY_EXISTS:
                    // Command was already accepted (dedup), treat as success
                    return CompletableFuture.completedFuture("deduplicated");
                case ABORTED:
                    // Contention — retry with backoff
                    Thread.sleep((long) Math.pow(2, attempt) * 100);
                    continue;
                case UNAVAILABLE:
                    // Service down — retry with backoff
                    Thread.sleep((long) Math.pow(2, attempt) * 1000);
                    continue;
                case INVALID_ARGUMENT:
                case PERMISSION_DENIED:
                    // Don't retry — fix the problem
                    throw e;
                default:
                    throw e;
            }
        }
    }
    throw new RuntimeException("Max retries exceeded");
}
```

## Batch Command Submission

Submit multiple commands in a single transaction (atomic).

```java
var commands = Commands.newBuilder()
    .setCommandId("batch-001")
    .setParty("Alice::ns")
    .addCommands(Command.newBuilder().setExercise(/* archive old */))
    .addCommands(Command.newBuilder().setCreate(/* create new */))
    .addCommands(Command.newBuilder().setCreate(/* create receipt */))
    .build();
// All three commands execute atomically
```

## Best Practices

1. **Use stable command IDs for idempotent operations** — Enables safe retries
2. **Set appropriate dedup periods** — Too short risks duplicates; too long uses memory
3. **Always handle ABORTED with retry** — Contention is normal in concurrent systems
4. **Use SubmitAndWait unless you need async** — Simpler error handling
5. **Log command IDs** — Essential for debugging and audit trails
6. **Batch related mutations** — Atomic transactions prevent inconsistent state
