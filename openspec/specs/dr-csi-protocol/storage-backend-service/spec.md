## ADDED Requirements

### Requirement: DR-CSI StorageBackend service shall manage storage backends
The DR-CSI StorageBackend gRPC service shall provide backend registration, status querying, and lifecycle management for the disaster recovery CSI protocol.

#### Scenario: Register a backend
- **WHEN** the storage-backend-controller calls RegisterBackend via DR-CSI
- **THEN** the service registers the backend with its configuration (storage type, parameters, credentials) and initializes the storage plugin

#### Scenario: Query backend status
- **WHEN** the sidecar controller calls GetBackendStatus via DR-CSI
- **THEN** the service returns the backend's online status, capabilities, pool capacities, and device specifications

#### Scenario: Query all backends
- **WHEN** the sidecar controller calls ListBackends via DR-CSI
- **THEN** the service returns all registered backends with their status

#### Scenario: Unregister a backend
- **WHEN** a backend is deleted
- **THEN** the service unregisters the backend, logs out from the storage array, and releases the client connection

#### Scenario: Register backend with cache consistency check
- **WHEN** a backend is registered via DR-CSI
- **THEN** the service adds it to the backend cache via AddBackendToCache, which builds the backend, analyzes pools, adds protocol topology, and initializes the plugin with login

#### Scenario: Handle backend registration failure
- **WHEN** backend.BuildBackend fails during registration (invalid config, plugin init failure)
- **THEN** the service returns a gRPC error with the build failure reason and does not add the backend to cache

#### Scenario: AddStorageBackend silently ignores SplitMetaNamespaceKey error
- **WHEN** AddStorageBackend is called with a backend name that cannot be parsed by SplitMetaNamespaceKey
- **THEN** the error from SplitMetaNamespaceKey is silently overwritten (bug: the error variable is immediately overwritten by the next call), and the backend name extraction proceeds with potentially incorrect values

#### Scenario: AddStorageBackend is idempotent for duplicate backends
- **WHEN** AddStorageBackend is called for a backend that already exists in the cache
- **THEN** the service calls FetchAndRegisterOneBackend which re-fetches from Kubernetes and updates the cache entry; no duplicate error is returned

#### Scenario: RemoveStorageBackend is idempotent for non-existent backends
- **WHEN** RemoveStorageBackend is called for a backend that does not exist in the cache
- **THEN** the cacheHandler.Delete is a no-op (safe on missing keys); no error is returned

#### Scenario: UpdateStorageBackend returns response AND error (BUG)
- **WHEN** UpdateStorageBackend is called and SetStorageBackendContentOnlineStatus fails
- **THEN** the service returns both a non-nil response AND an error, which is a gRPC anti-pattern (should return either response with nil error, or nil response with error)

#### Scenario: GetBackendStats skips offline backends
- **WHEN** GetBackendStats is called for a backend with Online=false
- **THEN** the service returns error "GetBackendStats backend: [X] is offline, skip get stats" without querying the storage array

#### Scenario: GetBackendStats registerOrUpdateOneBackend silent failures
- **WHEN** GetBackendStats completes and registerOrUpdateOneBackend is called to update the cache
- **THEN** errors in the cache update step (fetch failure, nil status, update failure) are logged but NOT returned to the caller; stats response is still returned successfully

#### Scenario: GetBackendStats with empty pools
- **WHEN** GetBackendStats is called for a backend with zero storage pools
- **THEN** the service returns an empty pools array in the response; no special handling for zero-pool backends

#### Scenario: DR-CSI errors use plain errors.New without gRPC status codes
- **WHEN** any DR-CSI StorageBackend service method encounters an error
- **THEN** all errors are returned as plain errors.New() without gRPC status codes (InvalidArgument, NotFound, FailedPrecondition, Internal, etc.)
