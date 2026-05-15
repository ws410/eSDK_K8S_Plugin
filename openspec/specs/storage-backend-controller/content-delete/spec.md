## ADDED Requirements

### Requirement: content-delete shall clean up StorageBackendContent
The content-delete process shall handle StorageBackendContent deletion by removing the reference from the bound StorageBackendClaim.

#### Scenario: Delete Content with bound Claim
- **WHEN** a StorageBackendContent with a BackendClaim reference is deleted
- **THEN** the controller clears the bound SBC's BoundContentName and resets its Phase to "Pending"

#### Scenario: Delete Content without bound Claim
- **WHEN** a StorageBackendContent without a BackendClaim reference is deleted
- **THEN** the controller completes the deletion without additional cleanup
