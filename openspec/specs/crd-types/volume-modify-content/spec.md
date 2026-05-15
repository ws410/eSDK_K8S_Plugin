## ADDED Requirements

### Requirement: VolumeModifyContent CRD shall track individual volume modifications
The VolumeModifyContent is a cluster-scoped Custom Resource that tracks the modification status of an individual volume. Each VolumeModifyClaim creates one VolumeModifyContent per matching PersistentVolume.

#### Scenario: VolumeModifyContent Spec fields
- **WHEN** a VolumeModifyContent is created by the controller
- **THEN** the Spec includes: VMCName (reference to the parent VolumeModifyClaim), SourceVolume (PVC namespace/name), Parameters (CSI driver-specific modification parameters)

#### Scenario: VolumeModifyContent Status fields
- **WHEN** the VolumeModifyContent is processed
- **THEN** the Status includes: Phase (Pending/InProgress/Completed/Failed), and error message if failed

#### Scenario: VolumeModifyContent lifecycle phases
- **WHEN** a VolumeModifyContent is created
- **THEN** its Phase is set to "Pending"
- **WHEN** the modification is being applied via DR-CSI gRPC
- **THEN** its Phase is set to "InProgress"
- **WHEN** the modification completes successfully
- **THEN** its Phase is set to "Completed"
- **WHEN** the modification fails
- **THEN** its Phase is set to "Failed"

#### Scenario: VolumeModifyContent is cluster-scoped
- **WHEN** a user queries VolumeModifyContent
- **THEN** it is accessible without namespace specification (cluster-scoped resource)
