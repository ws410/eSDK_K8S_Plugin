## ADDED Requirements

### Requirement: OceanDisk plugin shall manage OceanDisk storage arrays
The OceanDisk plugin shall implement the StoragePlugin interface for oceandisk-san storage type, supporting iSCSI, FC, RoCE, and RoCE-NVMe protocols.

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
