---
name: dar-management
description: DAR build, deploy, version, and upgrade management — compiling DAML to DAR, uploading to participants, package versioning, and upgrade strategies.
---

# DAR Management

A DAR (DAML Archive) is a compiled DAML package containing templates, interfaces, and data types. Managing DARs is core to the Canton deployment lifecycle.

## When to Activate

- Building and compiling DAML projects
- Deploying DARs to Canton participants
- Managing package versions
- Planning contract upgrades
- Troubleshooting package-related errors

## Build

```bash
# Compile DAML to DAR
daml build
# Output: .daml/dist/<name>-<version>.dar

# Clean and rebuild
daml clean && daml build

# Build with specific options
daml build --target=2.1 --output=my-app.dar
```

## DAR Contents

A DAR is a ZIP file containing:
```
my-app-1.0.0.dar
├── META-INF/
│   └── MANIFEST.MF
├── <package-id>.dalf        # Compiled DAML-LF (main package)
├── <dep-package-id>.dalf    # Transitive dependencies
└── daml.yaml                # Package metadata
```

## Deploy

### To Sandbox
```bash
# Upload to running sandbox
daml ledger upload-dar .daml/dist/my-app-1.0.0.dar \
  --host localhost --port 6865

# With authentication
daml ledger upload-dar my-app.dar \
  --host localhost --port 6865 \
  --access-token-file admin-token.jwt
```

### Via Canton Console
```scala
// Upload to a specific participant
participant1.dars.upload("path/to/my-app-1.0.0.dar")

// List uploaded packages
participant1.packages.list()

// Upload to multiple participants
Seq(participant1, participant2).foreach(_.dars.upload("my-app.dar"))
```

### Via gRPC (Programmatic)
```java
var darBytes = Files.readAllBytes(Path.of("my-app.dar"));
client.getPackageManagementClient()
    .uploadDarFile(ByteString.copyFrom(darBytes))
    .blockingGet();
```

## Package Versioning

### daml.yaml Version
```yaml
name: my-app
version: 1.0.0  # Semantic versioning
```

### Package ID
Every compiled package gets a unique **package ID** (hash of contents). Even a whitespace change produces a different package ID.

```bash
# List packages and their IDs
daml damlc inspect-dar my-app.dar

# The package ID is used in template identifiers:
# <package-id>:Module:TemplateName
```

## Upgrade Strategies

### Strategy 1: Side-by-Side (New Template)

Deploy V2 alongside V1. Migrate contracts individually.

```daml
-- V1 (already deployed)
template AssetV1 with ...

-- V2 (new package)
template AssetV2 with
    issuer : Party
    owner : Party
    quantity : Decimal
    currency : Text  -- New field
  where ...
```

Migration choice on V1:
```daml
choice MigrateToV2 : ContractId AssetV2
  controller issuer
  do
    create AssetV2 with
      issuer; owner; quantity
      currency = "USD"  -- Default
```

### Strategy 2: Interface-Based Upgrade

Use interfaces for stable APIs across versions.

```daml
-- Stable interface (never changes)
interface IAsset where
  viewtype AssetView
  ...

-- V1 implements IAsset
template AssetV1 with ...
  where
    interface instance IAsset for AssetV1 where ...

-- V2 also implements IAsset
template AssetV2 with ...
  where
    interface instance IAsset for AssetV2 where ...

-- Applications interact via IAsset — agnostic to version
```

### Strategy 3: Daml Upgrade Mechanism

DAML 2.x supports package upgrades with compatibility checks:

```yaml
# daml.yaml for V2
upgrades: ./v1/my-app-1.0.0.dar
```

The compiler checks that V2 is a compatible extension of V1.

## Multi-Participant Deployment

All participants that need to validate contracts must have the DAR:

```bash
# Deploy to all participants
for host in participant1.example.com participant2.example.com; do
  daml ledger upload-dar my-app.dar --host $host --port 5011
done
```

```scala
// Canton console — upload to all connected participants
participants.all.foreach(_.dars.upload("my-app.dar"))
```

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `PACKAGE_NOT_FOUND` | DAR not uploaded to participant | Upload DAR to all relevant participants |
| `DUPLICATE_PACKAGE` | Same DAR already uploaded | Safe to ignore |
| `INVALID_PACKAGE` | Corrupt or incompatible DAR | Rebuild with matching SDK version |
| `MISSING_DEPENDENCY` | DAR depends on unuploaded package | Upload dependency DARs first |

## Best Practices

1. **Version semantically** — Major for breaking changes, minor for additions, patch for fixes
2. **Upload to ALL participants** — Every participant validating contracts needs the DAR
3. **Use interfaces for stable APIs** — Decouple consumers from template versions
4. **Test upgrades in sandbox first** — Verify migration paths before production
5. **Track package IDs** — Log which package ID is deployed where
6. **Automate DAR deployment** — Script the upload process for consistency
