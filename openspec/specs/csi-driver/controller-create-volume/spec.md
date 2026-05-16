## ADDED Requirements

### Requirement: CreateVolume RPC must create storage volumes
The CreateVolume RPC shall create a new volume on Huawei storage backends (LUN, FileSystem, or DTree). It supports multiple creation modes: normal creation, managed volume (volume import), creation from snapshot, and volume cloning. The driver must validate request parameters, select an appropriate storage pool based on capability/topology/capacity filters, and handle topology constraints.

#### Scenario: Create a new volume normally
- **WHEN** the CO sends a CreateVolumeRequest with a unique name, valid CapacityRange (RequiredBytes > 0), VolumeCapabilities, and optional StorageClass parameters
- **THEN** the driver validates the request, processes StorageClass parameters (backend, volumeType, fsType, fsPermission, allocType, hyperMetro, replication, qos, applicationType, storagepool, parentname, reservedSnapshotSpaceRatio, description, volumeName template, nfsAutoAuthClient, nfsAutoAuthClientCIDRs, fileSystemMode, disableVerifyCapacity, advancedOptions), selects a storage pool via backendSelector using capability filters (backend, volumeType), primary filters (backend, pool, volumeType, allocType, qos, hyperMetro, replication, applicationType, storageQuota, sourceVolumeName, sourceSnapshotName, nfsProtocol), topology filters, and capacity filters, creates the volume on the Huawei storage array, and returns a CreateVolumeResponse with VolumeId (format: "backendName.volumeName"), CapacityBytes (aligned to sector size), VolumeContext (backend, name, fsPermission, dtreeParentName, disableVerifyCapacity, optional lunWWN, optional kvcacheStoreId), and AccessibleTopology

#### Scenario: Create volume with managed (imported) volume
- **WHEN** the CO sends a CreateVolumeRequest where the PVC annotations contain both manageVolumeName and manageBackendName (format: driverName + "/manageVolumeName" and driverName + "/manageBackendName")
- **THEN** the driver queries the existing volume on the specified backend, validates the backend name and volumeType match between StorageClass and PVC annotation, validates capacity matches the request exactly, and returns a CreateVolumeResponse without creating a new volume on the array

#### Scenario: Create volume from snapshot
- **WHEN** the CO sends a CreateVolumeRequest with VolumeContentSource containing a Snapshot source
- **THEN** the driver extracts sourceBackendName, snapshotParentId, and sourceSnapshotName from the snapshot ID (format: "backendName.parentID.snapshotName"), passes them as parameters (sourceSnapshotName, snapshotParentId, backend) to the backend plugin, filters pools by SupportClone capability, and creates a new volume from the snapshot data

#### Scenario: Create volume from clone (volume content source)
- **WHEN** the CO sends a CreateVolumeRequest with VolumeContentSource containing a Volume source, or with "cloneFrom" parameter in StorageClass (format: "backendName.volumeName")
- **THEN** the driver extracts sourceBackendName and sourceVolumeName, passes them to the backend plugin (sourceVolumeName, backend), filters pools by SupportClone capability, and creates a cloned volume

#### Scenario: Create volume with hyperMetro (active-active)
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter hyperMetro="true"
- **THEN** the driver validates that replication is not also set to "true" (mutually exclusive), filters local pools by SupportMetro capability, selects a remote pool from the metroBackend using SecondaryFilterFuncs (volumeType, allocType, qos, replication, applicationType), creates the volume with metroDomain and metrovStorePairID from the backend configuration, and returns the response

#### Scenario: Create volume with replication (remote replication)
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter replication="true"
- **THEN** the driver validates that hyperMetro is not also set to "true" (mutually exclusive), filters local pools by SupportReplication capability, selects a remote pool from the replicaBackend using SecondaryFilterFuncs, and creates the volume with replication configuration

#### Scenario: Reject CreateVolume with both hyperMetro and replication enabled
- **WHEN** the CO sends a CreateVolumeRequest with both hyperMetro="true" and replication="true"
- **THEN** the driver returns codes.Internal error indicating cannot create volume with both hyperMetro and replication properties

#### Scenario: Create volume with allocType thin or thick
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter allocType="thin" or allocType="thick"
- **THEN** the driver filters pools by SupportThin or SupportThick capability respectively; for thick allocation, also filters by pool FreeCapacity >= requestSize; for thin allocation, skips FreeCapacity check; after creation, for thick allocation, decrements the pool's FreeCapacity by requestSize

#### Scenario: Create volume with QoS parameters
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter qos set to a QoS specification string
- **THEN** the driver filters pools by SupportQoS capability, calls pool.Plugin.SupportQoSParameters to validate the QoS parameters against the storage array, and creates the volume with QoS configuration

#### Scenario: Create volume with topology constraints
- **WHEN** the CO sends a CreateVolumeRequest with AccessibilityRequirements containing Requisite and Preferred topologies
- **THEN** the driver processes the topology requirements into AccessibleTopology struct (RequisiteTopologies, PreferredTopologies), passes them to the backend pool selector, filters pools by matching backend SupportedTopologies (including protocol topology combinations), sorts pools by preferred topology order with random shuffling within each preference bucket, and returns AccessibleTopology in the response based on supported backend topologies

#### Scenario: Create volume with NFS protocol specification
- **WHEN** the CO sends a CreateVolumeRequest with VolumeCapabilities containing mountFlags that include "nfsvers=X" (where X is 3, 4, 4.0, 4.1, or 4.2)
- **THEN** the driver maps the nfsvers to the internal protocol name (nfs3, nfs4, nfs41, nfs42), passes it as nfsProtocol parameter to the backend, and filters pools by SupportNFS3/SupportNFS4/SupportNFS41/SupportNFS42 capabilities (DME A-Series pools always pass this filter)

#### Scenario: Create volume with volumeName template
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter volumeName set to a template string
- **THEN** the driver validates the template, extracts metadata from parameters (PVCNameKey, PVCNamespaceKey, PVNameKey), executes the Go template with metadata and appends volumeNameSuffix ("-{{.PVCUid}}"), and uses the generated name as the actual volume name on the storage array

#### Scenario: Create volume with parentname (DTree)
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter parentname
- **THEN** the driver validates that backend is also configured when parentname is set; the parentname is validated against the backend's parentname configuration (must match if both are set, or one can be empty); the valid parentname is used for DTree volume operations

#### Scenario: Create volume with nfsAutoAuthClient
- **WHEN** the CO sends a CreateVolumeRequest with protocol=nfs and StorageClass parameter nfsAutoAuthClient=true
- **THEN** the driver validates nfsAutoAuthClientCIDRs if provided (must be valid CIDR format), and during volume attachment, filters the node's host IPs by the CIDRs to determine authorized NFS clients

#### Scenario: Reject CreateVolume with invalid capacity
- **WHEN** the CO sends a CreateVolumeRequest with missing or invalid CapacityRange (RequiredBytes <= 0)
- **THEN** the driver returns codes.InvalidArgument error with message "CreateVolume CapacityRange must be provided"

#### Scenario: Reject CreateVolume with incompatible volume mode and type
- **WHEN** the CO sends a CreateVolumeRequest with VolumeMode=Block but volumeType=fs or volumeType=dtree
- **THEN** the driver returns codes.InvalidArgument error indicating the mismatch

#### Scenario: Reject CreateVolume with RWX access on LUN filesystem
- **WHEN** the CO sends a CreateVolumeRequest with volumeType=lun, volumeMode=FileSystem, and accessMode=ReadWriteMany
- **THEN** the driver returns codes.InvalidArgument error indicating this combination is not supported

#### Scenario: Reject CreateVolume with invalid fsType
- **WHEN** the CO sends a CreateVolumeRequest with fsType that is not one of ext2, ext3, ext4, or xfs
- **THEN** the driver returns codes.InvalidArgument error with the list of supported file types

#### Scenario: Reject CreateVolume with invalid fsPermission format
- **WHEN** the CO sends a CreateVolumeRequest with fsPermission that does not match the pattern [0-7][0-7][0-7]
- **THEN** the driver returns codes.InvalidArgument error indicating the format must be octal

#### Scenario: Reject CreateVolume with invalid description length
- **WHEN** the CO sends a CreateVolumeRequest with description parameter exceeding 255 characters
- **THEN** the driver returns codes.InvalidArgument error indicating the length exceeds the maximum

#### Scenario: Reject CreateVolume with invalid reservedSnapshotSpaceRatio
- **WHEN** the CO sends a CreateVolumeRequest with reservedSnapshotSpaceRatio that is not an integer or is outside the range [0, 50]
- **THEN** the driver returns codes.InvalidArgument error indicating the value must be in range [0, 50]

#### Scenario: Reject CreateVolume with invalid nfsAutoAuthClientCIDRs format
- **WHEN** the CO sends a CreateVolumeRequest with nfsAutoAuthClientCIDRs containing an invalid CIDR string
- **THEN** the driver returns an error indicating the invalid CIDR format

#### Scenario: Reject managed volume with content source
- **WHEN** the CO sends a CreateVolumeRequest for a managed volume (with manageVolumeName annotation) that also has VolumeContentSource set
- **THEN** the driver returns an error indicating managed volumes cannot have source content

#### Scenario: Reject managed volume with mismatched annotations
- **WHEN** the CO sends a CreateVolumeRequest where only one of manageVolumeName or manageBackendName is present in PVC annotations
- **THEN** the driver returns codes.FailedPrecondition error indicating both must be configured together

#### Scenario: Reject CreateVolume with invalid filesystemMode annotation
- **WHEN** the CO sends a CreateVolumeRequest where the PVC annotation filesystemMode is not "HyperMetro" or "local"
- **THEN** the driver returns an error indicating the invalid filesystemMode value

#### Scenario: Reject CreateVolume when no storage pool matches requirements
- **WHEN** the CO sends a CreateVolumeRequest and no storage pool passes all filter stages (capability, topology, capacity)
- **THEN** the driver returns codes.InvalidArgument or codes.Internal error with message "no storage pool meets the requirements" and details about which filter stage failed

#### Scenario: Reject CreateVolume when backend hyperMetro configuration is incomplete
- **WHEN** the CO sends a CreateVolumeRequest with hyperMetro="true" but the backend is missing metroBackend, or has metroBackend but missing both hyperMetroDomain and metrovStorePairID
- **THEN** the backend initialization fails with error indicating hyperMetro configuration is incorrect

#### Scenario: Reject CreateVolume with duplicate nfs protocol
- **WHEN** the CO sends a CreateVolumeRequest with multiple mountFlags containing different nfsvers values
- **THEN** the driver returns an error indicating duplicate nfs protocol

#### Scenario: Reject CreateVolume with unsupported nfs protocol version
- **WHEN** the CO sends a CreateVolumeRequest with mountFlags containing nfsvers other than 3, 4, 4.0, 4.1, or 4.2
- **THEN** the driver returns an error indicating unsupported nfs protocol version

#### Scenario: Create volume with capacity alignment to sector size
- **WHEN** the CO sends a CreateVolumeRequest with RequiredBytes that is not an integer multiple of the storage pool's sector size
- **THEN** the driver rounds up the capacity to the next sector size multiple using TransVolumeCapacity, logs the capacity adjustment, and returns the actual (aligned) CapacityBytes in the response

#### Scenario: Create volume with cloneFrom parameter
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter cloneFrom set to "backendName.volumeName"
- **THEN** the driver parses the cloneFrom value using utils.SplitVolumeId to extract sourceBackendName and sourceVolumeName, sets them in parameters as "backend" and "cloneFrom", filters pools by SupportClone capability, and creates a cloned volume

#### Scenario: Create volume with storagepool parameter
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter storagepool set to a specific pool name
- **THEN** the driver passes storagepool to the backend, and the filterByStoragePool function selects only pools matching the specified name (if empty, all pools pass)

#### Scenario: Create volume with applicationType parameter
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter applicationType set
- **THEN** the driver filters pools by SupportApplicationType capability (only Dorado V6/V7 support this), and passes the applicationType to the plugin for volume creation

#### Scenario: Create volume with storageQuota parameter
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter storageQuota set
- **THEN** the driver filters pools by SupportQuota capability, validates the storage quota configuration using fsUtils.IsStorageQuotaAvailable, and creates the volume with quota settings

#### Scenario: Create volume with volumeName annotation
- **WHEN** the CO sends a CreateVolumeRequest where the PVC annotation contains driverName + "/volumeName"
- **THEN** the driver validates the annotation value is not empty, sets it as "annVolumeName" in parameters for use by the backend plugin

#### Scenario: Create DTree volume with automatic parent selection
- **WHEN** the CO sends a CreateVolumeRequest with volumeType=dtree but no parentname specified in StorageClass
- **THEN** the selectDtreePool function filters candidate pools to only those with a non-empty GetDTreeParentName(), and returns error "no found any available dtree backend for volume" if no such pools exist

#### Scenario: Reject CreateVolume when DTree pool has no available parent
- **WHEN** the CO sends a CreateVolumeRequest with volumeType=dtree and all candidate DTree pools have empty DTreeParentName
- **THEN** the driver returns an error "no found any available dtree backend for volume"

#### Scenario: Create volume with advancedOptions parameter
- **WHEN** the CO sends a CreateVolumeRequest with StorageClass parameter advancedOptions set to a JSON string
- **THEN** the driver parses the JSON string into a map and merges it into the volume creation parameters via mergeAdvancedOptions during processCreateVolumeParametersAfterSelect, allowing storage-specific options to be passed to the backend plugin

#### Scenario: Create volume with restoreMode for snapshot (SAN-specific)
- **WHEN** the CO sends a CreateVolumeRequest with VolumeContentSource containing a Snapshot source and StorageClass parameter restoreMode="snapshot"
- **THEN** the resolveSnapshotLunName function returns the snapshot LUN name instead of the original LUN name, and for non-hyperMetro volumes, the SAN plugin creates the volume from the snapshot LUN data

#### Scenario: Create volume with restoreMode for hyperMetro snapshot
- **WHEN** the CO sends a CreateVolumeRequest with VolumeContentSource containing a Snapshot source, restoreMode="snapshot", and hyperMetro="true"
- **THEN** the resolveSnapshotLunName function returns the original LUN name (snapshot restore is not supported for hyperMetro volumes), and the volume is created from the original LUN

#### Scenario: Create volume with processCreateVolumeParametersAfterSelect
- **WHEN** the CO sends a CreateVolumeRequest and storage pool selection succeeds
- **THEN** the processCreateVolumeParametersAfterSelect function sets storagepool, metroDomain, and remoteStoragePool from the selected pools, calls IsCapacityAvailable to verify sector alignment (unless disableVerifyCapacity is set), and merges advancedOptions into parameters

#### Scenario: Reject CreateVolume when IsCapacityAvailable fails after pool selection
- **WHEN** the CO sends a CreateVolumeRequest and processCreateVolumeParametersAfterSelect calls IsCapacityAvailable with disableVerifyCapacity=false and the capacity is not a multiple of sector size
- **THEN** the driver returns codes.InvalidArgument error indicating the capacity is not a multiple of the sector size

#### Scenario: Create volume with default description
- **WHEN** the CO sends a CreateVolumeRequest without a description parameter
- **THEN** the processDescription function sets the description to the default value "Created from Kubernetes CSI"

#### Scenario: Reject CreateVolume when parentname requires backend
- **WHEN** the CO sends a CreateVolumeRequest with parentname set but backend not set
- **THEN** the processParentName function returns codes.InvalidArgument error indicating parentname requires backend to be configured

---

### Requirement: Pool filtering shall select storage pools by capability, topology, and capacity
The pool filtering system filters candidate storage pools through multiple stages: capability filtering, topology filtering, and capacity filtering, using configurable filter functions.

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
- **Note**: When backend has no SupportedTopologies configured or Requisite list is empty, the filter passes all pools

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

#### Scenario: Filter pools by NFS protocol for DME A-Series
- **WHEN** the CreateVolume request specifies nfsProtocol and the candidate pool is from a DME A-Series backend
- **THEN** the filterByNFSProtocol function always passes the pool (DME A-Series pools bypass NFS protocol capability checks)

#### Scenario: Filter pools with empty applicationType
- **WHEN** the CreateVolume request does not specify applicationType in StorageClass parameters
- **THEN** the filterByApplicationType function passes all pools regardless of SupportApplicationType capability

#### Scenario: Validate authClient for managed volume
- **WHEN** a managed volume (volume import) request is processed for an NFS protocol backend
- **THEN** the ValidateBackend function verifies that the backend's authClient configuration matches the expected NFS client settings

---

### Requirement: pool-weighting shall select the optimal storage pool by free capacity
The pool weighting system selects the optimal storage pool from filtered candidates based on free capacity, with support for local/remote pool pair selection for hyperMetro and replication scenarios.

#### Scenario: Select pool by maximum free capacity
- **WHEN** multiple pools pass all filter stages
- **THEN** the weightByFreeCapacity function selects the pool with the highest FreeCapacity

#### Scenario: Select local and remote pool pair
- **WHEN** hyperMetro or replication is enabled
- **THEN** the WeightPools function first selects the local pool by free capacity, then finds the corresponding remote pool pair from the metroBackend or replicaBackend

#### Scenario: Select remote pool independently
- **WHEN** only remote pool selection is needed
- **THEN** the SelectRemotePool function filters remote pools using SecondaryFilterFuncs (volumeType, allocType, qos, replication, applicationType) and selects by free capacity

#### Scenario: Reject hyperMetro and replication together
- **WHEN** both hyperMetro=true and replication=true are specified
- **THEN** the system returns an error "cannot create volume with hyperMetro and replication properties"

#### Scenario: Decrement free capacity for thick allocation
- **WHEN** a thick volume is created
- **THEN** the updateSelectPool function decrements the selected pool's FreeCapacity by the requestSize to prevent over-allocation

#### Scenario: Filter DTree pools by parent name availability
- **WHEN** volumeType=dtree and no parentname is specified in StorageClass
- **THEN** the selectDtreePool function filters candidate pools to only those with non-empty GetDTreeParentName(); returns error if no such pools exist

#### Scenario: Skip DTree pool filtering when parentname is specified
- **WHEN** volumeType=dtree and parentname is explicitly set in StorageClass parameters
- **THEN** the selectDtreePool function returns all candidate pools without filtering (parent name validation is handled elsewhere)

#### Scenario: Build pool pairs for local and remote selection
- **WHEN** hyperMetro or replication is enabled
- **THEN** the SelectPoolPair function iterates through each local pool, selects a corresponding remote pool via SelectRemotePool, builds SelectPoolPair structs (Local + Remote), and passes them to WeightPools for final selection

#### Scenario: Select remote pool returns nil when no match found
- **WHEN** SelectRemotePool is called for a hyperMetro or replication volume but no remote pools pass the SecondaryFilterFuncs
- **THEN** the function returns (nil, nil) -- not an error, allowing the local pool to be used without remote pairing

---

### Requirement: topology-matching shall match volume requests to backend topologies
The topology matching system matches CreateVolume requests with accessibility requirements to backend-supported topologies, including protocol-specific topology combinations.

#### Scenario: Match requisite topologies
- **WHEN** a CreateVolume request has AccessibilityRequirements with Requisite topologies
- **THEN** the filterPoolsOnTopology function selects only pools whose backend's SupportedTopologies match at least one requisite topology

#### Scenario: Sort pools by preferred topologies
- **WHEN** a CreateVolume request has Preferred topologies
- **THEN** the sortPoolsByPreferredTopologies function orders pools by preference match, shuffling pools randomly within each preference bucket to prevent hotspots

#### Scenario: Match protocol topology
- **WHEN** a topology requirement includes a protocol key (topology.kubernetes.io/protocol.iscsi)
- **THEN** the isTopologySupportedByBackend function extracts the protocol topology from the requirement and checks if the backend supports it via the protocol-specific topology label

#### Scenario: Add protocol topology to backend
- **WHEN** a backend is initialized with a protocol (e.g., iscsi)
- **THEN** the addProtocolTopology function adds protocol topology combinations to the backend's SupportedTopologies (e.g., topology.kubernetes.io/protocol.iscsi = csi.huawei.com)

#### Scenario: Handle backend without supported topologies
- **WHEN** a backend has no SupportedTopologies configured
- **THEN** the topology filter passes all pools (no topology constraint)

#### Scenario: Handle empty requisite topologies
- **WHEN** a CreateVolume request has AccessibilityRequirements but empty Requisite list
- **THEN** the topology filter passes all pools (no requisite constraint)

#### Scenario: Shuffle leftover pools after preferred topology matching
- **WHEN** sortPoolsByPreferredTopologies has processed all preferred topologies
- **THEN** the remaining pools that didn't match any preference are shuffled randomly and appended to the ordered list to prevent hotspots

#### Scenario: Protocol topology combination with existing topologies
- **WHEN** addProtocolTopology is called on a backend that already has SupportedTopologies configured
- **THEN** the function creates protocol topology combinations by copying each existing topology and adding the protocol key (e.g., topology.kubernetes.io/protocol.iscsi = csi.huawei.com), then appends these combinations plus a standalone protocol topology entry

#### Scenario: Protocol topology standalone entry
- **WHEN** addProtocolTopology is called on a backend
- **THEN** regardless of existing topologies, a standalone protocol topology entry is always added (map with only the protocol key and driver name as value)

---

### Requirement: hypermetro-replication shall manage active-active and remote replication
The hyperMetro-replication system manages hyperMetro (active-active) and replication (remote replication) configurations for storage pools, including remote pool selection and pair management.

#### Scenario: Select remote pool for hyperMetro
- **WHEN** hyperMetro=true is specified in CreateVolume
- **THEN** the SelectRemotePool function loads the metroBackend from cache, filters its pools using SecondaryFilterFuncs, and selects the optimal remote pool by free capacity

#### Scenario: Select remote pool for replication
- **WHEN** replication=true is specified in CreateVolume
- **THEN** the SelectRemotePool function loads the replicaBackend from cache, filters its pools, and selects the optimal remote pool

#### Scenario: Reject hyperMetro without metroBackend
- **WHEN** hyperMetro=true but the backend has no metroBackend configured
- **THEN** the system returns an error "no metro backend exists for volume"

#### Scenario: Reject replication without replicaBackend
- **WHEN** replication=true but the backend has no replicaBackend configured
- **THEN** the system returns an error "no replica backend exists for volume"

#### Scenario: Configure metro domain and vStore pair
- **WHEN** a volume is created with hyperMetro
- **THEN** the processCreateVolumeParametersAfterSelect function sets metroDomain, metrovStorePairID, and remoteStoragePool in the parameters for the plugin

#### Scenario: Reject hyperMetro with incomplete configuration
- **WHEN** a backend has metroBackend set but missing hyperMetroDomain or metrovStorePairID, OR has hyperMetroDomain/metrovStorePairID set but missing metroBackend
- **THEN** the NewBackend function returns error "hyperMetro configuration in backend <name> is incorrect"

#### Scenario: SelectRemotePool rejects hyperMetro with nil MetroBackend
- **WHEN** SelectRemotePool is called with hyperMetro=true but the local backend's MetroBackend field is nil
- **THEN** the function returns error "no metro backend of <name> exists for volume: <params>"
