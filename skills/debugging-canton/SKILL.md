---
name: debugging-canton
description: Debugging Canton nodes and DAML contracts — error codes, log analysis, transaction inspection, common issues and their solutions.
---

# Debugging Canton & DAML

Systematic approaches to diagnosing and resolving issues in Canton deployments and DAML contracts.

## When to Activate

- Transaction submission failures
- Contract not found errors
- Authorization rejections
- Node connectivity issues
- Performance problems

## Error Code System

Canton uses structured error codes: `<CATEGORY>_<SPECIFIC>(<GROUP>)`.

### Common Error Codes

| Code | Meaning | Solution |
|------|---------|----------|
| `INVALID_ARGUMENT` | Malformed request | Fix command payload |
| `NOT_FOUND` | Contract archived or missing | Check if contract still active |
| `ALREADY_EXISTS` | Duplicate command | Safe to ignore (dedup) |
| `FAILED_PRECONDITION` | Ensure clause failed | Check template invariants |
| `PERMISSION_DENIED` | Auth failure | Verify JWT token |
| `ABORTED` | Contention | Retry with backoff |
| `INCONSISTENT` | Stale contract ID | Re-fetch the contract |
| `CONTRACT_NOT_ACTIVE` | Exercising archived contract | Check for concurrent archive |
| `PARTY_NOT_KNOWN` | Party not allocated | Allocate party first |
| `PACKAGE_NOT_FOUND` | DAR not uploaded | Upload DAR to participant |

## Log Analysis

### Canton Log Levels

```hocon
// In canton config or logback.xml
canton {
  monitoring {
    log-level = DEBUG    // TRACE, DEBUG, INFO, WARN, ERROR
  }
}
```

### Key Log Patterns

```bash
# Find transaction failures
grep "REJECTED" canton.log

# Find authorization errors
grep "PERMISSION_DENIED\|NOT_AUTHORIZED" canton.log

# Find connectivity issues
grep "UNAVAILABLE\|connection" canton.log

# Find package errors
grep "PACKAGE_NOT_FOUND" canton.log

# Using lnav (Canton has custom format files)
lnav -I canton.lnav.json canton.log
```

### Useful Log Fields

```
timestamp | level | logger | message | correlationId | commandId
```

- **correlationId** — Links related log entries across components
- **commandId** — Matches logs to specific command submissions

## Transaction Inspection

### DAML Script

```daml
-- Use submitTree to see full transaction structure
testDebug : Script ()
testDebug = script do
  alice <- allocateParty "Alice"
  bank <- allocateParty "Bank"

  assetCid <- submit bank do
    createCmd Asset with issuer = bank; owner = alice; quantity = 100.0

  -- Get full transaction tree
  tree <- submitTree alice do
    exerciseCmd assetCid Transfer with newOwner = alice

  debug (show tree)  -- Print transaction tree
```

### Ledger API

```java
// Get transaction tree by ID
var tree = client.getTransactionsClient()
    .getTransactionById(transactionId, Set.of(partyId));

// Walk the tree
tree.getEventsById().forEach((eventId, event) -> {
    if (event.hasCreated()) {
        System.out.println("Created: " + event.getCreated().getContractId());
    } else if (event.hasExercised()) {
        System.out.println("Exercised: " + event.getExercised().getChoice());
    }
});
```

### Canton Console

```scala
// Inspect active contracts for a party
participant1.ledger_api.acs.of_party(alice)

// Check domain connectivity
participant1.domains.list_connected()

// Check health
participant1.health.status()
```

## Common Issues & Solutions

### Issue: "Contract not found"

```
Possible causes:
1. Contract was archived by another transaction
2. Contract ID is stale (from a previous version)
3. Party doesn't have visibility to the contract
4. DAR not uploaded to the validating participant

Debug steps:
1. Query ACS for the template — is the contract still active?
2. Check if another transaction archived it
3. Verify the querying party is a stakeholder
4. Verify DAR is uploaded to all relevant participants
```

### Issue: "Authorization failed"

```
Possible causes:
1. Submitting party is not the controller
2. Missing signatory authority for create
3. JWT token doesn't include the required party

Debug steps:
1. Check who the controller is for the choice
2. Verify the submitting party matches the controller
3. Check JWT token claims (actAs, readAs)
4. For multi-party: verify propose-accept flow is used
```

### Issue: "Timeout / DEADLINE_EXCEEDED"

```
Possible causes:
1. Sequencer overloaded
2. Network latency between participant and domain
3. Large transaction (many contracts)
4. Database slow

Debug steps:
1. Check sequencer health and metrics
2. Check network latency
3. Reduce transaction size (fewer commands per submission)
4. Check database connection pool and query performance
```

### Issue: "Contention / ABORTED"

```
Possible causes:
1. Multiple transactions trying to exercise the same contract
2. Contract key conflicts
3. Hot contracts (singleton patterns)

Solutions:
1. Implement retry with exponential backoff
2. Reduce contention by partitioning state
3. Use non-consuming choices where possible
4. Batch operations within a single transaction
```

### Issue: Node Won't Connect to Domain

```scala
// Debug connectivity
participant1.domains.connect("my-domain", "https://sequencer:5018")
// Check error message carefully

// Common fixes:
// 1. Verify sequencer is running and healthy
sequencer1.health.status()

// 2. Check TLS configuration matches
// 3. Verify network connectivity (firewall, DNS)
// 4. Check that participant's identity is registered on the domain
```

## Performance Debugging

### Key Metrics

```bash
# Prometheus metrics endpoint
curl http://localhost:9000/metrics | grep canton

# Key metrics to check:
# canton_participant_commands_total
# canton_participant_commands_failed_total
# canton_sequencer_send_latency
# canton_mediator_confirmation_time
```

### ACS Size Monitoring

```scala
// Large ACS = slow queries + high memory
val acsSize = participant1.ledger_api.acs.of_party(alice).size
if (acsSize > 10000) {
  println(s"WARNING: Large ACS ($acsSize contracts)")
  println("Consider archiving old contracts or pruning")
}
```

## Best Practices

1. **Always include commandId** — Links logs to submissions
2. **Use structured logging** — JSON format for log aggregation
3. **Monitor error rates** — Alert on spikes in failed commands
4. **Test failure scenarios** — Deliberately cause errors to verify handling
5. **Use debug in DAML Script** — Print intermediate values
6. **Keep DARs in sync** — Same DAR on all participants
7. **Check health first** — Before debugging complex issues
