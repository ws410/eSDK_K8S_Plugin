## ADDED Requirements

### Requirement: DeleteSnapshot RPC must delete volume snapshots
The DeleteSnapshot RPC shall delete a snapshot from Huawei storage backends. The SnapshotId format is "backendName.parentID.snapshotName". If the backend no longer exists, the request returns success with a warning (idempotent behavior).

#### Scenario: Delete a snapshot
- **WHEN** the CO sends a DeleteSnapshotRequest with a valid SnapshotId (format: "backendName.parentID.snapshotName")
- **THEN** the driver splits the SnapshotId using utils.SplitSnapshotId to get backendName, snapshotParentId, and snapshotName, selects the backend, calls backend.Plugin.DeleteSnapshot with snapshotParentId and snapshotName, and returns an empty DeleteSnapshotResponse on success

#### Scenario: Delete snapshot when backend doesn't exist
- **WHEN** the CO sends a DeleteSnapshotRequest with a SnapshotId referencing a non-existent backend
- **THEN** the driver logs a warning, returns success with an empty DeleteSnapshotResponse, and notes that manual cleanup from the array is required

#### Scenario: Reject DeleteSnapshot with missing snapshot ID
- **WHEN** the CO sends a DeleteSnapshotRequest with an empty SnapshotId
- **THEN** the driver returns codes.InvalidArgument error with message "Snapshot ID missing in request"

#### Scenario: Reject DeleteSnapshot when plugin deletion fails
- **WHEN** the CO sends a DeleteSnapshotRequest and backend.Plugin.DeleteSnapshot returns an error
- **THEN** the driver returns codes.Internal error with the plugin error message

---

### Requirement: ListSnapshots RPC is not implemented
The ListSnapshots RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests snapshot listing
- **WHEN** the CO sends a ListSnapshotsRequest
- **THEN** the driver returns codes.Unimplemented error with an empty message
