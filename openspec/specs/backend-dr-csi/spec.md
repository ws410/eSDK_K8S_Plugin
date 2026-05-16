## ADDED Requirements

### Requirement: DR-CSI StorageBackend service shall manage backends
The DR-CSI StorageBackend gRPC service provides backend registration, status querying, and lifecycle management for the disaster recovery CSI protocol.

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

#### Scenario: Handle backend registration failure
- **WHEN** backend.BuildBackend fails during registration (invalid config, plugin init failure)
- **THEN** the service returns a gRPC error with the build failure reason and does not add the backend to cache

#### Scenario: AddStorageBackend is idempotent for duplicate backends
- **WHEN** AddStorageBackend is called for a backend that already exists in the cache
- **THEN** the service calls FetchAndRegisterOneBackend which re-fetches from Kubernetes and updates the cache entry; no duplicate error is returned

#### Scenario: RemoveStorageBackend is idempotent for non-existent backends
- **WHEN** RemoveStorageBackend is called for a backend that does not exist in the cache
- **THEN** the cacheHandler.Delete is a no-op (safe on missing keys); no error is returned

#### Scenario: GetBackendStats skips offline backends
- **WHEN** GetBackendStats is called for a backend with Online=false
- **THEN** the service returns error "GetBackendStats backend: [X] is offline, skip get stats" without querying the storage array

#### Scenario: GetBackendStats with empty pools
- **WHEN** GetBackendStats is called for a backend with zero storage pools
- **THEN** the service returns an empty pools array in the response; no special handling for zero-pool backends
