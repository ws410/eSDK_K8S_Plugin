## ADDED Requirements

### Requirement: claim-processing shall handle VolumeModifyClaim lifecycle
The volume-modify-controller shall watch VolumeModifyClaim resources, find matching PersistentVolumes based on the Source (StorageClass), create VolumeModifyContent resources for each matching PV, track overall progress, and replace the StorageClass after all modifications complete. Only hyperMetro modification is currently implemented; tier-modify (SmartTier), qos-modify, and migration-modify (SmartMigration) are NOT implemented.

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
- **THEN** the controller rejects the request with an error indicating unsupported parameters; only hyperMetro modification is currently implemented (tier-modify, qos-modify, migration-modify are NOT implemented)

#### Scenario: Replace StorageClass after all modifications complete
- **WHEN** all VContents reach "Completed" status
- **THEN** the controller checks if the StorageClass parameters already contain the VMC parameters; if not, creates a backup StorageClass (<scName>-<claimName>), deletes the original StorageClass, and recreates it with the merged parameters; tolerates AlreadyExists errors during backup SC creation and NotFound errors during original SC deletion

#### Scenario: Cancel content processing when parent claim is deleted
- **WHEN** the content worker is processing a VolumeModifyContent and the parent VMC is deleted
- **THEN** the syncContent function detects the parent claim deletion and cancels the content processing
