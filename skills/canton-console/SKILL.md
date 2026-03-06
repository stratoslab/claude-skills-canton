---
name: canton-console
description: Canton console operations — Scala REPL commands for node management, domain connectivity, DAR uploads, party management, and health checks.
---

# Canton Console

The Canton console is an interactive Scala REPL for managing Canton nodes. It provides direct access to all participant, sequencer, and mediator operations.

## When to Activate

- Managing Canton nodes interactively
- Performing administrative operations
- Debugging connectivity or health issues
- Scripting Canton operations

## Starting the Console

```bash
# Start Canton with console
bin/canton -c my-config.conf

# Canton opens an interactive Scala REPL
@ // ready for commands
```

## Node References

Nodes are referenced by their config names:

```scala
// From config: canton.participants.participant1 { ... }
participant1           // Reference to participant node
participant1.id        // Participant's unique identifier

// From config: canton.sequencers.sequencer1 { ... }
sequencer1             // Reference to sequencer node

// From config: canton.mediators.mediator1 { ... }
mediator1              // Reference to mediator node
```

## Health & Status

```scala
// Check node health
participant1.health.status()
sequencer1.health.status()
mediator1.health.status()

// Check all nodes
nodes.local.map(_.health.status())

// Wait for node to be running
participant1.health.wait_for_running()
```

## Domain Connectivity

```scala
// Connect participant to local sequencer
participant1.domains.connect_local(sequencer1)

// Connect to remote domain
participant1.domains.connect("my-domain", "https://sequencer.example.com:5018")

// List connected domains
participant1.domains.list_connected()

// Disconnect from domain
participant1.domains.disconnect("my-domain")

// Reconnect
participant1.domains.reconnect("my-domain")

// Check domain parameters
sequencer1.domain.parameters()
```

## DAR Management

```scala
// Upload DAR
participant1.dars.upload("path/to/my-app.dar")

// Upload to multiple participants
Seq(participant1, participant2).foreach(_.dars.upload("my-app.dar"))

// List uploaded packages
participant1.packages.list()

// List DARs (with metadata)
participant1.dars.list()
```

## Party Management

```scala
// Allocate a new party
val alice = participant1.parties.enable("Alice")
val bob = participant2.parties.enable("Bob")

// With specific party ID hint
val trader = participant1.parties.enable("Trader", PartyIdHint("acme-trader"))

// List parties on a participant
participant1.parties.list()

// Check party-to-participant mapping
participant1.topology.party_to_participant_mappings.list()
```

## Topology Management

```scala
// List all topology transactions
participant1.topology.transactions.list()

// Party to participant mappings
participant1.topology.party_to_participant_mappings.list()

// Namespace delegations
participant1.topology.namespace_delegations.list()

// Owner to key mappings
participant1.topology.owner_to_key_mappings.list()
```

## Ledger API Operations (from Console)

```scala
// Run DAML Script from console
participant1.ledger_api.daml_script.run("path/to/script.dar", "Module:functionName")

// Get active contracts (simplified)
participant1.ledger_api.acs.of_party(alice)

// Filter by template
participant1.ledger_api.acs.of_party(alice).filter(_.templateId.contains("Asset"))
```

## Useful Scripting Patterns

### Bootstrap a Local Network
```scala
// Connect all participants to the domain
participant1.domains.connect_local(sequencer1)
participant2.domains.connect_local(sequencer1)

// Upload DARs everywhere
val darPath = "my-app.dar"
participant1.dars.upload(darPath)
participant2.dars.upload(darPath)

// Allocate parties
val alice = participant1.parties.enable("Alice")
val bob = participant2.parties.enable("Bob")
val bank = participant1.parties.enable("Bank")

println(s"Alice: $alice")
println(s"Bob: $bob")
println(s"Bank: $bank")
```

### Health Check Script
```scala
// Check all nodes are healthy
val allHealthy = nodes.local.forall(node =>
  node.health.status().isSuccess
)
if (!allHealthy) {
  println("WARNING: Not all nodes are healthy!")
  nodes.local.foreach(node =>
    println(s"${node.name}: ${node.health.status()}")
  )
}
```

## Console Scripting (Non-Interactive)

```bash
# Run a bootstrap script
bin/canton -c config.conf --bootstrap bootstrap.canton

# bootstrap.canton contains console commands:
# participant1.domains.connect_local(sequencer1)
# participant1.dars.upload("my-app.dar")
# ...
```

## Key Console Objects

| Object | Description |
|--------|-------------|
| `participant1` | Participant node reference |
| `participant1.health` | Health checks |
| `participant1.domains` | Domain connectivity |
| `participant1.dars` | DAR management |
| `participant1.packages` | Package management |
| `participant1.parties` | Party management |
| `participant1.topology` | Topology state |
| `participant1.ledger_api` | Ledger API access |
| `participant1.pruning` | Data pruning |
| `nodes.local` | All local nodes |

## Best Practices

1. **Use bootstrap scripts for repeatable setup** — `--bootstrap script.canton`
2. **Check health before operations** — Verify nodes are running
3. **Upload DARs to all participants** — Missing packages cause validation failures
4. **Use `nodes.local` for bulk operations** — Iterate over all nodes
5. **Save party IDs** — Log the full party ID (with namespace) after allocation
