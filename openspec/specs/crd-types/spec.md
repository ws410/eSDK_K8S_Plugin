## ADDED Requirements

### Requirement: CRD types shall define custom resources for backend and volume management
The system defines four Custom Resource Definitions: StorageBackendClaim (SBC), StorageBackendContent (SBCT), VolumeModifyClaim (VMC), and VolumeModifyContent. SBC/SBCT follow the PVC-PV pattern for backend management; VMC/VMContent follow a job-like pattern for bulk volume modification.

---

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

---

### Requirement: StorageBackendContent CRD shall represent actual backend configuration
The StorageBackendContent (SBCT) is a cluster-scoped Custom Resource that represents the actual storage backend configuration. It is bound to a StorageBackendClaim and contains pool information, capacity, capabilities, and online status.

#### Scenario: SBCT Spec fields
- **WHEN** a StorageBackendContent is created by the controller
- **THEN** the Spec includes: Provider (required, matches the SBC), ConfigmapMeta (current configmap namespace/name), SecretMeta (current secret namespace/name), BackendClaim (bound SBC namespace/name), MaxClientThreads, Parameters (extension parameters), UseCert, CertSecret

#### Scenario: SBCT Status fields
- **WHEN** the sidecar controller updates the SBCT status
- **THEN** the Status includes: ContentName (identity: provider-name@backend-name#pool-name), VendorName (e.g., Huawei), ProviderVersion (CSI driver version), Pools (array of pool names with capacities), Capacity (TotalCapacity/UsedCapacity/FreeCapacity map), Capabilities (map of capability names to booleans: SupportThin, SupportThick, SupportQoS, SupportMetro, SupportReplication, SupportClone, etc.), Specification (device SN, VStoreID, etc.), ConfigmapMeta, SecretMeta, Online (login success flag), MaxClientThreads, SN (storage device serial number), UseCert, CertSecret

#### Scenario: SBCT printed columns
- **WHEN** a user runs `kubectl get sbct`
- **THEN** the output includes columns: Claim, SN, VendorName, ProviderVersion, Online, Age

#### Scenario: SBCT short name
- **WHEN** a user uses the short name
- **THEN** `sbct` is accepted as an alias for StorageBackendContent

#### Scenario: SBCT is cluster-scoped
- **WHEN** a user queries SBCT
- **THEN** it is accessible without namespace specification (cluster-scoped resource)

---

### Requirement: VolumeModifyClaim CRD shall define volume modification requests
The VolumeModifyClaim (VMC) is a cluster-scoped Custom Resource that represents a request to modify volumes (e.g., QoS, SmartTier, SmartMigration). It follows a job-like pattern where the Claim tracks overall progress and Contents track individual volume modifications.

#### Scenario: VMC Spec fields
- **WHEN** a user creates a VolumeModifyClaim
- **THEN** the Spec must include: Source (required, with Kind - default "StorageClass", Name, and optional Namespace), and Parameters (CSI driver-specific opaque key-value pairs for modification settings)

#### Scenario: VMC Status fields
- **WHEN** the VMC is processed by the controller
- **THEN** the Status includes: Phase (Pending/Creating/Completed/Rollback/Deleting), Contents (array of ModifyContents with ModifyContentName, SourceVolume, and Status), Ready (progress indicator, e.g., "2/5"), Parameters (echo of spec parameters), StartedAt (timestamp), CompletedAt (timestamp)

#### Scenario: VMC lifecycle phases
- **WHEN** a new VMC is created
- **THEN** its Phase is set to "Pending"
- **WHEN** VContents are being created but not all completed
- **THEN** its Phase is set to "Creating"
- **WHEN** all associated VContents are completed
- **THEN** its Phase is set to "Completed"
- **WHEN** the VMC receives a deletion request and starts rollback
- **THEN** its Phase is set to "Rollback"
- **WHEN** the VMC starts deleting
- **THEN** its Phase is set to "Deleting"

#### Scenario: VMC printed columns
- **WHEN** a user runs `kubectl get vmc`
- **THEN** the output includes columns: Status, Ready, Age (and with -o wide: SourceKind, SourceName, StartedAt, CompletedAt)

#### Scenario: VMC short name
- **WHEN** a user uses the short name
- **THEN** `vmc` is accepted as an alias for VolumeModifyClaim

#### Scenario: VMC is cluster-scoped
- **WHEN** a user queries VMC
- **THEN** it is accessible without namespace specification (cluster-scoped resource)

---

### Requirement: VolumeModifyContent CRD shall track individual volume modifications
The VolumeModifyContent is a cluster-scoped Custom Resource that tracks the modification status of an individual volume. Each VolumeModifyClaim creates one VolumeModifyContent per matching PersistentVolume.

#### Scenario: VolumeModifyContent Spec fields
- **WHEN** a VolumeModifyContent is created by the controller
- **THEN** the Spec includes: VMCName (reference to the parent VolumeModifyClaim), SourceVolume (PVC namespace/name), Parameters (CSI driver-specific modification parameters)

#### Scenario: VolumeModifyContent Status fields
- **WHEN** the VolumeModifyContent is processed
- **THEN** the Status includes: Phase (Pending/InProgress/Completed/Failed), and error message if failed

#### Scenario: VolumeModifyContent lifecycle phases
- **WHEN** a VolumeModifyContent is created
- **THEN** its Phase is set to "Pending"
- **WHEN** the modification is being applied via DR-CSI gRPC
- **THEN** its Phase is set to "InProgress"
- **WHEN** the modification completes successfully
- **THEN** its Phase is set to "Completed"
- **WHEN** the modification fails
- **THEN** its Phase is set to "Failed"

#### Scenario: VolumeModifyContent is cluster-scoped
- **WHEN** a user queries VolumeModifyContent
- **THEN** it is accessible without namespace specification (cluster-scoped resource)
