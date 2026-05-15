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
