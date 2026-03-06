---
description: Always-follow standards for Canton configuration files.
globs: ["**/*.conf", "**/canton*.hocon"]
---

# Canton Configuration Standards

## General
- Use HOCON format (.conf files)
- Use environment variable substitution for secrets: `${CANTON_DB_PASSWORD}`
- Never hardcode credentials in config files
- Comment complex or non-obvious settings

## Storage
- Use `memory` for development/testing only
- Use `postgres` for staging and production
- Separate databases for each node (participant, sequencer, mediator)
- Set appropriate `max-connections` for the workload

## Security
- Enable TLS on all APIs in production
- Bind Admin API to `127.0.0.1` (never `0.0.0.0`)
- Configure JWT authentication for Ledger API
- Use certificate files from a secure path

## Monitoring
- Enable Prometheus metrics in production
- Configure health check endpoints
- Set appropriate log levels (INFO for production, DEBUG for troubleshooting)

## Naming
- Use descriptive node names: `participant-acme`, not `participant1`
- Use consistent port numbering scheme
- Document the topology in comments
