## ADDED Requirements

### Requirement: DR-CSI ModifyVolume service shall modify volume properties
The DR-CSI ModifyVolume gRPC service shall provide volume modification operations (QoS, SmartTier, SmartMigration) for the disaster recovery CSI protocol. The ModifyVolume RPC is implemented by storage plugins through the StoragePlugin.ModifyVolume interface method.

#### Scenario: Modify volume QoS
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with ModifyVolumeType=QoS
- **THEN** the service dispatches to the storage plugin's ModifyVolume method with the QoS type and parameters, and the plugin applies the QoS policy (IOPS limits, bandwidth limits) to the volume on the storage array

#### Scenario: Modify volume SmartTier
- **WHEN** the volume-modify-controller calls ModifyVolume with ModifyVolumeType=SmartTier
- **THEN** the service dispatches to the storage plugin's ModifyVolume method with the SmartTier type, and the plugin applies the SmartTier policy (automated data tiering) to the volume

#### Scenario: Modify volume SmartMigration
- **WHEN** the volume-modify-controller calls ModifyVolume with ModifyVolumeType=SmartMigration
- **THEN** the service dispatches to the storage plugin's ModifyVolume method with the SmartMigration type, and the plugin initiates online volume migration to the target pool

#### Scenario: Reject modify for unsupported volume
- **WHEN** ModifyVolume is called for a volume whose storage pool doesn't support the requested feature
- **THEN** the service returns a gRPC error indicating the feature is not supported

#### Scenario: Reject ModifyVolume on OceanDisk
- **WHEN** ModifyVolume is called on an oceandisk-san backend
- **THEN** the plugin returns error "oceandisk does not support volume modify feature"

#### Scenario: Reject ModifyVolume on A-Series
- **WHEN** ModifyVolume is called on an oceanstor-a-series-nas backend
- **THEN** the plugin returns error "oceanstor-a-series-nas storage does not support volume modify feature"

#### Scenario: Reject ModifyVolume on DME A-Series
- **WHEN** ModifyVolume is called on an oceanstor-a-series-nas-dme backend
- **THEN** the plugin returns error "oceanstor-a-series-nas-dme storage does not support volume modify feature"

#### Scenario: Reject ModifyVolume on DTree
- **WHEN** ModifyVolume is called on an oceanstor-dtree backend
- **THEN** the plugin returns error "not implement"
