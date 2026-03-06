---
name: canton-local-dev
description: Local Canton development — sandbox setup, local multi-node networks, daml start workflow, and development iteration cycle.
---

# Canton Local Development

Set up and run Canton locally for development, testing, and prototyping.

## When to Activate

- Starting a new DAML project locally
- Setting up a development sandbox
- Running multi-node Canton topologies locally
- Iterating on DAML contracts with fast feedback
- Integration testing against a local ledger

## Quick Start: daml sandbox

The fastest way to get a local ledger running.

```bash
# Install DAML SDK
curl -sSL https://get.daml.com/ | sh

# Create and start a project
daml new my-project
cd my-project
daml build
daml sandbox                    # Starts Canton sandbox on port 6865
```

### daml start (All-in-One)

```bash
daml start
# Starts:
#   - Sandbox (Ledger API on port 6865)
#   - JSON API (HTTP on port 7575)
#   - Navigator (Web UI on port 4000, if enabled)
#   - Codegen (if configured)
```

### Sandbox Options

```bash
daml sandbox                          # Wall-clock time (default)
daml sandbox --static-time            # Static time (for deterministic tests)
daml sandbox --port 6866              # Custom port
daml sandbox my-app.dar               # Start with specific DAR
daml sandbox --log-level=DEBUG        # Verbose logging
```

## Full Canton Setup (Multi-Node)

For testing multi-participant scenarios, use a full Canton configuration.

### Step 1: Create Configuration

```hocon
// local-canton.conf
canton {
  participants {
    participant1 {
      storage.type = memory
      ledger-api.port = 5011
      admin-api.port = 5012
    }
    participant2 {
      storage.type = memory
      ledger-api.port = 5021
      admin-api.port = 5022
    }
  }

  sequencers {
    sequencer1 {
      storage.type = memory
      public-api.port = 5018
      admin-api.port = 5019
      sequencer.type = reference
    }
  }

  mediators {
    mediator1 {
      storage.type = memory
      admin-api.port = 5028
    }
  }
}
```

### Step 2: Start Canton

```bash
bin/canton -c local-canton.conf
```

### Step 3: Console Operations

The Canton console is a Scala REPL for node management.

```scala
// Connect participants to the synchronizer
participant1.domains.connect_local(sequencer1)
participant2.domains.connect_local(sequencer1)

// Upload DARs
participant1.dars.upload("path/to/my-app.dar")
participant2.dars.upload("path/to/my-app.dar")

// Allocate parties
val alice = participant1.parties.enable("Alice")
val bob = participant2.parties.enable("Bob")

// Check status
participant1.health.status()
participant2.health.status()

// List connected domains
participant1.domains.list_connected()
```

## Development Iteration Cycle

```
┌─────────────────┐
│ 1. Edit DAML    │
│    source files  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. daml build   │
│    (compile DAR) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. daml test    │
│    (run scripts) │
└────────┬────────┘
         │
         ▼
┌──────────────────┐
│ 4. Deploy to     │
│    sandbox        │◄──── restart or upload new DAR
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Test via      │
│    Ledger API     │
└──────────────────┘
```

### Fast Feedback Loop

```bash
# Terminal 1: Run sandbox
daml start

# Terminal 2: Edit and test
daml build && daml test --show-trace

# Terminal 3: Deploy updated DAR to running sandbox
daml ledger upload-dar .daml/dist/my-project-1.0.0.dar \
  --host localhost --port 6865
```

## Local Postgres Setup (Persistent)

For development requiring persistence across restarts:

```bash
# Start Postgres via Docker
docker run -d --name canton-db \
  -e POSTGRES_USER=canton \
  -e POSTGRES_PASSWORD=canton \
  -e POSTGRES_DB=canton \
  -p 5432:5432 \
  postgres:15

# Create databases
docker exec -i canton-db psql -U canton -c "CREATE DATABASE participant1;"
```

```hocon
// persistent-local.conf
canton {
  participants {
    participant1 {
      storage {
        type = postgres
        config {
          dataSourceClass = "org.postgresql.ds.PGSimpleDataSource"
          properties {
            databaseName = "participant1"
            serverName = "localhost"
            portNumber = 5432
            user = "canton"
            password = "canton"
          }
        }
      }
      ledger-api.port = 5011
      admin-api.port = 5012
    }
  }
}
```

## Navigator (Web UI)

```bash
# Start Navigator against a running ledger
daml navigator server localhost 6865

# Access at http://localhost:4000
# Select a party to view their contracts and submit commands
```

## JSON API Server

```bash
# Start JSON API against running ledger
daml json-api \
  --ledger-host localhost \
  --ledger-port 6865 \
  --http-port 7575 \
  --allow-insecure-tokens  # Dev mode only!

# Test with curl
curl -X POST http://localhost:7575/v1/query \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(daml auth-token --party Alice)" \
  -d '{"templateIds": ["Main:Asset"]}'
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port already in use | Kill existing process: `lsof -ti:6865 \| xargs kill` |
| DAR upload fails | Check SDK version compatibility between DAR and sandbox |
| Party not found | Allocate party first: `daml ledger allocate-parties Alice` |
| Sandbox won't start | Check JDK version (11+ required, 17 recommended) |
| Slow startup | First run downloads dependencies; subsequent runs are faster |
| Out of memory | Increase JVM heap: `JAVA_OPTS="-Xmx4g" daml sandbox` |

## Best Practices

1. **Use daml start for simple projects** — One command to run everything
2. **Use full Canton config for multi-party** — Test realistic topologies
3. **Run tests before deploying** — `daml test` catches most issues instantly
4. **Use static time for deterministic tests** — Avoid flaky time-dependent tests
5. **Restart sandbox for clean state** — In-memory mode has no reset API
6. **Keep a terminal for logs** — Watch sandbox output for errors
