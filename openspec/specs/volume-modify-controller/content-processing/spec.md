## ADDED Requirements

### Requirement: content-processing shall execute volume modifications via DR-CSI
The volume-modify-controller shall process each VolumeModifyContent by calling the DR-CSI ModifyVolume gRPC service to apply the modification to the storage array.

#### Scenario: Process a VolumeModifyContent
- **WHEN** a VolumeModifyContent is in "Pending" phase
- **THEN** the controller connects to the DR-CSI provider, calls ModifyVolume with the volume ID and Parameters, and updates the Content status to "InProgress"

#### Scenario: Complete content on successful modification
- **WHEN** the DR-CSI ModifyVolume call succeeds
- **THEN** the controller sets the Content Phase to "Completed"

#### Scenario: Fail content on modification error
- **WHEN** the DR-CSI ModifyVolume call fails
- **THEN** the controller sets the Content Phase to "Failed" with the error message, and retries with exponential backoff (baseDelay=5s, maxDelay=5min)

#### Scenario: Retry failed content
- **WHEN** a Content is in "Failed" phase and the VMC is not being deleted
- **THEN** the controller retries the modification with exponential backoff

#### Scenario: Rollback content on VMC deletion
- **WHEN** a VMC is being deleted and its Content is in "InProgress"
- **THEN** the controller cancels the modification and sets the Content Phase to "Completed" (rollback)
