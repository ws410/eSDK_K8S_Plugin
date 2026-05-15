## ADDED Requirements

### Requirement: DTree plugin shall manage DTree quota volumes
The DTree plugin shall implement the StoragePlugin interface for oceanstor-dtree storage type, managing directory tree quotas on OceanStor NAS systems.

#### Scenario: Create DTree volume
- **WHEN** CreateVolume is called with volumeType=dtree
- **THEN** the plugin creates a DTree quota on the NAS filesystem with the specified size and parent directory

#### Scenario: Attach DTree volume
- **WHEN** AttachVolume is called for a DTree volume
- **THEN** the plugin returns the DTreeParentName in the mapping info for NFS mount path construction

#### Scenario: Delete DTree volume
- **WHEN** DeleteDTreeVolume is called
- **THEN** the plugin removes the DTree quota using the volume name and parent name

#### Scenario: Expand DTree volume
- **WHEN** ExpandDTreeVolume is called
- **THEN** the plugin expands the DTree quota and returns whether node expansion is required

#### Scenario: Query DTree volume
- **WHEN** QueryVolume is called for a DTree volume
- **THEN** the plugin returns the volume metadata including size, parent name, and filesystem mode
