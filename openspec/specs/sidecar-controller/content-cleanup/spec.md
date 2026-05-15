## ADDED Requirements

### Requirement: content-cleanup shall handle SBCT deletion via informer events
The sidecar controller handles StorageBackendContent deletion when the Kubernetes CRD is deleted (detected via informer Delete event). The sidecar does NOT proactively delete SBCTs when a backend is removed from the DR-CSI provider -- that responsibility belongs to the storage-backend-controller. CheckConsistency is handled by the DR-CSI provider, not the sidecar.

#### Scenario: Delete Content when SBCT CRD is deleted
- **WHEN** the sidecar informer receives a Delete event for a StorageBackendContent
- **THEN** the syncContentByKey function detects the content is not found in the informer lister but exists in the local contentStore, and calls deleteContentCache to remove it from the local store

#### Scenario: Remove backend from provider on Content deletion
- **WHEN** the sidecar's deleteContentTask runs for a Content with DeletionTimestamp set
- **THEN** if the Content has a registered backendId (Status.ContentName is non-empty), the removeProviderBackend function calls the DR-CSI provider's RemoveStorageBackend to unregister the backend, then clears the Content status

#### Scenario: Handle Content deletion when backend was never registered
- **WHEN** the deleteContentTask runs for a Content with Status=nil or Status.ContentName=""
- **THEN** the removeProviderBackend function returns nil without calling RemoveStorageBackend (nothing to clean up)

#### Scenario: Remove backend from cache on deletion
- **WHEN** RemoveRegisteredOneBackend is called with a backend name
- **THEN** the cacheHandler.Delete function calls bk.Plugin.Logout(ctx) to release storage client connections, then removes the backend entry from the cache map

#### Scenario: CheckConsistency handled by DR-CSI provider
- **WHEN** FetchAndRegisterAllBackend completes registration of online backends
- **THEN** the DR-CSI provider (not the sidecar) calls CheckConsistency which compares cached backends against the SBCT list; any cached backend not in the SBCT list or with Online=false is deleted from the cache

#### Scenario: Sidecar does NOT proactively delete SBCTs
- **WHEN** a backend is removed from the DR-CSI provider but the SBCT CRD still exists in Kubernetes
- **THEN** the sidecar controller does NOT delete the SBCT; the storage-backend-controller is responsible for SBCT lifecycle management
