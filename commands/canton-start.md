---
description: Start a local Canton sandbox or multi-node network for development.
---

# Start Local Canton

## Sandbox (Simple)
1. Run `daml sandbox` (or `daml start` for sandbox + JSON API)
2. Verify Ledger API is accessible on port 6865
3. Report available endpoints

## Full Canton (Multi-Node)
1. Check for a Canton config file (`.conf`)
2. Start with `bin/canton -c config.conf`
3. Verify all nodes are healthy
4. Connect participants to domains
5. Upload DARs to all participants
6. Allocate initial parties
