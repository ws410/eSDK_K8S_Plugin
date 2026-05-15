## ADDED Requirements

### Requirement: content-processing shall execute volume modifications via DR-CSI
The volume-modify-controller shall process each VolumeModifyContent by calling the DR-CSI ModifyVolume gRPC service to apply the modification to the storage array. Only hyperMetro modification (Local2HyperMetro and HyperMetro2Local) is currently implemented. The Content phase transitions are: Pending -> Creating -> Completed (or Rollback on deletion). There is NO "InProgress" or "Failed" phase in the code.

#### Scenario: Process a VolumeModifyContent
- **WHEN** a VolumeModifyContent is in "Pending" phase
- **THEN** the controller validates Parameters are not empty, validates hyperMetro parameter (must be "true" or "false"), validates metroPairSyncSpeed if present (integer in range [1, 4]), connects to the DR-CSI provider, calls ModifyVolume with the volume ID, StorageClassParameters, and MutableParameters, and updates the Content status to "Creating" (NOT "InProgress")

#### Scenario: Complete content on successful modification
- **WHEN** the DR-CSI ModifyVolume call succeeds
- **THEN** the controller sets the Content Phase to "Completed" with up to 10 retries at 100ms intervals for the status update

#### Scenario: Retry failed content modification
- **WHEN** the DR-CSI ModifyVolume call fails
- **THEN** the controller records a Warning event, returns the error for rate-limited requeue (exponential backoff 5s-5min); on the next retry, before updating status, the canRetry function validates that none of (VolumeHandle, VolumeModifyClaimName, SourceVolume, Parameters, StorageClassParameters) have changed; if any have changed, aborts the retry

#### Scenario: Rollback content on failed status update
- **WHEN** the setContentToCompleted function fails to update the Content status to "Completed" after 10 retries
- **THEN** the controller calls doRollback which sends a ModifyVolume gRPC call with hyperMetro="false" to revert the modification

#### Scenario: Rollback content on VMC deletion
- **WHEN** a VMC is being deleted and its Content is in "Creating" phase
- **THEN** the controller annotates the Content with modify.xuanwu.io/reclaimPolicy=rollback, the syncDeleteContent handler detects the annotation, sets the Content Phase to "Rollback", calls doRollback with generateRollbackParams (which sets hyperMetro="false"), and removes the finalizer after rollback completes

#### Scenario: Rollback content with generateRollbackParams
- **WHEN** doRollback is called for a Content
- **THEN** the generateRollbackParams function creates a rollback parameter map containing only hyperMetro="false" (other modification types like QoS, SmartTier, SmartMigration are NOT supported for rollback)

#### Scenario: Skip content processing when VMC is in Rollback phase
- **WHEN** the controller reconciles a VMC in "Rollback" phase
- **THEN** it cancels in-progress modifications and waits for all Contents to complete rollback before transitioning to "Deleting"

#### Scenario: Handle DR-CSI connection failure during modification
- **WHEN** the controller cannot connect to the DR-CSI gRPC server during ModifyVolume call
- **THEN** the error is returned for rate-limited requeue; the Content remains in "Creating" phase and will be retried on the next reconcile cycle

#### Scenario: Validate ModifyVolume parameters at provider level
- **WHEN** the DR-CSI provider receives a ModifyVolume request
- **THEN** the modifyHyperMetro function validates hyperMetro value is "true" or "false", selects the backend, verifies MetroBackend is configured, selects the remote pool, and calls the plugin's ModifyVolume method; for OceanstorNasPlugin, validates both local and remote WWN are non-empty, devices are different, and logic ports are running on own site

#### Scenario: Content sync cancellation on parent claim deletion
- **WHEN** the content worker is processing a VolumeModifyContent and the parent VMC is deleted
- **THEN** the syncContent function checks if the parent claim exists; if not found, returns early without processing the modification
