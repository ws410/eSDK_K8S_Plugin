## ADDED Requirements

### Requirement: content-sync shall manage StorageBackendContent lifecycle and trigger claim status updates
The content-sync process in the storage-backend-controller manages the StorageBackendContent lifecycle (finalizers, deletion) and triggers claim status updates. The actual backend status querying (capabilities, capacity, pools, online status) is performed by the sidecar controller via the DR-CSI provider, not by the main controller.

#### Scenario: Sync Content with finalizer management
- **WHEN** a StorageBackendContent is created or updated
- **THEN** the main controller checks if the Content needs the ContentBoundFinalizer added (DeletionTimestamp is nil and finalizer not present); if needed, adds the finalizer and updates the Content via API

#### Scenario: Sync Content with finalizer removal on deletion
- **WHEN** a StorageBackendContent has DeletionTimestamp set and has the ContentBoundFinalizer
- **THEN** the main controller removes the finalizer, updates the Content via API (with ResourceExpired retry), and updates the local contentStore

#### Scenario: Sync Content triggers claim status update
- **WHEN** the main controller's content-sync runs and the bound SBC needs status update (claim.Status is nil, BoundContentName is empty, StorageBackendId is empty while Content.Status.ContentName is set, or Phase is not Bound)
- **THEN** the controller enqueues the claim key to the claimQueue for re-processing

#### Scenario: Sync Content when backend is offline
- **WHEN** the backend storage array is unreachable (wrong password, network issue)
- **THEN** the sidecar controller (not the main controller) receives the offline status from the DR-CSI provider's GetBackendStats response and sets the Content's Online status to false; the sidecar does NOT directly set the SBC Phase -- the main controller's claim-sync detects the Content status change and updates the SBC Phase

#### Scenario: Sync Content when backend comes online
- **WHEN** a previously offline backend becomes reachable
- **THEN** the sidecar controller receives the online status from the DR-CSI provider and sets the Content's Online status to true; the main controller's claim-sync detects this and updates the SBC Phase to "Bound"

#### Scenario: Sync Content does NOT query backend directly
- **WHEN** the main controller's content-sync runs
- **THEN** the main controller does NOT query the storage array for capabilities, capacity, or pools; these are updated exclusively by the sidecar controller via the DR-CSI GetBackendStats gRPC call

#### Scenario: Sync Content aggregate Capacity map is never populated
- **WHEN** the sidecar controller updates the Content status
- **THEN** only the Pools array is updated with individual pool capacities; the aggregate Capacity map (TotalCapacity/UsedCapacity/FreeCapacity) is NEVER populated by either the main controller or the sidecar controller
