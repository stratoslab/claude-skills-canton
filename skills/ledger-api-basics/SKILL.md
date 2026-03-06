---
name: ledger-api-basics
description: DAML Ledger API fundamentals — gRPC services, authentication, connection setup, and core concepts for client applications.
---

# Ledger API Basics

The Ledger API is the primary programmatic interface to a DAML ledger. It's a gRPC API that allows applications to submit commands, read transactions, and query active contracts.

## When to Activate

- Building client applications for Canton
- Setting up Ledger API connections
- Understanding available gRPC services
- Configuring authentication
- Choosing between gRPC and JSON API

## Core Services

| Service | Purpose |
|---------|---------|
| **CommandService** | Synchronous command submission (submit + wait) |
| **CommandSubmissionService** | Async command submission (fire and forget) |
| **CommandCompletionService** | Track command completion status |
| **TransactionService** | Read transaction streams (flat or tree) |
| **ActiveContractsService** | Snapshot of active contracts |
| **PackageService** | Query uploaded packages |
| **PackageManagementService** | Upload DAR files |
| **PartyManagementService** | Allocate and list parties |
| **VersionService** | Query API version |

## Connection Setup

### gRPC (Direct)

```java
// Java
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("localhost", 6865)
    .usePlaintext()  // Dev only! Use TLS in production
    .build();
```

```typescript
// TypeScript (via JSON API)
const ledger = new Ledger({
  token: jwtToken,
  httpBaseUrl: 'http://localhost:7575',
});
```

### With TLS

```java
ManagedChannel channel = NettyChannelBuilder
    .forAddress("participant.example.com", 5011)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(new File("/certs/ca.crt"))
        .build())
    .build();
```

## Authentication

### JWT Token Structure

```json
{
  "https://daml.com/ledger-api": {
    "ledgerId": null,
    "applicationId": "my-app",
    "actAs": ["Alice::namespace123"],
    "readAs": ["Alice::namespace123", "PublicParty::namespace456"],
    "admin": false
  },
  "exp": 1700000000
}
```

### Token Claims

| Claim | Description |
|-------|-------------|
| `actAs` | Parties the token holder can submit commands as |
| `readAs` | Parties the token holder can read data for |
| `admin` | Whether the token grants admin operations |
| `applicationId` | Application identifier (optional) |

### Sandbox (No Auth)

```bash
# Start sandbox without auth (development only)
daml sandbox

# Start with JWT auth
daml sandbox --auth=jwt-rs-256-jwks --auth-jwt-rs256-jwks=https://...
```

## Key Concepts

### Command ID and Deduplication

Every command submission requires a unique `command_id` + `application_id` + `act_as` combination. The ledger deduplicates commands within the deduplication period.

```
command_id: "transfer-001"     # Client-chosen, unique per app+party
application_id: "my-app"      # Identifies the application
act_as: ["Alice"]              # Submitting party
deduplication_period: 300s     # Window for dedup
```

### Workflow ID

Optional grouping identifier across related commands.

```
workflow_id: "batch-settlement-2024-01-15"
```

### Offsets

Monotonically increasing positions in the transaction stream. Used for:
- Resuming transaction subscriptions after reconnection
- Specifying where to start reading transactions
- Crash recovery

```
LEDGER_BEGIN  → Start from genesis
LEDGER_END   → Start from now (only future transactions)
"000000123"  → Resume from specific offset (opaque string)
```

## Command Types

### CreateCommand

```json
{
  "templateId": {
    "packageId": "abc123",
    "moduleName": "Main",
    "entityName": "Asset"
  },
  "createArguments": {
    "fields": [
      {"label": "issuer", "value": {"party": "Alice"}},
      {"label": "owner", "value": {"party": "Alice"}},
      {"label": "quantity", "value": {"numeric": "100.0"}}
    ]
  }
}
```

### ExerciseCommand

```json
{
  "templateId": {"moduleName": "Main", "entityName": "Asset"},
  "contractId": "#1:0",
  "choice": "Transfer",
  "choiceArgument": {
    "fields": [
      {"label": "newOwner", "value": {"party": "Bob"}}
    ]
  }
}
```

### ExerciseByKeyCommand

```json
{
  "templateId": {"moduleName": "Main", "entityName": "Asset"},
  "contractKey": {
    "record": {
      "fields": [
        {"value": {"party": "Bank"}},
        {"value": {"text": "Gold"}}
      ]
    }
  },
  "choice": "Transfer",
  "choiceArgument": { ... }
}
```

## Error Handling

| gRPC Status | Meaning | Action |
|-------------|---------|--------|
| `ALREADY_EXISTS` | Duplicate command | Safe to ignore (dedup) |
| `NOT_FOUND` | Contract archived or doesn't exist | Application logic error |
| `INVALID_ARGUMENT` | Malformed command | Fix the command, don't retry |
| `ABORTED` | Contention/conflict | Retry with backoff |
| `PERMISSION_DENIED` | Auth failure | Check JWT token |
| `DEADLINE_EXCEEDED` | Timeout | Check completion, then retry |
| `UNAVAILABLE` | Service unavailable | Retry with backoff |

## JSON API vs gRPC

| Feature | JSON API | gRPC |
|---------|----------|------|
| Protocol | HTTP/JSON | HTTP/2 + Protobuf |
| Use case | Web apps, REST clients | Backend services, high throughput |
| Streaming | Server-Sent Events | gRPC streaming |
| Port | 7575 (default) | 6865 (sandbox) or 5011 (Canton) |
| Auth | Bearer token in header | Bearer token in metadata |
| Codegen | TypeScript types | Java classes |

## Best Practices

1. **Use CommandService.SubmitAndWait for simplicity** — Handles completion tracking for you
2. **Always set command_id** — Enables deduplication and tracking
3. **Persist offsets** — For crash recovery of transaction stream consumers
4. **Use exponential backoff for ABORTED** — Contention resolves with retry
5. **Prefer gRPC for backends, JSON API for frontends** — Match protocol to client type
6. **Never hardcode package IDs** — They change with every DAR build
