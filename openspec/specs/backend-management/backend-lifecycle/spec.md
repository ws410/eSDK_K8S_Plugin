## ADDED Requirements

### Requirement: backend-lifecycle shall manage backend registration, selection, and caching
The backend management system shall provide lifecycle operations for storage backends including registration, fetching, selection, and caching with thread-safe concurrent access.

#### Scenario: Register a new backend
- **WHEN** the storage-backend-controller creates a new StorageBackendContent
- **THEN** the BackendRegister handler builds a Backend object from the ConfigMap and Secret, analyzes pools from the configuration, adds protocol topology, initializes the storage plugin with login, and stores it in the thread-safe cache

#### Scenario: Select backend by name
- **WHEN** the CSI driver needs to operate on a specific backend
- **THEN** the BackendSelector.SelectBackend loads the backend from the cache by name and returns it

#### Scenario: Fetch backend configuration
- **WHEN** the sidecar controller needs to query backend status
- **THEN** the BackendFetcher retrieves the backend from the cache and returns its configuration

#### Scenario: Handle backend not found
- **WHEN** a backend is requested by name but doesn't exist in the cache
- **THEN** the selector returns nil without error (caller handles the nil case)

#### Scenario: Clear backend cache
- **WHEN** the CSI controller is shutting down
- **THEN** the CacheWrapper.Clear releases all storage client connections and clears the backend cache

#### Scenario: Handle concurrent backend access
- **WHEN** multiple goroutines access the backend cache simultaneously
- **THEN** the cache provider uses mutex locks to ensure thread-safe read/write operations

#### Scenario: Rebuild backend when content name changes
- **WHEN** LoadOrRebuildOneBackend is called with a contentName different from the cached backend's ContentName
- **THEN** the backend.NeedRebuild check returns true, the cache entry is deleted, and the backend is re-fetched and re-registered from Kubernetes

#### Scenario: Skip rebuild when content name matches
- **WHEN** LoadOrRebuildOneBackend is called with a contentName matching the cached backend's ContentName
- **THEN** the backend.NeedRebuild check returns false, and the cached backend is returned directly without re-fetching

#### Scenario: Check consistency and remove stale backends
- **WHEN** FetchAndRegisterAllBackend completes registration of online backends
- **THEN** the CheckConsistency function compares cached backends against the SBCT list; any cached backend not in the SBCT list or with Online=false is deleted from the cache

#### Scenario: Establish hyperMetro backend relationships
- **WHEN** UpdateCacheBackendMetro is called after backend registration or update
- **THEN** the function iterates through all backends, finds pairs where MetroBackendName matches reciprocally and (MetroDomain or MetrovStorePairID matches), links them as MetroBackend references, and calls UpdateMetroRemotePlugin on both plugins

#### Scenario: Skip metro relationship when already established
- **WHEN** UpdateCacheBackendMetro processes a backend that already has MetroBackend set
- **THEN** the function skips this backend (condition: i.MetroBackend != nil)

#### Scenario: Update existing backend in cache
- **WHEN** UpdateAndAddBackend is called for a backend name that already exists in cache
- **THEN** the function calls UpdateCacheBackend to update the storage pools and hyperMetro relationships, then returns the existing cached backend

#### Scenario: Register only online backends
- **WHEN** UpdateOrRegisterOnlineBackend processes a list of SBCTs
- **THEN** it skips any SBCT with Status.Online=false or nil Status, and only registers/updates backends that are online
