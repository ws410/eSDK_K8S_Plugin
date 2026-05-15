## ADDED Requirements

### Requirement: backend-status-sync shall poll backend status from DR-CSI
The sidecar controller shall periodically poll the DR-CSI provider for storage backend status and update the corresponding StorageBackendContent resources.

#### Scenario: Poll backend status via DR-CSI
- **WHEN** the sidecar controller runs its sync loop
- **THEN** it connects to the DR-CSI gRPC server, calls the StorageBackend service to get backend status (online, capabilities, capacity, pools, specifications), and updates the corresponding SBCT status

#### Scenario: Sync backend status for all registered backends
- **WHEN** the sidecar controller polls
- **THEN** it iterates through all registered backends in the DR-CSI provider and updates each corresponding SBCT

#### Scenario: Handle DR-CSI connection failure
- **WHEN** the sidecar controller cannot connect to the DR-CSI gRPC server
- **THEN** it logs the error and retries on the next sync cycle

#### Scenario: Sync backend status when backend is newly registered
- **WHEN** a new backend is registered in the DR-CSI provider
- **THEN** the sidecar controller creates a new StorageBackendContent and populates its status

#### Scenario: Handle backend registration via LoadOrRebuildOneBackend
- **WHEN** the sidecar controller detects a backend content name change
- **THEN** it calls LoadOrRebuildOneBackend which deletes the stale cache entry and re-registers the backend from the updated SBCT

#### Scenario: Sync backend status with MetroBackend linking
- **WHEN** the sidecar controller updates a backend's cache entry
- **THEN** UpdateCacheBackendMetro is called to establish MetroBackend references between reciprocal hyperMetro partner backends
