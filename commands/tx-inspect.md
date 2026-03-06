---
description: Inspect a transaction tree for debugging — show events, causality, and party visibility.
---

# Inspect Transaction

1. Get the transaction ID from the user
2. Fetch the transaction tree via Ledger API or DAML Script:
   ```daml
   tree <- submitTree party do exerciseCmd cid ChoiceName with ..
   debug (show tree)
   ```
3. Display:
   - Root events (exercises)
   - Child events (creates, archives) with indentation
   - For each event: template, choice, parties, contract IDs
   - Which parties can see which events (projection)
4. Highlight any rollback nodes
5. Summarize: N creates, M archives, K exercises
