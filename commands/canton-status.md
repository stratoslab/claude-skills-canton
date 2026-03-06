---
description: Check health and status of Canton nodes.
---

# Canton Status Check

1. Check if Canton/sandbox process is running
2. Test Ledger API connectivity: `grpcurl -plaintext localhost:6865 grpc.health.v1.Health/Check`
3. If Canton console is available:
   - `participant1.health.status()`
   - `participant1.domains.list_connected()`
4. Report: node health, connected domains, loaded packages, allocated parties
