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
