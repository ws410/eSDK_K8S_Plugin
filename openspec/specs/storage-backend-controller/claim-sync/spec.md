## ADDED Requirements

### Requirement: claim-sync shall bind StorageBackendClaim to StorageBackendContent
The claim-sync process in the storage-backend-controller shall watch StorageBackendClaim resources, validate referenced ConfigMap and Secret resources, create a StorageBackendContent bound to the SBC, manage finalizers, and update the SBC status. The sync is orchestrated as an 8-task sequential task flow with automatic revert on failure.

#### Scenario: Sync a new StorageBackendClaim
- **WHEN** a new SBC is created with Provider, ConfigMapMeta, and SecretMeta set
- **THEN** the controller executes the sync task flow: sets status to Pending, removes stale ConfigMap/Secret finalizers, handles deletion if DeletionTimestamp is set, adds ClaimBoundFinalizer, creates StorageBackendContent (named "content-<claim-UID>"), updates SBC status from Content status, and propagates spec changes to Content; the Content's Spec includes Provider, ConfigmapMeta, SecretMeta, BackendClaim, MaxClientThreads, and Parameters

#### Scenario: Sync SBC with missing ConfigMapMeta
- **WHEN** an SBC is created without ConfigMapMeta
- **THEN** the webhook validation rejects the request with an error indicating ConfigMapMeta is empty; the controller does NOT wait or requeue -- enforcement is at admission time, not controller time

#### Scenario: Sync SBC with missing SecretMeta
- **WHEN** an SBC is created without SecretMeta
- **THEN** the webhook validation rejects the request with an error indicating SecretMeta is empty; the controller does NOT wait or requeue -- enforcement is at admission time, not controller time

#### Scenario: Sync SBC update
- **WHEN** an existing SBC is updated (e.g., MaxClientThreads, SecretMeta, UseCert, CertSecret)
- **THEN** the controller detects changes via NeedChangeContent, updates the bound StorageBackendContent's Spec fields (MaxClientThreads, SecretMeta, UseCert, CertSecret) to match the SBC's Spec; the controller does NOT update the external ConfigMap or Secret resources themselves -- those are managed by CLI tools or external processes

#### Scenario: Sync SBC with provider mismatch
- **WHEN** an SBC has a Provider that doesn't match any registered provider
- **THEN** the webhook validation rejects the request during the Plugin.Validate() call which attempts to login to the storage array and fails; the controller does NOT set Phase to "Unavailable" -- rejection happens at admission time

#### Scenario: Sync SBC with ConfigMap finalizer management
- **WHEN** the claim-sync task flow runs for a new SBC
- **THEN** the removeConfigmapFinalizerTask checks if the ConfigMap has the ConfigMapFinalizer set; if the ConfigMap is not used by any other SBC in the same namespace (checked via isConfigmapUsed), the finalizer is removed; if still in use, the finalizer is preserved

#### Scenario: Sync SBC with Secret finalizer management
- **WHEN** the claim-sync task flow runs for a new SBC
- **THEN** the removeSecretFinalizerTask checks if the Secret has the SecretFinalizer set; if the Secret is not used by any other SBC in the same namespace (checked via isSecretUsed), the finalizer is removed; if still in use, the finalizer is preserved

#### Scenario: Sync SBC with Content creation idempotency
- **WHEN** the createContentTask creates a StorageBackendContent but it already exists (race condition)
- **THEN** the controller catches the creation error, calls GetContent to verify existence, and reuses the existing Content without returning an error

#### Scenario: Sync SBC with ResourceExpired retry during finalizer removal
- **WHEN** the removeContentFinalizer function encounters a ResourceExpired error during Content update
- **THEN** the function re-fetches the Content from the API server, removes the finalizer again, and retries the update

#### Scenario: Sync SBC with MaxClientThreads propagation
- **WHEN** the SBC's MaxClientThreads field differs from the Content's MaxClientThreads
- **THEN** the updateClaimTask detects the change via NeedChangeContent, updates both the SBC Status.MaxClientThreads and the Content Spec.MaxClientThreads

#### Scenario: Sync SBC sets Phase to Bound only when Content is ready
- **WHEN** the updateClaimStatusTask runs and the bound Content has Status.VendorName set (non-empty)
- **THEN** the isUpdateFinalClaimStatus function sets the SBC Phase to "Bound"; if Content.Status is nil or ContentName/VendorName are empty, the Phase remains unchanged
