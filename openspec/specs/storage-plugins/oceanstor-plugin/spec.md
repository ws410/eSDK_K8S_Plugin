## ADDED Requirements

### Requirement: OceanStor plugin shall manage OceanStor and Dorado storage arrays
The OceanStor plugin shall implement the StoragePlugin interface for oceanstor-san and oceanstor-nas storage types, supporting Dorado V3/V5/V6/V7 and OceanStor V3/V5.

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
