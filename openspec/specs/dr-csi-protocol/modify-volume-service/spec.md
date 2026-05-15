## ADDED Requirements

### Requirement: DR-CSI ModifyVolume service shall modify volume properties
The DR-CSI ModifyVolume gRPC service shall provide volume modification operations (QoS, SmartTier, SmartMigration) for the disaster recovery CSI protocol.

#### Scenario: Modify volume QoS
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with ModifyVolumeType=QoS
- **THEN** the service applies the QoS policy (IOPS limits, bandwidth limits) to the volume on the storage array

#### Scenario: Modify volume SmartTier
- **WHEN** the volume-modify-controller calls ModifyVolume with ModifyVolumeType=SmartTier
- **THEN** the service applies the SmartTier policy (automated data tiering) to the volume

#### Scenario: Modify volume SmartMigration
- **WHEN** the volume-modify-controller calls ModifyVolume with ModifyVolumeType=SmartMigration
- **THEN** the service initiates online volume migration to the target pool

#### Scenario: Reject modify for unsupported volume
- **WHEN** ModifyVolume is called for a volume whose storage pool doesn't support the requested feature
- **THEN** the service returns a gRPC error indicating the feature is not supported
