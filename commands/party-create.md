---
description: Allocate a new party on a Canton participant.
---

# Create Party

1. Get party name from user
2. Run `daml ledger allocate-parties <name> --host <host> --port <port>`
3. If auth required, add `--access-token-file <token-file>`
4. Report the full party ID (with namespace)
5. Optionally update application configuration with the new party ID
