---
name: party-management
description: Party allocation and management — party IDs, display names, party-to-participant mapping, and multi-participant party hosting.
---

# Party Management

Parties are the identity primitive in DAML. They represent entities (people, organizations, roles) that can own contracts and authorize actions.

## When to Activate

- Allocating parties on Canton participants
- Mapping parties to participants
- Designing identity models for applications
- Configuring multi-participant party hosting

## Party Allocation

### CLI
```bash
# Allocate parties
daml ledger allocate-parties Alice Bob Bank \
  --host localhost --port 6865

# List known parties
daml ledger list-parties --host localhost --port 6865
```

### Canton Console
```scala
// Allocate on a specific participant
val alice = participant1.parties.enable("Alice")
// Result: Alice::1234abcdef (party ID with namespace)

// With specific party ID hint
val bob = participant1.parties.enable("Bob", PartyIdHint("bob-trading"))

// List parties on participant
participant1.parties.list()

// Check party-to-participant mapping
participant1.topology.party_to_participant_mappings.list()
```

### DAML Script
```daml
-- Simple allocation
alice <- allocateParty "Alice"

-- With hint
bob <- allocatePartyWithHint "Bob" (PartyIdHint "bob-1")

-- On specific participant
charlie <- allocatePartyOn "Charlie" (ParticipantName "participant2")
```

### Ledger API (gRPC)
```java
var response = client.getPartyManagementClient()
    .allocateParty(
        Optional.of("alice-hint"),     // Party ID hint
        Optional.of("Alice Trading"))  // Display name
    .blockingGet();

String partyId = response.getPartyDetails().getParty();
// "alice-hint::1234abcdef"
```

## Party Identity Format

```
alice-trading::1234abcdef5678
└─── hint ────┘└── namespace ─┘
```

- **Hint** — Human-readable prefix (optional, suggested by client)
- **Namespace** — Cryptographic fingerprint of the participant's key
- **Full party ID** — `hint::namespace` (globally unique)

## Party-to-Participant Mapping

A party can be hosted on one or more participants.

### Single Hosting (Typical)
```
Alice::ns1  →  Participant 1
Bob::ns2    →  Participant 2
```

### Multi-Hosting (Advanced)
```
Alice::ns1  →  Participant 1  (active)
Alice::ns1  →  Participant 3  (active)  ← Same party, two participants
```

Multi-hosting allows a party to submit commands from multiple participants. Used for high availability.

### Configure in Canton Console
```scala
// Add Alice to participant2 (Alice already exists on participant1)
participant2.topology.party_to_participant_mappings.authorize(
  TopologyChangeOp.Add,
  alice,
  participant2.id,
  ParticipantPermission.Submission
)
```

## Participant Permissions

| Permission | Can Read | Can Submit |
|-----------|----------|------------|
| `Observation` | Yes | No |
| `Confirmation` | Yes | Confirm only |
| `Submission` | Yes | Yes |

## Party Design Patterns

### Per-User Parties
```
alice-user::ns1      — Alice's personal identity
bob-user::ns2        — Bob's personal identity
```

### Role-Based Parties
```
acme-admin::ns1      — Admin role for Acme Corp
acme-trader::ns1     — Trader role
acme-compliance::ns1 — Compliance role
```

### Service Parties
```
exchange-operator::ns1  — Exchange operator party
public-reader::ns1      — Public read-only party
```

### Public Party Pattern
```daml
-- A "public" party that everyone can read as
-- Used for publishing public reference data
template PublicReference
  with
    issuer : Party
    publicParty : Party  -- Everyone has readAs access to this party
    data : Text
  where
    signatory issuer
    observer publicParty
```

## Best Practices

1. **Use meaningful party ID hints** — `acme-trading` not `party1`
2. **Separate user parties from service parties** — Different lifecycle
3. **Document party roles** — Who is signatory, observer, controller
4. **Use a public party for shared data** — Simpler than adding individual observers
5. **Plan for party migration** — Parties can move between participants
6. **Limit multi-hosting** — Only when HA is required; adds complexity
