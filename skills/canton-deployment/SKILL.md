---
name: canton-deployment
description: Canton production deployment — Docker, Kubernetes, Postgres storage, TLS configuration, JWT authentication, monitoring, and high availability.
---

# Canton Production Deployment

Deploying Canton for production with proper storage, security, monitoring, and high availability.

## When to Activate

- Deploying Canton to staging or production
- Configuring TLS and authentication
- Setting up Postgres for persistent storage
- Planning high-availability topology
- Configuring monitoring and alerting

## Docker Deployment

### Docker Compose Example

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: canton
      POSTGRES_PASSWORD: ${CANTON_DB_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

  sequencer:
    image: digitalasset/canton-open-source:latest
    command: daemon --config /canton/config.conf
    volumes:
      - ./config/sequencer.conf:/canton/config.conf
      - ./certs:/canton/certs
    ports:
      - "5018:5018"  # Public API
      - "5019:5019"  # Admin API
    depends_on:
      - postgres

  mediator:
    image: digitalasset/canton-open-source:latest
    command: daemon --config /canton/config.conf
    volumes:
      - ./config/mediator.conf:/canton/config.conf
      - ./certs:/canton/certs
    ports:
      - "5028:5028"  # Admin API
    depends_on:
      - postgres
      - sequencer

  participant:
    image: digitalasset/canton-open-source:latest
    command: daemon --config /canton/config.conf
    volumes:
      - ./config/participant.conf:/canton/config.conf
      - ./certs:/canton/certs
    ports:
      - "5011:5011"  # Ledger API
      - "5012:5012"  # Admin API
      - "5013:5013"  # HTTP Ledger API
    depends_on:
      - postgres
      - sequencer

volumes:
  pg_data:
```

### Database Initialization

```sql
-- init-db.sql
CREATE DATABASE sequencer1;
CREATE DATABASE mediator1;
CREATE DATABASE participant1;
```

## Production Configuration

### Participant with TLS + JWT Auth

```hocon
canton {
  participants {
    participant1 {
      storage {
        type = postgres
        config {
          dataSourceClass = "org.postgresql.ds.PGSimpleDataSource"
          properties {
            databaseName = "participant1"
            currentSchema = "public"
            serverName = ${CANTON_DB_HOST}
            portNumber = 5432
            user = ${CANTON_DB_USER}
            password = ${CANTON_DB_PASSWORD}
          }
        }
        max-connections = 16
      }

      ledger-api {
        port = 5011
        address = "0.0.0.0"
        tls {
          cert-chain-file = "/certs/ledger-api.crt"
          private-key-file = "/certs/ledger-api.key"
          trust-collection-file = "/certs/ca.crt"
        }
        auth-services = [{
          type = jwt-rs-256-jwks
          url = "https://auth.example.com/.well-known/jwks.json"
        }]
      }

      admin-api {
        port = 5012
        address = "127.0.0.1"  # Admin API should not be exposed
      }

      http-ledger-api {
        port = 5013
        address = "0.0.0.0"
      }

      parameters {
        initial-protocol-version = 5
      }
    }
  }
}
```

### JWT Token Format

```json
{
  "https://daml.com/ledger-api": {
    "ledgerId": "my-ledger",
    "applicationId": "my-app",
    "actAs": ["Alice::1234abcd"],
    "readAs": ["Alice::1234abcd", "Public::5678efgh"]
  },
  "exp": 1700000000,
  "iss": "https://auth.example.com"
}
```

## Monitoring

### Prometheus Metrics

```hocon
canton {
  monitoring {
    metrics {
      reporters = [{
        type = prometheus
        address = "0.0.0.0"
        port = 9000
      }]
    }
  }
}
```

### Key Metrics to Monitor

| Metric | Description |
|--------|-------------|
| `canton_sequencer_processed_total` | Messages processed by sequencer |
| `canton_participant_commands_total` | Commands submitted |
| `canton_participant_commands_failed_total` | Failed commands |
| `canton_mediator_confirmations_total` | Transaction confirmations |
| `canton_participant_acs_size` | Active contract set size |
| `canton_participant_ledger_api_latency` | API response latency |

### Health Checks

```hocon
canton {
  monitoring {
    health {
      server {
        port = 8080
        address = "0.0.0.0"
      }
    }
  }
}
```

```bash
# Health check endpoint
curl http://localhost:8080/health

# gRPC health (via grpcurl)
grpcurl -plaintext localhost:5011 grpc.health.v1.Health/Check
```

## High Availability

### Active-Passive Participant

```hocon
canton {
  participants {
    participant1 {
      replication {
        enabled = true
      }
      storage {
        type = postgres
        config { ... }  # Shared database
      }
    }
  }
}
```

### Multi-Sequencer (BFT)

```hocon
canton {
  sequencers {
    sequencer1 {
      storage.type = postgres
      sequencer.type = BFT
      public-api.port = 5001
    }
    sequencer2 {
      storage.type = postgres
      sequencer.type = BFT
      public-api.port = 5002
    }
    sequencer3 {
      storage.type = postgres
      sequencer.type = BFT
      public-api.port = 5003
    }
    sequencer4 {
      storage.type = postgres
      sequencer.type = BFT
      public-api.port = 5004
    }
  }
}
```

## Operational Runbook

### Deploy a New DAR

```bash
# Via CLI
daml ledger upload-dar my-app.dar \
  --host participant.example.com \
  --port 5011 \
  --tls \
  --access-token-file token.jwt

# Via Canton console
participant1.dars.upload("/path/to/my-app.dar")
```

### Allocate a Party

```bash
daml ledger allocate-parties "NewParty" \
  --host participant.example.com \
  --port 5011 \
  --access-token-file admin-token.jwt
```

### Pruning (GDPR)

```scala
// Canton console
participant1.pruning.prune(offset)
```

### Backup

```bash
# Database backup (Postgres)
pg_dump -h localhost -U canton participant1 > backup_participant1.sql

# All databases
pg_dumpall -h localhost -U canton > backup_all.sql
```

## Security Checklist

- [ ] TLS enabled on all APIs (Ledger, Admin, Public)
- [ ] Admin API bound to localhost or internal network only
- [ ] JWT authentication configured for Ledger API
- [ ] Database credentials in environment variables, not config files
- [ ] Network policies restrict inter-node communication
- [ ] Prometheus metrics endpoint not publicly accessible
- [ ] Regular database backups configured
- [ ] Log rotation configured
- [ ] Rate limiting on Ledger API
- [ ] Certificate rotation plan in place

## Best Practices

1. **Always use Postgres in production** — Memory storage loses data on restart
2. **Separate databases per node** — Each participant, sequencer, mediator gets its own DB
3. **Never expose Admin API publicly** — Bind to localhost or use network policies
4. **Use environment variables for secrets** — HOCON supports `${ENV_VAR}` substitution
5. **Monitor ACS size** — Growing ACS indicates contracts not being archived
6. **Plan for pruning** — Schedule regular pruning for GDPR compliance
7. **Test failover** — Regularly test HA failover procedures
