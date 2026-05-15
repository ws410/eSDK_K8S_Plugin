## ADDED Requirements

### Requirement: StorageBackendClaim CRD shall define user-facing backend requests
The StorageBackendClaim (SBC) is a namespaced Custom Resource that represents a user's request to configure a storage backend. It follows the PVC-PV pattern where the Claim is the user-facing request and the Content is the cluster-scoped actual configuration.

#### Scenario: SBC Spec fields
- **WHEN** a user creates a StorageBackendClaim
- **THEN** the Spec must include: Provider (required, filters the provider), and may optionally include: ConfigMapMeta (namespace/name format for storage management info), SecretMeta (namespace/name format for sensitive info), MaxClientThreads (limits storage client connections), Parameters (user-defined extension parameters), UseCert (boolean, default false, enables certificate usage), CertSecret (name of the certificate Secret)

#### Scenario: SBC Status fields
- **WHEN** the SBC is processed by the controller
- **THEN** the Status is populated with: Phase (Pending/Bound/Unavailable), StorageBackendId (unique backend identifier), ConfigmapMeta (current configmap namespace/name), SecretMeta (current secret namespace/name), MaxClientThreads (current value), BoundContentName (reference to the bound StorageBackendContent), StorageType (e.g., oceanstor-san), Protocol (e.g., iscsi, nfs), MetroBackend (hyperMetro partner backend), UseCert, CertSecret

#### Scenario: SBC lifecycle phases
- **WHEN** a new SBC is created
- **THEN** its Phase is set to "Pending"
- **WHEN** the controller binds it to a StorageBackendContent
- **THEN** its Phase is set to "Bound"
- **WHEN** the backend fails to log in (e.g., wrong password)
- **THEN** its Phase is set to "Unavailable"

#### Scenario: SBC printed columns
- **WHEN** a user runs `kubectl get sbc`
- **THEN** the output includes columns: StorageBackendContentName, Status, Age (and with -o wide: StorageType, Protocol, MetroBackend)

#### Scenario: SBC short name
- **WHEN** a user uses the short name
- **THEN** `sbc` is accepted as an alias for StorageBackendClaim

#### Scenario: SBC immutable Provider field
- **WHEN** a user attempts to update an existing SBC's Provider field
- **THEN** the update is rejected by the admission webhook (Provider is immutable after creation)

#### Scenario: SBC with UseCert enabled
- **WHEN** a user creates an SBC with UseCert=true
- **THEN** the CertSecret field must be populated with the certificate Secret name; the controller will use the certificate for backend authentication
