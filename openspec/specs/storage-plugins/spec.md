## ADDED Requirements

### Requirement: storage-plugins shall manage storage array operations
The storage plugin system implements the StoragePlugin interface for different Huawei storage types. Each plugin handles array-specific operations: volume creation/deletion/expansion, attachment/detachment, snapshot management, capability discovery, and parameter validation.

---

### Requirement: OceanStor plugin shall manage OceanStor and Dorado storage arrays
The OceanStor plugin implements the StoragePlugin interface for oceanstor-san and oceanstor-nas storage types, supporting Dorado V3/V5/V6/V7 and OceanStor V3/V5.

#### Scenario: Initialize OceanStor plugin
- **WHEN** the plugin is initialized with backend configuration
- **THEN** it creates an OceanStor client, logs in, sets system info, and for Dorado V6/V7 switches to the V6 client with LIF management

#### Scenario: Create LUN volume
- **WHEN** CreateVolume is called with volumeType=lun
- **THEN** the plugin creates a LUN on the storage array with the specified size, storage pool, and parameters (allocType, QoS, etc.)

#### Scenario: Create FileSystem volume
- **WHEN** CreateVolume is called with volumeType=fs
- **THEN** the plugin creates a FileSystem on the storage array with the specified parameters

#### Scenario: Update backend capabilities
- **WHEN** the plugin updates capabilities
- **THEN** it queries the license features and sets: SupportThin (SmartThin), SupportThick (non-Dorado), SupportQoS (SmartQoS), SupportMetro (HyperMetro), SupportMetroNAS (HyperMetroNAS), SupportReplication (HyperReplication), SupportClone (HyperClone/HyperCopy), SupportApplicationType (Dorado V6/V7 only)

#### Scenario: Attach volume via iSCSI/FC
- **WHEN** AttachVolume is called
- **THEN** the plugin creates/updates the host, adds initiators, creates mapping views, and returns the mapping info (portals, IQNs/WWNs, LUN WWN, host LUN IDs)

#### Scenario: Create snapshot
- **WHEN** CreateSnapshot is called
- **THEN** the plugin creates a HyperSnap snapshot and returns the snapshot metadata (SizeBytes, ParentID, CreationTime)

#### Scenario: Expand volume
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the LUN/FileSystem to the new size and returns whether node expansion is required

#### Scenario: Delete volume
- **WHEN** DeleteVolume is called
- **THEN** the plugin deletes the LUN or FileSystem from the storage array

#### Scenario: Query volume
- **WHEN** QueryVolume is called
- **THEN** the plugin returns volume metadata including size, LUN WWN, and other attributes

#### Scenario: Detach volume
- **WHEN** DetachVolume is called
- **THEN** the plugin removes the host mapping for the volume from the specified host

#### Scenario: Delete snapshot
- **WHEN** DeleteSnapshot is called with snapshotParentId and snapshotName
- **THEN** the plugin deletes the HyperSnap snapshot from the storage array

#### Scenario: Handle Dorado V6/V7 client switch
- **WHEN** the plugin detects the storage array is Dorado V6 or V7
- **THEN** it switches from the standard client to the V6 client and configures LIF (Logical Interface) management for NAS operations

#### Scenario: Support QoS parameter validation
- **WHEN** SupportQoSParameters is called with QoS configuration
- **THEN** the plugin validates the QoS parameters against the storage array's supported QoS types (IOPS, bandwidth)

#### Scenario: Attach volume with HyperMetro dual-site handling
- **WHEN** AttachVolume is called for a HyperMetro-enabled volume
- **THEN** the handler method checks storage online status: if both local and remote are online, uses metroHandler for dual-site attach; if only local is online, falls back to commonHandler with local client; if only remote is online, falls back to commonHandler with remote client; if both are offline, returns error

#### Scenario: Expand NAS volume with logic port site assertion
- **WHEN** ExpandVolume is called for an OceanStor NAS volume and metroRemotePlugin is nil
- **THEN** the plugin calls assertLogicPortRunOnOwnSite to verify the NAS logic port is running on the local site before proceeding with expansion

#### Scenario: DTree volume operations
- **WHEN** CreateVolume is called for an OceanStor DTree volume
- **THEN** the plugin validates the parentname against the backend's parentname configuration using getValidParentname: if both SC and backend parentname are set, they must match; if one is empty, the other is used; if both are empty, returns error
- **WHEN** DetachVolume is called for a DTree volume with nfsAutoAuthClient enabled
- **THEN** the plugin gets filtered IPs by CIDRs, calls dtree.AutoManageAuthClient with NoAccess to remove NFS client authorization, and if IOIsolation is true, calls dtree.CheckAllClientsStatus to verify all clients are disconnected

---

### Requirement: DTree plugin shall manage DTree quota volumes
The DTree plugin (OceanstorDTreePlugin) implements the StoragePlugin interface for oceanstor-dtree storage type, managing directory tree quotas on OceanStor NAS systems. It extends the OceanstorPlugin with DTree-specific operations.

#### Scenario: Initialize DTree plugin
- **WHEN** the plugin is initialized
- **THEN** it extracts parentname from parameters (optional, must be string if provided), verifies protocol and portals, validates and creates NfsAutoAuthClient from parameters, and calls the parent OceanstorPlugin init

#### Scenario: Create DTree volume
- **WHEN** CreateVolume is called with volumeType=dtree
- **THEN** the plugin resolves the volume name from PV name or parameters (for Dorado V6/V7 only), determines the valid parentname from StorageClass parameter or backend config, sets vstoreId and parentname in parameters, creates the DTree quota, and sets the DTreeParentName on the volume object

#### Scenario: Query DTree volume
- **WHEN** QueryVolume is called for a DTree volume
- **THEN** the plugin determines the valid parentname from parameters or backend config, and queries the DTree by name, parentName, and vStoreId

#### Scenario: Delete DTree volume
- **WHEN** DeleteDTreeVolume is called
- **THEN** the plugin removes the DTree quota using the volume name, parent name, and vStoreId

#### Scenario: Expand DTree volume
- **WHEN** ExpandDTreeVolume is called
- **THEN** the plugin converts the spaceHardQuota from sector size to bytes using TransK8SCapacity, expands the DTree quota on the array, and returns NodeExpansionRequired=false

#### Scenario: Attach DTree volume with auto auth client
- **WHEN** AttachVolume is called for a DTree volume
- **THEN** the plugin returns the DTreeParentName in the mapping info; if nfsAutoAuthClient is enabled, it filters node IPs by CIDRs, and calls AutoManageAuthClient with ReadWrite access on the DTree

#### Scenario: Detach DTree volume with auto auth client cleanup
- **WHEN** DetachVolume is called for a DTree volume
- **THEN** if nfsAutoAuthClient is enabled, the plugin filters node IPs by CIDRs, calls AutoManageAuthClient with NoAccess, and if IOIsolation is true, checks all clients status before returning

#### Scenario: Reject unsupported operations on DTree
- **WHEN** DeleteVolume (non-DTree variant), ExpandVolume (non-DTree variant), CreateSnapshot, DeleteSnapshot, or ModifyVolume is called
- **THEN** the plugin returns an error indicating the operation is not implemented (use the DTree-specific variants instead)

#### Scenario: DTree disables advanced capabilities
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin inherits from OceanstorPlugin but sets SupportMetro=false, SupportMetroNAS=false, SupportReplication=false, SupportClone=false, SupportApplicationType=false, SupportQoS=false; updates SupportThin for Dorado and updates NFS4 capability from NFS service settings

#### Scenario: DTree pool capacities are zero
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin returns zero capacities for all pool names (DTree uses quota-based capacity, not pool-based)

#### Scenario: Validate DTree parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies config parameters, validates DTree-specific params (protocol, portals, parentname), creates a test client, performs ValidateLogin, and logs out

#### Scenario: Create DTree volume with parentname and Dorado V6/V7 name template
- **WHEN** CreateVolume is called for a DTree volume
- **THEN** the plugin resolves the volume name from the PV name template (validating it contains {{.PVCNamespace}} and {{.PVCName}}, executing with metadata and appending "-{{.PVCUid}}"); validates parentname: if both StorageClass and backend parentname are set to different values, returns an error

#### Scenario: Attach DTree volume with parentName resolution
- **WHEN** AttachVolume is called for a DTree volume
- **THEN** the attachDTreeVolume function first checks volumeContext[DTreeParentKey]; if found, returns it; otherwise falls back to p.parentName; if nfsAutoAuthClient is enabled and parentName cannot be determined, returns error "failed to get parent name"

---

### Requirement: A-Series plugin shall manage OceanStor A-Series NAS and DTree
The A-Series plugin (OceanstorASeriesPlugin) implements the StoragePlugin interface for oceanstor-a-series-nas storage type, supporting NFS and DTFS protocols.

#### Scenario: Initialize A-Series plugin
- **WHEN** the plugin is initialized
- **THEN** it validates protocol is "nfs" or "dtfs", verifies portals (required for NFS with exactly 1 portal; not required for DTFS), creates an A-Series client, logs in, sets system info, and optionally logs out based on keepLogin flag

#### Scenario: Create A-Series NAS volume
- **WHEN** CreateVolume is called for oceanstor-a-series-nas
- **THEN** the plugin resolves the volume name from PV name or parameters, converts parameters to CreateASeriesVolumeParameter struct, generates the volume model with protocol, and creates the filesystem/DTree on the A-Series storage

#### Scenario: Query A-Series volume with workload type
- **WHEN** QueryVolume is called
- **THEN** the plugin extracts applicationType from parameters as workloadType and queries the filesystem through the A-Series API

#### Scenario: Delete A-Series NAS volume with KvCacheStoreId
- **WHEN** DeleteVolume is called for an A-Series NAS volume
- **THEN** the plugin extracts KvCacheStoreId from params (if present) and includes it in the delete model for the A-Series API

#### Scenario: Expand A-Series volume
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the filesystem capacity through the A-Series API and returns NodeExpansionRequired=false

#### Scenario: Handle A-Series specific protocols
- **WHEN** the plugin is initialized
- **THEN** it validates the protocol is either "nfs" or "dtfs" for A-Series storage

#### Scenario: A-Series capabilities from license
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin queries license features and sets: SupportThin=true, SupportApplicationType=true, SupportQoS based on SmartQoS feature, SupportThick=false, SupportMetro=false, SupportReplication=false, SupportClone=false; also queries NFS service settings to update NFS protocol capabilities (SupportNFS3/4/41/42)

#### Scenario: A-Series specifications include WWN
- **WHEN** getBackendSpecifications is called
- **THEN** the plugin returns LocalDeviceSN, VStoreID, VStoreName, and DeviceWWN

#### Scenario: Update pool capabilities with vStore quota
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin queries vStore capacity (if vStoreName is set), parses nasCapacityQuota and nasFreeCapacityQuota, and combines with pool data; if quota is not set (nasCapacityQuota=0 or nasFreeCapacityQuota=-1), returns empty capacity map

#### Scenario: Reject unsupported operations on A-Series
- **WHEN** CreateSnapshot, DeleteSnapshot, DeleteDTreeVolume, ExpandDTreeVolume, or ModifyVolume is called
- **THEN** the plugin returns an error indicating the storage type does not support the requested feature

#### Scenario: SupportQoSParameters validates range for A-Series
- **WHEN** SupportQoSParameters is called
- **THEN** the plugin calls smartx.CheckQoSParametersValueRange to validate QoS parameter values

#### Scenario: Validate A-Series parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies parameters, protocol, portals, creates a test client, performs ValidateLogin, and logs out

#### Scenario: Create A-Series volume with advanced options
- **WHEN** CreateVolume is called with advancedOptions parameter set to a JSON string
- **THEN** the genCreateVolumeModel function unmarshals the JSON into the volume creation model, allowing storage-specific options to be passed through
- **Note**: If EnableKVCache is set, passes EnableKVCache, EnableTimeAwareGC, and GCTimeThreshold for KVCache support; for DTFS protocol, authUser is required and authUsers are split by ";"

#### Scenario: Create A-Series DTree volume with parentname
- **WHEN** CreateVolume is called for oceanstor-aseries-dtree
- **THEN** the OceanstorASeriesDtreePlugin validates the optional parentname parameter, passes it to genCreateDTreeModel along with protocol, and creates the DTree on the A-Series storage

#### Scenario: A-Series DTree capabilities and capacities override
- **WHEN** UpdateBackendCapabilities is called for oceanstor-aseries-dtree
- **THEN** the plugin calls the parent A-Series method and overrides: SupportApplicationType=false, SupportQoS=false, SupportThick=false, SupportQuota=true; UpdatePoolCapabilities returns zero capacities via getZeroPoolsCapacities

---

### Requirement: DME plugin shall manage A-Series via DME
The DME plugin (DMEASeriesPlugin) implements the StoragePlugin interface for oceanstor-a-series-nas-dme storage type, managing A-Series storage through the DME (Data Management Engine) API. It supports NFS and DTFS protocols.

#### Scenario: Initialize DME A-Series plugin
- **WHEN** the plugin is initialized
- **THEN** it validates protocol is "nfs" or "dtfs", verifies portals (required for NFS with exactly 1 portal; not required for DTFS), verifies storageDeviceSN exists in config, creates a DME client, logs in, sets system info with the device SN, and optionally logs out based on keepLogin flag

#### Scenario: Create volume via DME
- **WHEN** CreateVolume is called
- **THEN** the plugin resolves the volume name from PV name or parameters, converts parameters to CreateDmeVolumeParameter struct, generates the volume model with protocol and sector size, and creates the volume via the DME API

#### Scenario: Query volume via DME
- **WHEN** QueryVolume is called
- **THEN** the plugin queries the volume by name through the DME API and returns volume metadata

#### Scenario: Delete volume via DME
- **WHEN** DeleteVolume is called
- **THEN** the plugin deletes the volume by name and protocol through the DME API

#### Scenario: Expand volume via DME
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the volume capacity (size * sectorSize) through the DME API and returns NodeExpansionRequired=false

#### Scenario: Set default maxClientThreads for DME
- **WHEN** maxClientThreads is not specified in the backend configuration
- **THEN** the CLI sets it to DMEDefaultMaxClientThreads (different from the standard default)

#### Scenario: DME capabilities are fixed
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin returns fixed capabilities: SupportThin=true, all others (SupportApplicationType, SupportQoS, SupportThick, SupportMetro, SupportReplication, SupportClone, SupportMetroNAS, SupportConsistentSnapshot) = false

#### Scenario: Reject unsupported operations on DME A-Series
- **WHEN** CreateSnapshot, DeleteSnapshot, DeleteDTreeVolume, ExpandDTreeVolume, or ModifyVolume is called
- **THEN** the plugin returns an error indicating the storage type does not support the requested feature

#### Scenario: SupportQoSParameters always passes for DME
- **WHEN** SupportQoSParameters is called
- **THEN** the plugin returns nil (always passes, no actual validation)

#### Scenario: Update pool capabilities via DME
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin queries HyperScale pools from DME, converts capacities from MB to bytes using DmeCapacityUnitMb, and returns pool capacity maps

#### Scenario: Validate DME parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies parameters, protocol, portals, creates a test client, performs ValidateLogin, and logs out

---

### Requirement: FusionStorage plugin shall manage FusionStorage distributed storage
The FusionStorage plugin implements the StoragePlugin interface for fusionstorage-san, fusionstorage-nas, and fusionstorage-dtree storage types.

#### Scenario: Create FusionStorage SAN volume
- **WHEN** CreateVolume is called with volumeType=lun on fusionstorage-san
- **THEN** the plugin creates a volume on the FusionStorage cluster with the specified size and pool

#### Scenario: Create FusionStorage NAS volume
- **WHEN** CreateVolume is called with volumeType=fs on fusionstorage-nas
- **THEN** the plugin creates a shared filesystem on the FusionStorage cluster

#### Scenario: Create FusionStorage DTree volume
- **WHEN** CreateVolume is called with volumeType=dtree on fusionstorage-dtree
- **THEN** the plugin creates a DTree quota with the specified capacity (1-byte allocation unit)

#### Scenario: Handle FusionStorage allocation units
- **WHEN** capacity is calculated for SAN volumes
- **THEN** the plugin uses FusionAllocUnitBytes (1MB) as the allocation unit
- **WHEN** capacity is calculated for NAS volumes
- **THEN** the plugin uses FusionFileCapacityUnit (1KB) as the capacity unit

#### Scenario: FusionStorage SAN supports QoS and Clone
- **WHEN** UpdateBackendCapabilities is called for fusionstorage-san
- **THEN** the plugin sets SupportThin=true, SupportThick=false, SupportQoS=true, SupportClone=true

#### Scenario: FusionStorage NAS supports Quota
- **WHEN** UpdateBackendCapabilities is called for fusionstorage-nas
- **THEN** the plugin sets SupportThin=true, SupportThick=false, SupportQoS=true, SupportQuota=true, SupportClone=false, and checks NFS4/NFS41 capabilities via updateNFS4Capability

#### Scenario: Reject unsupported operations on FusionStorage
- **WHEN** CreateSnapshot is called for fusionstorage-nas
- **THEN** the plugin returns error "fusionstorage-nas not support snapshot feature"
- **WHEN** DeleteVolume is called for fusionstorage-dtree
- **THEN** the plugin returns error "fusionstorage-dtree not support DeleteVolume feature"; use DeleteDTreeVolume instead
- **WHEN** ExpandVolume is called for fusionstorage-dtree
- **THEN** the plugin returns error "fusionstorage-dtree not support ExpandVolume feature"; use ExpandDTreeVolume instead

---

### Requirement: OceanDisk plugin shall manage OceanDisk storage arrays
The OceanDisk plugin implements the StoragePlugin interface for oceandisk-san storage type, supporting iSCSI, FC, RoCE, and RoCE-NVMe protocols.

#### Scenario: Initialize OceanDisk SAN plugin
- **WHEN** the plugin is initialized with backend configuration
- **THEN** it validates the protocol is one of [iscsi, fc, roce, roce-nvme], verifies portals for protocols that require them (iscsi, roce, roce-nvme), creates an OceanDisk client, logs in, and initializes the OceandiskAttacher with protocol, portals, and ALUA configuration

#### Scenario: Create OceanDisk volume
- **WHEN** CreateVolume is called on oceandisk-san
- **THEN** the plugin creates a namespace (LUN) on the OceanDisk array with the specified parameters

#### Scenario: Attach OceanDisk volume
- **WHEN** AttachVolume is called
- **THEN** the plugin retrieves the namespace info by name, calls the attacher's ControllerAttach with protocol-specific parameters, and returns the mapping info (portals, IQNs/WWNs, LUN WWN, host LUN IDs)

#### Scenario: Detach OceanDisk volume
- **WHEN** DetachVolume is called
- **THEN** the plugin retrieves the namespace info by name; if namespace doesn't exist, logs a warning and returns success (idempotent); otherwise calls ControllerDetach

#### Scenario: Delete OceanDisk volume
- **WHEN** DeleteVolume is called
- **THEN** the plugin deletes the namespace from the OceanDisk array

#### Scenario: Query OceanDisk volume
- **WHEN** QueryVolume is called
- **THEN** the plugin queries the namespace by name and returns volume metadata

#### Scenario: Expand OceanDisk volume
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the namespace to the new size and returns whether node expansion is required

#### Scenario: Reject unsupported operations on OceanDisk
- **WHEN** CreateSnapshot, DeleteSnapshot, DeleteDTreeVolume, ExpandDTreeVolume, or ModifyVolume is called on oceandisk-san
- **THEN** the plugin returns an error indicating the storage type does not support the requested feature

#### Scenario: Validate OceanDisk SAN parameters
- **WHEN** Validate is called with backend configuration
- **THEN** the plugin verifies parameters exist, validates protocol and portals, creates a test client, performs ValidateLogin, and logs out

#### Scenario: Get sector size for OceanDisk
- **WHEN** GetSectorSize is called
- **THEN** the plugin returns 512 bytes (standard sector size)
