## ADDED Requirements

### Requirement: claim-sync shall bind StorageBackendClaim to StorageBackendContent
The claim-sync process in the storage-backend-controller shall watch StorageBackendClaim resources, create corresponding ConfigMap and Secret resources, bind the SBC to a StorageBackendContent, and update the SBC status.

#### Scenario: Sync a new StorageBackendClaim
- **WHEN** a new SBC is created with Provider, ConfigMapMeta, and SecretMeta set
- **THEN** the controller creates a StorageBackendContent with matching Provider, ConfigmapMeta, SecretMeta, and BackendClaim, sets the SBC's BoundContentName to the Content name, and updates the SBC Phase to "Bound"

#### Scenario: Sync SBC with missing ConfigMapMeta
- **WHEN** an SBC is created without ConfigMapMeta
- **THEN** the controller waits until the ConfigMapMeta is populated (by the CLI tool or external process)

#### Scenario: Sync SBC with missing SecretMeta
- **WHEN** an SBC is created without SecretMeta
- **THEN** the controller waits until the SecretMeta is populated

#### Scenario: Sync SBC update
- **WHEN** an existing SBC is updated (e.g., MaxClientThreads, Parameters, UseCert, CertSecret)
- **THEN** the controller updates the corresponding ConfigMap, Secret, and StorageBackendContent with the new values

#### Scenario: Sync SBC with provider mismatch
- **WHEN** an SBC has a Provider that doesn't match any registered provider
- **THEN** the controller sets the SBC Phase to "Unavailable"
