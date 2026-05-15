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
