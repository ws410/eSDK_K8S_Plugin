## ADDED Requirements

### Requirement: VolumeModifyClaim CRD shall define volume modification requests
The VolumeModifyClaim (VMC) is a cluster-scoped Custom Resource that represents a request to modify volumes (e.g., QoS, SmartTier, SmartMigration). It follows a job-like pattern where the Claim tracks overall progress and Contents track individual volume modifications.

#### Scenario: VMC Spec fields
- **WHEN** a user creates a VolumeModifyClaim
- **THEN** the Spec must include: Source (required, with Kind - default "StorageClass", Name, and optional Namespace), and Parameters (CSI driver-specific opaque key-value pairs for modification settings)

#### Scenario: VMC Status fields
- **WHEN** the VMC is processed by the controller
- **THEN** the Status includes: Phase (Pending/Creating/Completed/Rollback/Deleting), Contents (array of ModifyContents with ModifyContentName, SourceVolume, and Status), Ready (progress indicator, e.g., "2/5"), Parameters (echo of spec parameters), StartedAt (timestamp), CompletedAt (timestamp)

#### Scenario: VMC lifecycle phases
- **WHEN** a new VMC is created
- **THEN** its Phase is set to "Pending"
- **WHEN** VContents are being created but not all completed
- **THEN** its Phase is set to "Creating"
- **WHEN** all associated VContents are completed
- **THEN** its Phase is set to "Completed"
- **WHEN** the VMC receives a deletion request and starts rollback
- **THEN** its Phase is set to "Rollback"
- **WHEN** the VMC starts deleting
- **THEN** its Phase is set to "Deleting"

#### Scenario: VMC printed columns
- **WHEN** a user runs `kubectl get vmc`
- **THEN** the output includes columns: Status, Ready, Age (and with -o wide: SourceKind, SourceName, StartedAt, CompletedAt)

#### Scenario: VMC short name
- **WHEN** a user uses the short name
- **THEN** `vmc` is accepted as an alias for VolumeModifyClaim

#### Scenario: VMC is cluster-scoped
- **WHEN** a user queries VMC
- **THEN** it is accessible without namespace specification (cluster-scoped resource)
