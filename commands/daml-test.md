---
description: Run DAML Script tests and report results.
---

# Run DAML Tests

1. Run `daml test --show-trace`
2. Parse the output for pass/fail counts
3. If tests fail:
   - Show the failing test name and error message
   - Suggest fixes based on the error type
   - Common issues: authorization failures, contract not found, assertion failures
4. Report summary: X passed, Y failed
