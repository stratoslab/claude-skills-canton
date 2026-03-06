---
name: java-bindings
description: DAML Java bindings — codegen, client setup, command submission, transaction streams, and reactive patterns with RxJava.
---

# Java Bindings for DAML

Build Java/Kotlin backend applications that interact with the DAML Ledger API using generated type-safe classes.

## When to Activate

- Building Java/Kotlin backend services for Canton
- Setting up DAML codegen for Java
- Implementing reactive transaction processing
- Building automation bots in Java

## Setup

### Maven Dependencies

```xml
<properties>
  <daml.sdk.version>2.10.0</daml.sdk.version>
</properties>

<dependencies>
  <dependency>
    <groupId>com.daml</groupId>
    <artifactId>bindings-java</artifactId>
    <version>${daml.sdk.version}</version>
  </dependency>
  <dependency>
    <groupId>com.daml</groupId>
    <artifactId>bindings-rxjava</artifactId>
    <version>${daml.sdk.version}</version>
  </dependency>
</dependencies>
```

### Code Generation

```bash
# Generate Java classes from DAR
daml codegen java

# Produces classes in target/generated-sources/
# com.daml.model.<ModuleName>.<TemplateName>
```

## Client Connection

```java
import com.daml.ledger.rxjava.DamlLedgerClient;

// Plain text (dev only)
DamlLedgerClient client = DamlLedgerClient.newBuilder("localhost", 6865)
    .build();
client.connect();

// With JWT auth
DamlLedgerClient client = DamlLedgerClient.newBuilder("localhost", 5011)
    .withAccessToken("eyJhbG...")
    .withSslContext(sslContext)  // TLS
    .build();
client.connect();
```

## Command Submission

```java
import com.daml.model.main.Asset;

// Create using generated class
var createCmd = new Asset("Bank::ns", "Alice::ns", "Gold", new BigDecimal("100.0"))
    .create();

client.getCommandClient().submitAndWait(
    "workflow-001",
    "my-app",
    UUID.randomUUID().toString(),
    "Alice::ns",
    List.of(createCmd)
).blockingGet();

// Exercise using generated class
var exerciseCmd = contractId.exerciseTransfer("Bob::ns");

client.getCommandClient().submitAndWait(
    "workflow-001", "my-app", UUID.randomUUID().toString(),
    "Alice::ns", List.of(exerciseCmd)
).blockingGet();
```

## Transaction Streams (RxJava)

```java
import io.reactivex.Flowable;

TransactionFilter filter = new FiltersByParty(Map.of(
    "Alice::ns", NoFilter.instance
));

// Flat transactions
Flowable<Transaction> txStream = client.getTransactionsClient()
    .getTransactions(LedgerOffset.LedgerBegin.getInstance(), filter, true);

txStream.forEach(tx -> {
    for (Event event : tx.getEvents()) {
        if (event instanceof CreatedEvent ce) {
            if (Asset.TEMPLATE_ID.equals(ce.getTemplateId())) {
                Asset.Contract asset = Asset.Contract.fromCreatedEvent(ce);
                System.out.println("New asset: " + asset.data.description);
            }
        } else if (event instanceof ArchivedEvent ae) {
            System.out.println("Archived: " + ae.getContractId());
        }
    }
});
```

## ACS Bootstrap + Stream

```java
AtomicReference<LedgerOffset> offset = new AtomicReference<>();
Map<String, Asset.Contract> assets = new ConcurrentHashMap<>();

// Step 1: ACS snapshot
client.getActiveContractSetClient()
    .getActiveContracts(filter, true)
    .blockingForEach(response -> {
        for (CreatedEvent ce : response.getCreatedEvents()) {
            if (Asset.TEMPLATE_ID.equals(ce.getTemplateId())) {
                var contract = Asset.Contract.fromCreatedEvent(ce);
                assets.put(ce.getContractId(), contract);
            }
        }
        if (!response.getOffset().isEmpty()) {
            offset.set(new LedgerOffset.Absolute(response.getOffset()));
        }
    });

// Step 2: Live stream from ACS offset
client.getTransactionsClient()
    .getTransactions(offset.get(), filter, true)
    .forEach(tx -> {
        for (Event event : tx.getEvents()) {
            if (event instanceof CreatedEvent ce) {
                assets.put(ce.getContractId(), Asset.Contract.fromCreatedEvent(ce));
            } else if (event instanceof ArchivedEvent ae) {
                assets.remove(ae.getContractId());
            }
        }
    });
```

## Bot Pattern

```java
public class TransferBot {
    private final DamlLedgerClient client;
    private final String party;

    public void run() {
        TransactionFilter filter = new FiltersByParty(Map.of(
            party, new InclusiveFilter(Set.of(TransferProposal.TEMPLATE_ID))
        ));

        client.getTransactionsClient()
            .getTransactions(LedgerOffset.LedgerEnd.getInstance(), filter, true)
            .forEach(tx -> {
                for (Event event : tx.getEvents()) {
                    if (event instanceof CreatedEvent ce) {
                        var proposal = TransferProposal.Contract.fromCreatedEvent(ce);
                        // Auto-accept proposals under $1000
                        if (proposal.data.amount.compareTo(new BigDecimal("1000")) < 0) {
                            client.getCommandClient().submitAndWait(
                                "auto-accept", "transfer-bot",
                                UUID.randomUUID().toString(), party,
                                List.of(ce.getContractId().exerciseAccept())
                            ).blockingGet();
                        }
                    }
                }
            });
    }
}
```

## Best Practices

1. **Use generated classes** — Type-safe, catches errors at compile time
2. **Use RxJava for streams** — Proper backpressure handling
3. **Persist offsets** — For crash recovery
4. **Close clients on shutdown** — `client.close()` releases gRPC resources
5. **Use connection pools** — Reuse gRPC channels across threads
