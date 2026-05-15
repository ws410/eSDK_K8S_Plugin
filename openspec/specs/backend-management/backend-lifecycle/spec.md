## ADDED Requirements

### Requirement: backend-lifecycle shall manage backend registration, selection, and caching
The backend management system shall provide lifecycle operations for storage backends including registration, fetching, selection, and caching with thread-safe concurrent access.

#### Scenario: Register a new backend
- **WHEN** the storage-backend-controller creates a new StorageBackendContent
- **THEN** the BackendRegister handler builds a Backend object from the ConfigMap and Secret, analyzes pools from the configuration, adds protocol topology, initializes the storage plugin with login, and stores it in the thread-safe cache

#### Scenario: Select backend by name
- **WHEN** the CSI driver needs to operate on a specific backend
- **THEN** the BackendSelector.SelectBackend calls LoadOrRegisterOneBackend which first tries to load from cache; if not found, fetches from Kubernetes via FetchAndRegisterOneBackend and registers it

#### Scenario: Select backend when not in cache
- **WHEN** the CSI driver requests a backend that is not in the in-memory cache
- **THEN** LoadOrRegisterOneBackend calls FetchAndRegisterOneBackend which fetches the StorageBackendContent from Kubernetes API, builds the Backend object, and adds it to cache; returns error if the backend is offline or configuration is invalid

#### Scenario: Fetch backend configuration
- **WHEN** the sidecar controller needs to query backend status
- **THEN** the BackendFetcher fetches the StorageBackendContent from the Kubernetes API (not from cache), validates that Status is not nil and Capabilities are non-empty via contentCanSync, and returns the filtered content

#### Scenario: Handle backend not found
- **WHEN** a backend is requested by name but doesn't exist in the cache
- **THEN** LoadOrRegisterOneBackend calls FetchAndRegisterOneBackend which attempts to fetch from Kubernetes; if the backend does not exist or is offline, returns an error (not nil without error)

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

#### Scenario: UpdateOrRegisterOnlineBackend returns only last error
- **WHEN** UpdateOrRegisterOnlineBackend processes multiple SBCTs and some fail while others succeed
- **THEN** the function logs each error but continues processing; only the last encountered error is returned (intermediate errors are not accumulated)

#### Scenario: FetchAndRegisterAllBackend silently swallows fetch errors
- **WHEN** FetchAndRegisterAllBackend calls FetchAllBackends and the Kubernetes API call fails
- **THEN** the function logs a warning and returns without error (does NOT propagate the fetch error), skipping the consistency check

#### Scenario: FetchAndRegisterAllBackend skips consistency check on register failure
- **WHEN** FetchAndRegisterAllBackend calls UpdateOrRegisterOnlineBackend and it returns an error
- **THEN** the function returns immediately without calling CheckConsistency, leaving stale backends in cache

#### Scenario: Update backend with ReLogin
- **WHEN** UpdateStorageBackend is called via the DR-CSI provider
- **THEN** the function calls FetchAndRegisterOneBackend with checkOnline=false to re-fetch the backend, then calls bk.Plugin.ReLogin(ctx) to refresh the storage array session, and sets the SBCT online status to true

#### Scenario: GetBackendStats with registerOrUpdateOneBackend
- **WHEN** GetBackendStats is called via the DR-CSI provider
- **THEN** after fetching backend stats (capabilities, pools, online status), the registerOrUpdateOneBackend function updates the cache with new data; if the backend is not in cache, fetches from Kubernetes; errors in the update step are logged but NOT returned

#### Scenario: Subscribe to backend status changes
- **WHEN** RunSyncBackendTaskInBackground starts
- **THEN** it subscribes to backend status changes via pkgUtils.Subscribe(pkgUtils.BackendStatus, ...), and calls FetchAndRegisterAllBackend once at startup to initialize the cache

#### Scenario: RemoveOneBackend standalone function
- **WHEN** RemoveOneBackend is called with a backend name
- **THEN** the function removes the backend from the cache via cacheHandler.Delete, which triggers Plugin.Logout to release storage client connections

#### Scenario: FilterStoragePool alternative pool filtering
- **WHEN** FilterStoragePool is called directly (not via BackendSelector)
- **THEN** the function filters pools by the given criteria as an alternative entry point for pool selection, separate from the BackendSelector.SelectPoolPair flow

#### Scenario: BuildBackend with invalid tuple
- **WHEN** BuildBackend is called with a StorageBackendContent that has missing BackendClaim, ConfigmapMeta, or SecretMeta
- **THEN** the function returns an error "valid tuple failed"
