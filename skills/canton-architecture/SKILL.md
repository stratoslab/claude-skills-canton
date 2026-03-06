---
name: canton-architecture
description: Canton Network architecture — participants, synchronizers (domains), sequencers, mediators, topology management, and the synchronization protocol.
---

# Canton Network Architecture

Canton is the synchronization protocol for DAML ledgers. It enables multi-party workflows across organizational boundaries with sub-transaction privacy, composability, and GDPR compliance.

## When to Activate

- Designing Canton network topologies
- Understanding transaction flow and commit protocol
- Planning multi-participant deployments
- Configuring synchronizer components
- Troubleshooting consensus or connectivity issues

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Canton Network                            │
│                                                              │
│  ┌──────────────┐          ┌──────────────────────────────┐ │
│  │ Participant 1 │─────────│      Synchronizer (Domain)    │ │
│  │  - Ledger API │         │  ┌───────────┐ ┌──────────┐  │ │
│  │  - DAML Engine│         │  │ Sequencer │ │ Mediator │  │ │
│  │  - Contract   │         │  └───────────┘ └──────────┘  │ │
│  │    Store      │         │  ┌─────────────────────────┐  │ │
│  └──────────────┘         │  │ Topology Manager        │  │ │
│                            │  └─────────────────────────┘  │ │
│  ┌──────────────┐          │                                │ │
│  │ Participant 2 │─────────│                                │ │
│  │  - Ledger API │         └──────────────────────────────┘ │
│  │  - DAML Engine│                                           │
│  │  - Contract   │         ┌──────────────────────────────┐ │
│  │    Store      │─────────│   Synchronizer 2 (optional)   │ │
│  └──────────────┘          └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### Participant Node
The primary interface for applications. Hosts parties and executes DAML contracts.

- **Ledger API Server** — gRPC API for applications (port 5011 default)
- **Admin API** — Node administration (port 5012 default)
- **HTTP Ledger API** — JSON/HTTP interface (port 5013 default)
- **DAML Engine** — Interprets and executes DAML smart contracts
- **Transaction Protocol Processor** — Manages the Canton commit protocol
- **Contract Store** — Persistent storage for active contracts
- **Event Store** — Transaction history for the Ledger API

### Synchronizer (Domain)
Provides ordering and mediation for transactions between participants.

**Sequencer** — Total-order multicast service
- Assigns timestamps to messages
- Ensures all participants see messages in the same order
- Implementations: Database-backed (Postgres), BFT sequencer
- Does NOT see transaction contents (encrypted blobs only)

**Mediator** — Consensus coordinator
- Collects confirmation/rejection votes from participants
- Computes the final transaction verdict (approve/reject)
- Does NOT see full transaction contents
- Sees confirmation responses for privacy-protected sub-transactions

**Topology Manager** — Identity state management
- Maps parties to participants
- Manages key registrations
- Handles namespace delegations

### Identity Model

```
Unique Identifier = name :: namespace
                     ↓        ↓
               human-      cryptographic
               readable    fingerprint of
               name        root signing key
```

- **Namespace** — Derived from a root signing key; owned by one entity
- **Party ID** — `alice::1234abcd` format
- **Participant ID** — `participant1::5678efgh` format
- Identity is **decoupled from infrastructure** — same party can be hosted on multiple participants

## Synchronization Protocol (Commit Protocol)

### Transaction Flow

```
1. SUBMISSION      Submitting participant sends transaction to sequencer
                   (encrypted, only informees can decrypt their views)
      │
      ▼
2. SEQUENCING      Sequencer assigns timestamp, broadcasts to relevant participants
      │
      ▼
3. VALIDATION      Each participant validates their view using DAML engine
                   (checks authorization, contract state, ensure clauses)
      │
      ▼
4. CONFIRMATION    Participants send confirm/reject to mediator via sequencer
      │
      ▼
5. MEDIATION       Mediator collects responses, computes verdict
      │
      ▼
6. RESULT          Verdict broadcast via sequencer to all informees
      │
      ▼
7. COMMITMENT      Participants commit or rollback based on verdict
```

### Key Properties

- **Atomicity** — Transactions are all-or-nothing, even across participants
- **Sub-transaction privacy** — Each participant sees only their relevant views
- **Consistency** — Double-spend prevention via the sequencer's total order
- **Non-repudiation** — All messages are signed by their senders

## Configuration (HOCON)

### Minimal Topology

```hocon
canton {
  participants {
    participant1 {
      storage.type = memory
      ledger-api.port = 5011
      admin-api.port = 5012
    }
  }

  sequencers {
    sequencer1 {
      storage.type = memory
      public-api.port = 5018
      admin-api.port = 5019
      sequencer.type = reference   # or BFT
    }
  }

  mediators {
    mediator1 {
      storage.type = memory
      admin-api.port = 5022
    }
  }
}
```

### Production Configuration (Postgres)

```hocon
canton {
  participants {
    participant1 {
      storage {
        type = postgres
        config {
          dataSourceClass = "org.postgresql.ds.PGSimpleDataSource"
          properties = {
            databaseName = "participant1"
            currentSchema = "participant1"
            serverName = "db.example.com"
            portNumber = 5432
            user = "canton"
            password = ${CANTON_DB_PASSWORD}
          }
        }
      }
      ledger-api {
        port = 5011
        tls {
          cert-chain-file = "/certs/server.crt"
          private-key-file = "/certs/server.key"
          trust-collection-file = "/certs/ca.crt"
        }
      }
      admin-api.port = 5012
    }
  }
}
```

### Multi-Synchronizer Setup

```hocon
canton {
  sequencers {
    sequencer1 { ... }
    sequencer2 { ... }  # Second synchronizer for isolation/scaling
  }
  mediators {
    mediator1 { ... }
    mediator2 { ... }
  }
  participants {
    participant1 { ... }  # Can connect to both synchronizers
  }
}
```

## Running Canton

```bash
# Start with config file
bin/canton -c my-config.conf

# Start with multiple configs (merged)
bin/canton -c base.conf -c overrides.conf

# Start with auto-connect
bin/canton -c config.conf --auto-connect-local
```

## Design Principles

1. **Privacy by design** — Sub-transaction privacy means parties only see what they need to
2. **Composability** — Participants can connect to multiple synchronizers; workflows can span them
3. **Regulatory compliance** — Right to forget (GDPR) via pruning; audit trails for compliance
4. **Horizontal scaling** — Add synchronizers for throughput; add participants for more parties
5. **Fault tolerance** — BFT sequencer tolerates Byzantine failures; mediator can be replicated

## Key Differences from Other DLTs

| Feature | Canton | Ethereum/Fabric |
|---------|--------|-----------------|
| Privacy | Sub-transaction | All-or-nothing |
| Consensus | Per-transaction | Global block |
| Composability | Cross-domain | Single chain |
| Smart contracts | DAML (typed, verified) | Solidity/Chaincode |
| Finality | Immediate | Block confirmation |
| GDPR | Supported (pruning) | Not supported |
