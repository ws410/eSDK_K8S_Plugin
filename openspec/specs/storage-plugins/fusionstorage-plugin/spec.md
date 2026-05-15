## ADDED Requirements

### Requirement: FusionStorage plugin shall manage FusionStorage distributed storage
The FusionStorage plugin shall implement the StoragePlugin interface for fusionstorage-san, fusionstorage-nas, and fusionstorage-dtree storage types.

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
