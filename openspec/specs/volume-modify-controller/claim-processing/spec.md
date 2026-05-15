## ADDED Requirements

### Requirement: claim-processing shall handle VolumeModifyClaim lifecycle
The volume-modify-controller shall watch VolumeModifyClaim resources, find matching PersistentVolumes based on the Source (StorageClass), create VolumeModifyContent resources for each matching PV, and track overall progress.

#### Scenario: Process a new VolumeModifyClaim
- **WHEN** a new VMC is created with Source.Kind="StorageClass" and Source.Name=<sc-name>
- **THEN** the controller finds all PVs using the specified StorageClass, creates a VolumeModifyContent for each PV with the VMC's Parameters, sets the VMC Phase to "Creating", and records the StartedAt timestamp

#### Scenario: Process VMC with source namespace
- **WHEN** a VMC is created with Source.Namespace specified
- **THEN** the controller filters PVs by the specified namespace when finding matching volumes

#### Scenario: Update VMC progress
- **WHEN** VContents are being processed
- **THEN** the controller updates the VMC's Ready field (e.g., "2/5") and Contents array with each Content's name, source volume, and status

#### Scenario: Complete VMC when all Contents are done
- **WHEN** all associated VContents reach Completed or Failed status
- **THEN** the controller sets the VMC Phase to "Completed" and records the CompletedAt timestamp

#### Scenario: Rollback VMC on deletion
- **WHEN** a VMC in "Creating" phase is deleted
- **THEN** the controller sets the VMC Phase to "Rollback", waits for all VContents to complete rollback, then sets Phase to "Deleting" and cleans up

#### Scenario: Handle VMC with no matching PVs
- **WHEN** a VMC is created but no PVs match the Source StorageClass
- **THEN** the controller sets the VMC Phase to "Completed" immediately with Ready="0/0"
