---
name: ledger-api-queries
description: Active contract set queries — ACS snapshots, filtering by template and party, contract key lookups, and building read models.
---

# Ledger API Queries

Query the Active Contract Set (ACS) to get the current state of the ledger. The ACS contains all contracts that have been created but not yet archived.

## When to Activate

- Querying current ledger state
- Building dashboards or read models
- Looking up specific contracts by key
- Implementing search and filter functionality
- Bootstrapping application state on startup

## ACS Query (gRPC)

### Basic Query

```java
TransactionFilter filter = new FiltersByParty(Map.of(
    "Alice::ns", NoFilter.instance  // All templates visible to Alice
));

List<CreatedEvent> contracts = new ArrayList<>();
AtomicReference<String> offset = new AtomicReference<>();

client.getActiveContractSetClient()
    .getActiveContracts(filter, true)  // verbose=true for full record
    .blockingForEach(response -> {
        contracts.addAll(response.getCreatedEvents());
        if (!response.getOffset().isEmpty()) {
            offset.set(response.getOffset());
        }
    });
```

### Filtered by Template

```java
TransactionFilter filter = new FiltersByParty(Map.of(
    "Alice::ns", new InclusiveFilter(Set.of(
        Identifier.of("pkg-id", "Main", "Asset")
    ))
));
```

## JSON API Queries

### Query All Contracts of a Template

```bash
curl -X POST http://localhost:7575/v1/query \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateIds": ["Main:Asset"]
  }'
```

### Query with Filter

```bash
curl -X POST http://localhost:7575/v1/query \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateIds": ["Main:Asset"],
    "query": {
      "owner": "Alice",
      "quantity": { "%gte": 100 }
    }
  }'
```

### Fetch by Contract ID

```bash
curl -X POST http://localhost:7575/v1/fetch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "contractId": "#1:0"
  }'
```

### Fetch by Contract Key

```bash
curl -X POST http://localhost:7575/v1/fetch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateId": "Main:Asset",
    "key": ["Bank", "Gold"]
  }'
```

## TypeScript Queries

```typescript
import Ledger from '@daml/ledger';
import { Asset } from '@daml.js/my-model';

const ledger = new Ledger({ token, httpBaseUrl: 'http://localhost:7575' });

// Query all assets
const allAssets = await ledger.query(Asset);

// Query with filter
const myAssets = await ledger.query(Asset, { owner: 'Alice' });

// Fetch by contract ID
const asset = await ledger.fetch(Asset, contractId);

// Fetch by key
const assetByKey = await ledger.fetchByKey(Asset, { _1: 'Bank', _2: 'Gold' });

// Streaming query (live updates)
const stream = ledger.streamQuery(Asset, { owner: 'Alice' });
stream.on('change', (contracts) => {
  console.log('Current assets:', contracts);
});
```

## React Hooks

```typescript
import { useQuery, useStreamQuery, useFetchByKey } from '@daml/react';
import { Asset } from '@daml.js/my-model';

function AssetList() {
  // One-shot query
  const { contracts, loading } = useQuery(Asset);

  // Live streaming query
  const { contracts: liveContracts } = useStreamQuery(Asset, () => ({
    owner: currentParty,
  }), [currentParty]);

  // Fetch by key
  const { contract } = useFetchByKey(Asset, () => ({
    _1: 'Bank', _2: 'Gold',
  }), []);

  return (
    <ul>
      {liveContracts.map(c => (
        <li key={c.contractId}>
          {c.payload.description}: {c.payload.quantity}
        </li>
      ))}
    </ul>
  );
}
```

## Building Read Models

### Pattern: Project ACS into a Database

```java
// On startup
void initializeReadModel() {
    // 1. Clear stale data
    db.execute("TRUNCATE TABLE assets");

    // 2. Load ACS
    client.getActiveContractSetClient()
        .getActiveContracts(filter, true)
        .blockingForEach(response -> {
            for (CreatedEvent ce : response.getCreatedEvents()) {
                insertAsset(ce);
            }
            lastOffset = response.getOffset();
        });

    // 3. Subscribe to updates
    client.getTransactionsClient()
        .getTransactions(new LedgerOffset.Absolute(lastOffset), filter, true)
        .forEach(tx -> {
            db.beginTransaction();
            for (Event event : tx.getEvents()) {
                if (event instanceof CreatedEvent) {
                    insertAsset((CreatedEvent) event);
                } else if (event instanceof ArchivedEvent) {
                    deleteAsset(((ArchivedEvent) event).getContractId());
                }
            }
            db.updateOffset(tx.getOffset());
            db.commit();
        });
}
```

## JSON API Query Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| (none) | Exact match | `{"owner": "Alice"}` |
| `%lte` | Less than or equal | `{"quantity": {"%lte": 100}}` |
| `%lt` | Less than | `{"quantity": {"%lt": 100}}` |
| `%gte` | Greater than or equal | `{"quantity": {"%gte": 50}}` |
| `%gt` | Greater than | `{"quantity": {"%gt": 0}}` |

## Best Practices

1. **Use template filters** — Don't query all templates when you only need one
2. **Use ACS + stream for real-time** — Point-in-time queries miss updates
3. **Cache locally** — Build read models in your application database
4. **Use contract keys for lookups** — More stable than contract IDs
5. **Handle large ACS carefully** — Paginate or stream for large datasets
6. **Visibility is party-scoped** — Queries only return contracts visible to the authenticated party
