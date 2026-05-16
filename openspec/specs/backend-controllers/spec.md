## ADDED Requirements

### Requirement: storage-backend-controller shall manage SBC lifecycle
The storage-backend-controller watches StorageBackendClaim resources, validates referenced ConfigMap and Secret resources, creates StorageBackendContent bound to the SBC, manages finalizers, and updates the SBC status.

#### Scenario: Sync a new StorageBackendClaim
- **WHEN** a new SBC is created with Provider, ConfigMapMeta, and SecretMeta set
- **THEN** the controller executes the sync task flow: sets status to Pending, removes stale ConfigMap/Secret finalizers, handles deletion if DeletionTimestamp is set, adds ClaimBoundFinalizer, creates StorageBackendContent (named "content-<claim-UID>"), updates SBC status from Content status, and propagates spec changes to Content; the Content's Spec includes Provider, ConfigmapMeta, SecretMeta, BackendClaim, MaxClientThreads, and Parameters

#### Scenario: Sync SBC update
- **WHEN** an existing SBC is updated (e.g., MaxClientThreads, SecretMeta, UseCert, CertSecret)
- **THEN** the controller detects changes via NeedChangeContent, updates the bound StorageBackendContent's Spec fields (MaxClientThreads, SecretMeta, UseCert, CertSecret) to match the SBC's Spec; the controller does NOT update the external ConfigMap or Secret resources themselves -- those are managed by CLI tools or external processes

#### Scenario: Sync SBC with Content creation idempotency
- **WHEN** the createContentTask creates a StorageBackendContent but it already exists (race condition)
- **THEN** the controller catches the creation error, calls GetContent to verify existence, and reuses the existing Content without returning an error

#### Scenario: Sync SBC sets Phase to Bound only when Content is ready
- **WHEN** the updateClaimStatusTask runs and the bound Content has Status.VendorName set (non-empty)
- **THEN** the isUpdateFinalClaimStatus function sets the SBC Phase to "Bound"; if Content.Status is nil or ContentName/VendorName are empty, the Phase remains unchanged

---

#### Scenario: Delete SBC with bound Content
- **WHEN** a StorageBackendClaim with a BoundContentName is deleted
- **THEN** the controller deletes the bound StorageBackendContent, removes finalizers from the ConfigMap, and deletes the ConfigMap and Secret

#### Scenario: Delete SBC without bound Content
- **WHEN** a StorageBackendClaim without a BoundContentName is deleted
- **THEN** the controller cleans up any partially created ConfigMap and Secret resources

#### Scenario: Delete SBC with certificate
- **WHEN** a StorageBackendClaim with CertSecret set is deleted
- **THEN** the controller also deletes the certificate Secret resource

---

#### Scenario: Sync Content with finalizer management
- **WHEN** a StorageBackendContent is created or updated
- **THEN** the main controller checks if the Content needs the ContentBoundFinalizer added (DeletionTimestamp is nil and finalizer not present); if needed, adds the finalizer and updates the Content via API

#### Scenario: Sync Content with finalizer removal on deletion
- **WHEN** a StorageBackendContent has DeletionTimestamp set and has the ContentBoundFinalizer
- **THEN** the main controller removes the finalizer, updates the Content via API (with ResourceExpired retry), and updates the local contentStore

#### Scenario: Sync Content triggers claim status update
- **WHEN** the main controller's content-sync runs and the bound SBC needs status update (claim.Status is nil, BoundContentName is empty, StorageBackendId is empty while Content.Status.ContentName is set, or Phase is not Bound)
- **THEN** the controller enqueues the claim key to the claimQueue for re-processing

#### Scenario: Sync Content does NOT query backend directly
- **WHEN** the main controller's content-sync runs
- **THEN** the main controller does NOT query the storage array for capabilities, capacity, or pools; these are updated exclusively by the sidecar controller via the DR-CSI GetBackendStats gRPC call

---

#### Scenario: Delete Content with bound Claim
- **WHEN** a StorageBackendContent with a BackendClaim reference is deleted
- **THEN** the controller clears the bound SBC's BoundContentName and resets its Phase to "Pending"

#### Scenario: Delete Content without bound Claim
- **WHEN** a StorageBackendContent without a BackendClaim reference is deleted
- **THEN** the controller completes the deletion without additional cleanup

---

### Requirement: sidecar-controller shall sync backend status via DR-CSI
The sidecar controller processes StorageBackendContent informer events (Add/Update/Delete) and triggers status updates via the DR-CSI provider's GetBackendStats gRPC service.

#### Scenario: Poll backend status via DR-CSI
- **WHEN** the sidecar controller's worker picks up an SBCT from the content queue
- **THEN** the syncContent task flow runs: initContentStatusTask initializes status if nil, deleteContentTask handles deletion if DeletionTimestamp is set, createContentTask registers backend with provider if not ready, updateContentTask updates backend credentials if spec changed, and getContentTask fetches stats via GetBackendStats gRPC call

#### Scenario: Sync backend status for existing SBCTs only
- **WHEN** the sidecar controller processes SBCT informer events
- **THEN** it only processes SBCTs that exist as Kubernetes CRDs; the sidecar does NOT discover new backends from the DR-CSI provider that don't have corresponding SBCT CRDs

#### Scenario: Handle DR-CSI connection failure
- **WHEN** the sidecar controller's GetBackendStats gRPC call fails due to connection error
- **THEN** the error propagates up the task flow, the item is re-queued with exponential backoff (5s start, 5min max), and will be retried on the next worker cycle

#### Scenario: Skip sidecar processing for non-matching provider
- **WHEN** the sidecar controller receives an SBCT event
- **THEN** the isMatchProvider function checks if content.Spec.Provider matches the sidecar's provider name; if not, the SBCT is skipped (each sidecar instance only processes SBCTs for its own provider)

#### Scenario: Optimize enqueue with needEnQueue check
- **WHEN** the sidecar informer receives an SBCT Update event
- **THEN** the needEnQueue function checks if only Pools, Capabilities, or Specification fields changed in status; if so, skips enqueue to prevent infinite loops (since the controller itself updates these fields)

---

#### Scenario: Update Online status from provider response
- **WHEN** the sidecar controller's getContentTask calls GetBackendStats via the DR-CSI provider
- **THEN** the provider checks IsSBCTOnline before fetching stats; if online, the response includes online=true; the shouldUpdateContentStatus function compares the response's Online field with the current SBCT status and updates if changed

#### Scenario: Update Online status to false when backend is unreachable
- **WHEN** a storage plugin client (OceanStor, FusionStorage, etc.) fails to communicate with the storage array
- **THEN** the plugin calls SetStorageBackendContentOnlineStatus to set SBCT.Status.Online=false and publishes a BackendStatus event; this is handled by the storage plugin, NOT by the sidecar controller directly

#### Scenario: Update pool capacities (no aggregate Capacity)
- **WHEN** the sidecar controller receives GetBackendStatsResponse with pool data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Pools with each pool's name and capacities (FreeCapacity, TotalCapacity, UsedCapacity); the aggregate SBCT.Status.Capacity map (TotalCapacity/UsedCapacity/FreeCapacity) is NEVER populated by either the sidecar or the main controller

#### Scenario: Update backend capabilities
- **WHEN** the sidecar controller receives GetBackendStatsResponse with capability data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Capabilities via DeepEqual comparison with: SupportThin, SupportThick, SupportQoS, SupportMetro, SupportReplication, SupportClone, SupportApplicationType, SupportMetroNAS, and NFS protocol support flags

#### Scenario: Update device specifications
- **WHEN** the sidecar controller receives GetBackendStatsResponse with specification data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Specification with LocalDeviceSN, RemoteDevicesSN, VStoreID, VStoreName, and updates SBCT.Status.SN with the device serial number

#### Scenario: Update vendor and version
- **WHEN** the sidecar controller receives GetBackendStatsResponse
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.VendorName (e.g., "Huawei") and SBCT.Status.ProviderVersion (CSI driver version)

---

#### Scenario: Delete Content when SBCT CRD is deleted
- **WHEN** the sidecar informer receives a Delete event for a StorageBackendContent
- **THEN** the syncContentByKey function detects the content is not found in the informer lister but exists in the local contentStore, and calls deleteContentCache to remove it from the local store

#### Scenario: Remove backend from provider on Content deletion
- **WHEN** the sidecar's deleteContentTask runs for a Content with DeletionTimestamp set
- **THEN** if the Content has a registered backendId (Status.ContentName is non-empty), the removeProviderBackend function calls the DR-CSI provider's RemoveStorageBackend to unregister the backend, then clears the Content status

#### Scenario: Handle Content deletion when backend was never registered
- **WHEN** the deleteContentTask runs for a Content with Status=nil or Status.ContentName=""
- **THEN** the removeProviderBackend function returns nil without calling RemoveStorageBackend (nothing to clean up)
