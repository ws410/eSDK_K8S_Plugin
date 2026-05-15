## ADDED Requirements

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
