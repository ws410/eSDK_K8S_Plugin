## ADDED Requirements

### Requirement: pool-filtering shall select storage pools by capability, topology, and capacity
The pool filtering system shall filter candidate storage pools through multiple stages: capability filtering, topology filtering, and capacity filtering, using configurable filter functions.

#### Scenario: Filter pools by backend name
- **WHEN** the CreateVolume request specifies a backend in StorageClass parameters
- **THEN** the filterByBackendName function selects only pools belonging to the specified backend

#### Scenario: Filter pools by volume type
- **WHEN** the CreateVolume request specifies volumeType=lun
- **THEN** the filterByVolumeType function selects only SAN pools (oceanstor-san, fusionstorage-san, oceandisk-san)
- **WHEN** volumeType=fs
- **THEN** only NAS pools are selected (oceanstor-nas, oceanstor-a-series-nas, fusionstorage-nas)
- **WHEN** volumeType=dtree
- **THEN** only DTree pools are selected (oceanstor-dtree, fusionstorage-dtree, oceanstor-a-series-dtree)

#### Scenario: Filter pools by allocation type
- **WHEN** the CreateVolume request specifies allocType=thin
- **THEN** the filterByAllocType function selects only pools with SupportThin=true
- **WHEN** allocType=thick
- **THEN** only pools with SupportThick=true are selected

#### Scenario: Filter pools by QoS
- **WHEN** the CreateVolume request specifies QoS parameters
- **THEN** the filterByQos function selects only pools with SupportQoS=true and validates QoS parameters against each pool's plugin

#### Scenario: Filter pools by hyperMetro
- **WHEN** the CreateVolume request specifies hyperMetro=true
- **THEN** the filterByMetro function selects only pools with SupportMetro=true and a configured metroBackend

#### Scenario: Filter pools by replication
- **WHEN** the CreateVolume request specifies replication=true
- **THEN** the filterByReplication function selects only pools with SupportReplication=true and a configured replicaBackend

#### Scenario: Filter pools by clone source
- **WHEN** the CreateVolume request has a VolumeContentSource (snapshot or volume)
- **THEN** the filterBySupportClone function selects only pools with SupportClone=true

#### Scenario: Filter pools by NFS protocol
- **WHEN** the CreateVolume request specifies nfsProtocol (nfs3/nfs4/nfs41/nfs42)
- **THEN** the filterByNFSProtocol function selects only pools with the corresponding SupportNFS* capability (DME A-Series pools always pass)

#### Scenario: Filter pools by topology
- **WHEN** the CreateVolume request has AccessibilityRequirements with Requisite topologies
- **THEN** the FilterByTopology function selects only pools whose backend's SupportedTopologies match at least one requisite topology, including protocol topology combinations

#### Scenario: Filter pools by capacity
- **WHEN** the CreateVolume request specifies allocType=thick
- **THEN** the FilterByCapacity function selects only pools with FreeCapacity >= requestSize
- **WHEN** allocType=thin
- **THEN** all pools with SupportThin=true pass (no capacity check)

#### Scenario: Reject when no pools match
- **WHEN** no pools pass all filter stages
- **THEN** the system returns an error "no storage pool meets the requirements" with details about which filter stage failed

#### Scenario: Filter pools by storage pool name
- **WHEN** the CreateVolume request specifies storagepool in StorageClass parameters
- **THEN** the filterByStoragePool function selects only pools matching the specified pool name (if empty string, all pools pass)

#### Scenario: Filter pools by application type
- **WHEN** the CreateVolume request specifies applicationType in StorageClass parameters
- **THEN** the filterByApplicationType function selects only pools with SupportApplicationType=true (Dorado V6/V7 only); if applicationType is empty, all pools pass

#### Scenario: Filter pools by storage quota
- **WHEN** the CreateVolume request specifies storageQuota in StorageClass parameters
- **THEN** the filterByStorageQuota function selects only pools with SupportQuota=true and validates the quota configuration using fsUtils.IsStorageQuotaAvailable; returns error if quota validation fails

#### Scenario: Validate backend name for managed volume
- **WHEN** a managed volume (volume import) request is processed
- **THEN** the validateBackendName function compares the backend name from StorageClass parameters with the PVC annotation backend name; returns error if they differ

#### Scenario: Validate volume type for managed volume
- **WHEN** a managed volume (volume import) request is processed
- **THEN** the validateVolumeType function verifies that the volumeType from StorageClass matches the backend's pool storage type using filterByVolumeType; returns error "filterPools is empty" if mismatch
