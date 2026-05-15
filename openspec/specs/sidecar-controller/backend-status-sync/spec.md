## ADDED Requirements

### Requirement: backend-status-sync shall process SBCT informer events and trigger status updates
The sidecar controller processes StorageBackendContent informer events (Add/Update/Delete) and triggers status updates via the DR-CSI provider's GetBackendStats gRPC service. The sidecar does NOT discover new backends from DR-CSI that don't have corresponding SBCTs -- it only processes SBCTs that exist as Kubernetes CRDs. LoadOrRebuildOneBackend and UpdateCacheBackendMetro are handled by the DR-CSI provider, not the sidecar.

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

#### Scenario: LoadOrRebuildOneBackend handled by DR-CSI provider
- **WHEN** the CSI driver's BackendSelector.SelectBackend is called with a content name that differs from the cached backend's ContentName
- **THEN** the DR-CSI provider (not the sidecar) calls LoadOrRebuildOneBackend which deletes the stale cache entry and re-registers the backend from the updated SBCT

#### Scenario: UpdateCacheBackendMetro handled by DR-CSI provider
- **WHEN** a backend is added or updated in the cache via AddBackendToCache or UpdateCacheBackend
- **THEN** the DR-CSI provider (not the sidecar) calls UpdateCacheBackendMetro to establish MetroBackend references between reciprocal hyperMetro partner backends
