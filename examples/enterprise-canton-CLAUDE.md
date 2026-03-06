# Enterprise Workflow on Canton

## Project Overview
Multi-organization enterprise workflow on Canton Network with privacy requirements.

## Tech Stack
- DAML (smart contracts)
- Canton (multi-participant, multi-synchronizer)
- Java (backend services)
- Postgres + Docker + Kubernetes

## Skills Reference
Reference: ~/claude-skills-canton/skills/

## Key Skills
- canton-architecture — Multi-participant topology design
- canton-deployment — Production deployment with HA
- multi-party-workflows — Cross-organization authorization
- canton-privacy — Data isolation between organizations
- dar-management — Package versioning and upgrades
- party-management — Per-organization party allocation
- debugging-canton — Production troubleshooting

## Architecture
- Each organization runs their own participant node
- Shared synchronizer(s) for cross-org transactions
- Private synchronizer for org-internal workflows
- BFT sequencer for shared synchronizers

## Deployment
- Kubernetes with Helm charts
- Postgres per node (separate databases)
- TLS everywhere, JWT auth on Ledger API
- Prometheus + Grafana for monitoring

## Privacy Model
- Organization A cannot see Organization B's internal contracts
- Shared contracts visible only to involved organizations
- Regulatory reporting via observer patterns
- GDPR compliance via Canton pruning
