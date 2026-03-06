---
description: Deploy a DAR to a running Canton participant or sandbox.
---

# Deploy DAR to Canton

1. Verify the DAR exists (`.daml/dist/` directory)
2. Identify the target ledger (host, port)
3. Run `daml ledger upload-dar <dar-path> --host <host> --port <port>`
4. If auth is required, add `--access-token-file <token-file>`
5. Verify upload success
6. For multi-participant: upload to ALL participants that need the package
