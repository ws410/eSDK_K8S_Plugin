## ADDED Requirements

### Requirement: volume-modification shall modify volume properties in bulk
The volume modification system allows bulk modification of volume properties (currently only hyperMetro conversion is implemented) through VolumeModifyClaim (VMC) and VolumeModifyContent CRDs. The volume-modify-controller processes VMCs, creates VContents for matching PVs, calls DR-CSI ModifyVolume gRPC to apply changes, tracks progress, and replaces the StorageClass after all modifications complete.

---

### Requirement: claim-processing shall handle VolumeModifyClaim lifecycle
The volume-modify-controller watches VolumeModifyClaim resources, finds matching PersistentVolumes based on the Source (StorageClass), creates VolumeModifyContent resources for each matching PV, tracks overall progress, and replaces the StorageClass after all modifications complete. Only hyperMetro modification is currently implemented.

#### Scenario: Process a new VolumeModifyClaim
- **WHEN** a new VMC is created with Source.Kind="StorageClass" and Source.Name=<sc-name>
- **THEN** the controller validates Source.Kind equals "StorageClass", validates the StorageClass exists and its Provisioner matches the controller's provisioner (default "csi.huawei.com"), validates Parameters are not empty and contain only supported keys (hyperMetro, metroPairSyncSpeed), finds ALL PVs using the specified StorageClass (no namespace filtering), creates a VolumeModifyContent for each PV with the VMC's Parameters, sets the VMC Phase to "Creating", and records the StartedAt timestamp

#### Scenario: Process VMC with source namespace (NOT implemented)
- **WHEN** a VMC is created with Source.Namespace specified
- **THEN** the controller does NOT filter PVs by namespace; it iterates ALL PVs in the cluster regardless of the Source.Namespace field

#### Scenario: Update VMC progress
- **WHEN** VContents are being processed
- **THEN** the controller updates the VMC's Ready field (e.g., "2/5") and Contents array with each Content's name, source volume, and status; the Ready format is "completed/total" where completed counts Contents in "Completed" phase

#### Scenario: Complete VMC when all Contents are done
- **WHEN** all associated VContents reach "Completed" status
- **THEN** the controller replaces the StorageClass (deletes and recreates with updated parameters), creates a backup StorageClass (<scName>-<claimName>) for safety, sets the VMC Phase to "Completed", and records the CompletedAt timestamp

#### Scenario: Rollback VMC on deletion
- **WHEN** a VMC in "Creating" phase is deleted
- **THEN** the controller sets the VMC Phase to "Rollback", annotates all associated VContents with modify.xuanwu.io/reclaimPolicy=rollback, deletes the VContents, waits for all VContents to complete rollback, then sets Phase to "Deleting" and removes the finalizer

#### Scenario: Handle VMC with no matching PVs
- **WHEN** a VMC is created but no PVs match the Source StorageClass
- **THEN** the controller sets the VMC Phase to "Completed" immediately with Ready="0/0"

#### Scenario: Handle VMC deletion with finalizer
- **WHEN** a VMC is deleted while in "Creating" or "Completed" phase
- **THEN** the controller processes the deletion via finalizer, sets Phase to "Rollback" if in "Creating" (triggers rollback of all Contents), waits for VContents to complete rollback, then removes the finalizer and deletes the VMC

#### Scenario: Process VMC with exponential backoff for failed Contents
- **WHEN** a VolumeModifyContent fails during modification
- **THEN** the controller retries with exponential backoff (baseDelay=5s, maxDelay=5min, delay sequence: 5s, 10s, 20s, 40s, 80s, 160s, 300s capped) until the Content succeeds or the VMC is deleted

#### Scenario: Skip VMC processing when no Contents need processing
- **WHEN** the VMC reconcile loop runs and all associated VContents are in "Completed" state
- **THEN** the controller updates the VMC Phase to "Completed" and records the CompletedAt timestamp

#### Scenario: Validate VMC parameters for hyperMetro
- **WHEN** a VMC is created with Parameters containing hyperMetro
- **THEN** the controller validates that hyperMetro value is exactly "true"; if metroPairSyncSpeed is also specified, validates it is parseable as an integer in range [1, 4]; rejects with error if validation fails

#### Scenario: Reject VMC with unsupported modification type
- **WHEN** a VMC is created with Parameters containing keys other than hyperMetro or metroPairSyncSpeed
- **THEN** the controller rejects the request with an error indicating unsupported parameters; only hyperMetro modification is currently implemented

#### Scenario: Replace StorageClass after all modifications complete
- **WHEN** all VContents reach "Completed" status
- **THEN** the controller checks if the StorageClass parameters already contain the VMC parameters; if not, creates a backup StorageClass (<scName>-<claimName>), deletes the original StorageClass, and recreates it with the merged parameters; tolerates AlreadyExists errors during backup SC creation and NotFound errors during original SC deletion

#### Scenario: Cancel content processing when parent claim is deleted
- **WHEN** the content worker is processing a VolumeModifyContent and the parent VMC is deleted
- **THEN** the syncContent function detects the parent claim deletion and cancels the content processing

---

### Requirement: content-processing shall execute volume modifications via DR-CSI
The volume-modify-controller processes each VolumeModifyContent by calling the DR-CSI ModifyVolume gRPC service to apply the modification to the storage array. Only hyperMetro modification (Local2HyperMetro and HyperMetro2Local) is currently implemented.

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

---

### Requirement: DR-CSI ModifyVolume service shall modify volume properties
The DR-CSI ModifyVolume gRPC service provides volume modification operations. Currently ONLY hyperMetro conversion (Local2HyperMetro and HyperMetro2Local) is implemented.

#### Scenario: Modify volume hyperMetro (Local to HyperMetro)
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with MutableParameters containing hyperMetro="true"
- **THEN** the modifyHyperMetro function validates the hyperMetro value, selects the backend, verifies MetroBackend is configured, selects the remote pool, and calls the plugin's ModifyVolume with ModifyVolumeType=Local2HyperMetro; the OceanstorNasPlugin validates both local and remote WWN are non-empty, devices are different, and logic ports are running on own site

#### Scenario: Modify volume hyperMetro (HyperMetro to Local)
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with MutableParameters containing hyperMetro="false"
- **THEN** the modifyHyperMetro function processes the request with ModifyVolumeType=HyperMetro2Local, converting the volume from HyperMetro active-active to local-only mode

#### Scenario: Reject ModifyVolume with invalid hyperMetro value
- **WHEN** the volume-modify-controller calls ModifyVolume with MutableParameters containing hyperMetro set to a value other than "true" or "false"
- **THEN** the modifyHyperMetro function returns error "hyperMetro value must be \"true\" or \"false\", \"X\" is invalid."

#### Scenario: Reject ModifyVolume when no metro backend configured
- **WHEN** ModifyVolume is called with hyperMetro="true" but the backend's MetroBackend is nil
- **THEN** the service returns error "have not configured hyper metro backend"

#### Scenario: Reject ModifyVolume when remote pool selection fails
- **WHEN** ModifyVolume is called and SelectRemotePool fails (no metro backend, no compatible pools, or hyperMetro+replication conflict)
- **THEN** the service returns error "select remote pool failed, backend name: X, error: Y"

#### Scenario: Modify volume with no-op for unrecognized parameters
- **WHEN** ModifyVolume is called with MutableParameters that do NOT contain the "hyperMetro" key
- **THEN** the modifyHyperMetro function returns (empty response, nil) immediately without performing any modification

#### Scenario: Reject ModifyVolume on unsupported storage types
- **WHEN** ModifyVolume is called on oceandisk-san, oceanstor-a-series-nas, oceanstor-a-series-nas-dme, or oceanstor-dtree backends
- **THEN** the plugin returns an error indicating the storage type does not support volume modify feature
