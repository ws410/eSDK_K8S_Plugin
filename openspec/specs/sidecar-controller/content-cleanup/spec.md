## ADDED Requirements

### Requirement: content-cleanup shall remove SBCT when backend is deleted
The sidecar controller shall handle StorageBackendContent cleanup when the corresponding backend is removed from the DR-CSI provider.

#### Scenario: Delete Content when backend is unregistered
- **WHEN** a backend is unregistered from the DR-CSI provider
- **THEN** the sidecar controller deletes the corresponding StorageBackendContent resource

#### Scenario: Handle Content deletion when SBC still exists
- **WHEN** the sidecar controller attempts to delete a Content that still has a bound SBC
- **THEN** the controller logs a warning and lets the storage-backend-controller handle the SBC cleanup

#### Scenario: Remove backend from cache on deletion
- **WHEN** RemoveOneBackend is called with a storageBackendId
- **THEN** the cache provider deletes the backend entry from the cache and logs the removal

#### Scenario: Check consistency removes stale cached backends
- **WHEN** CheckConsistency runs after backend registration
- **THEN** it compares cached backends against the SBCT list; any cached backend not found in the SBCT list or with Online=false is deleted from the cache
