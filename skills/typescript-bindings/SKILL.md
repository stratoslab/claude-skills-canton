---
name: typescript-bindings
description: DAML TypeScript bindings — @daml/ledger, @daml/react hooks, JSON API interaction, codegen, and frontend integration patterns.
---

# TypeScript Bindings for DAML

Build web applications that interact with the DAML ledger via the JSON API using type-safe TypeScript clients and React hooks.

## When to Activate

- Building web frontends for Canton applications
- Setting up JSON API and TypeScript codegen
- Using React hooks for DAML
- Implementing real-time contract streaming in the browser

## Setup

### Install Packages

```bash
npm install @daml/types @daml/ledger @daml/react
```

### Code Generation

```bash
# Generate TypeScript types from DAR
daml codegen js -o daml.js .daml/dist/my-app-1.0.0.dar

# Install generated package
cd daml.js/my-app-1.0.0
npm install
npm run build
cd ../..
npm install ./daml.js/my-app-1.0.0
```

### Start JSON API

```bash
daml json-api \
  --ledger-host localhost \
  --ledger-port 6865 \
  --http-port 7575 \
  --allow-insecure-tokens  # Dev only
```

## @daml/ledger (Core Client)

```typescript
import Ledger from '@daml/ledger';
import { Asset, Transfer } from '@daml.js/my-app';

const ledger = new Ledger({
  token: jwtToken,
  httpBaseUrl: 'http://localhost:7575',
});

// Create a contract
const { contractId } = await ledger.create(Asset, {
  issuer: 'Bank',
  owner: 'Alice',
  description: 'Gold',
  quantity: '100.0',
});

// Exercise a choice
const result = await ledger.exercise(
  Asset.Transfer,
  contractId,
  { newOwner: 'Bob' }
);

// Query active contracts
const assets = await ledger.query(Asset);                 // All
const myAssets = await ledger.query(Asset, { owner: 'Alice' }); // Filtered

// Fetch by contract ID
const asset = await ledger.fetch(Asset, contractId);

// Fetch by key
const assetByKey = await ledger.fetchByKey(Asset, {
  _1: 'Bank',
  _2: 'Gold'
});

// Create and exercise atomically
const { exerciseResult } = await ledger.createAndExercise(
  Asset,
  { issuer: 'Bank', owner: 'Bank', description: 'Silver', quantity: '50.0' },
  Asset.Transfer,
  { newOwner: 'Alice' }
);
```

## @daml/react (React Hooks)

### Provider Setup

```tsx
import { DamlLedger } from '@daml/react';

function App() {
  return (
    <DamlLedger
      token={userToken}
      httpBaseUrl="http://localhost:7575"
      wsBaseUrl="ws://localhost:7575"
    >
      <Dashboard />
    </DamlLedger>
  );
}
```

### useQuery (One-Shot)

```tsx
import { useQuery } from '@daml/react';
import { Asset } from '@daml.js/my-app';

function AssetList() {
  const { contracts, loading } = useQuery(Asset);

  if (loading) return <p>Loading...</p>;

  return (
    <ul>
      {contracts.map(c => (
        <li key={c.contractId}>
          {c.payload.description}: {c.payload.quantity}
        </li>
      ))}
    </ul>
  );
}
```

### useStreamQuery (Real-Time)

```tsx
import { useStreamQuery } from '@daml/react';

function LiveAssets({ party }: { party: string }) {
  const { contracts, loading } = useStreamQuery(
    Asset,
    () => ({ owner: party }),
    [party]
  );

  return (
    <div>
      <h2>Assets ({contracts.length})</h2>
      {contracts.map(c => (
        <AssetCard key={c.contractId} asset={c.payload} />
      ))}
    </div>
  );
}
```

### useLedger (Actions)

```tsx
import { useLedger } from '@daml/react';

function TransferButton({ contractId, newOwner }) {
  const ledger = useLedger();
  const [loading, setLoading] = useState(false);

  const handleTransfer = async () => {
    setLoading(true);
    try {
      await ledger.exercise(Asset.Transfer, contractId, { newOwner });
    } catch (error) {
      console.error('Transfer failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button onClick={handleTransfer} disabled={loading}>
      Transfer to {newOwner}
    </button>
  );
}
```

### useFetchByKey

```tsx
import { useFetchByKey } from '@daml/react';

function AccountView({ bank, accountNum }) {
  const { contract, loading } = useFetchByKey(
    Account,
    () => ({ _1: bank, _2: accountNum }),
    [bank, accountNum]
  );

  if (!contract) return <p>Account not found</p>;
  return <p>Balance: {contract.payload.balance}</p>;
}
```

## Direct JSON API (fetch/axios)

```typescript
// Query
const response = await fetch('http://localhost:7575/v1/query', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    templateIds: ['Main:Asset'],
    query: { owner: 'Alice' },
  }),
});
const { result } = await response.json();

// Create
await fetch('http://localhost:7575/v1/create', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({
    templateId: 'Main:Asset',
    payload: { issuer: 'Bank', owner: 'Alice', description: 'Gold', quantity: '100.0' },
  }),
});

// Server-Sent Events streaming
const eventSource = new EventSource(
  'http://localhost:7575/v1/stream/query',
  { headers: { 'Authorization': `Bearer ${token}` } }
);
eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Contract update:', data);
};
```

## JWT Token for Dev

```typescript
// For sandbox with --allow-insecure-tokens
function makeToken(party: string): string {
  const payload = {
    "https://daml.com/ledger-api": {
      ledgerId: null,
      applicationId: "my-app",
      actAs: [party],
      readAs: [party],
    },
    exp: Math.floor(Date.now() / 1000) + 86400,
  };
  // In dev, use unsigned JWT (sandbox accepts it with --allow-insecure-tokens)
  return btoa(JSON.stringify({alg: "HS256"})) + '.' +
         btoa(JSON.stringify(payload)) + '.fake-signature';
}
```

## Best Practices

1. **Use @daml/react for React apps** — Handles streaming and lifecycle
2. **Use useStreamQuery for live data** — Automatic updates via WebSocket
3. **Use codegen types** — Catches template/choice mismatches at compile time
4. **Handle loading states** — Always show loading indicators
5. **Implement error boundaries** — Catch and display ledger errors gracefully
6. **Use JSON API for web, gRPC for backend** — Match protocol to environment
