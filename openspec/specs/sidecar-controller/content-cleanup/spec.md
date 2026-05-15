## ADDED Requirements

### Requirement: content-cleanup shall remove SBCT when backend is deleted
The sidecar controller shall handle StorageBackendContent cleanup when the corresponding backend is removed from the DR-CSI provider.

#### Scenario: Delete Content when backend is unregistered
- **WHEN** a backend is unregistered from the DR-CSI provider
- **THEN** the sidecar controller deletes the corresponding StorageBackendContent resource

#### Scenario: Handle Content deletion when SBC still exists
- **WHEN** the sidecar controller attempts to delete a Content that still has a bound SBC
- **THEN** the controller logs a warning and lets the storage-backend-controller handle the SBC cleanup
