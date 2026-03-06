---
description: Query active contracts on the ledger via JSON API or CLI.
---

# Query Active Contracts

1. Identify the target party and template
2. Query via JSON API:
   ```bash
   curl -X POST http://localhost:7575/v1/query \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"templateIds": ["<Module>:<Template>"]}'
   ```
3. Or via DAML Script:
   ```daml
   contracts <- query @Template party
   ```
4. Display results: contract IDs, key fields, counts
