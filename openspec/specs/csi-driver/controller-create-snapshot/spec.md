## ADDED Requirements

### Requirement: CreateSnapshot RPC must create volume snapshots
The CreateSnapshot RPC shall create a snapshot of an existing volume on Huawei storage backends. The snapshot ID format is "backendName.parentID.snapshotName". Snapshots are created synchronously and are immediately ready to use (ReadyToUse=true).

#### Scenario: Create a snapshot from a volume
- **WHEN** the CO sends a CreateSnapshotRequest with SourceVolumeId, snapshot Name, and optional Parameters
- **THEN** the driver splits the SourceVolumeId to get backendName and volName, selects the backend, copies the request parameters, calls backend.Plugin.CreateSnapshot with volName, snapshotName, and parameters, and returns CreateSnapshotResponse with Snapshot containing: SizeBytes (int64 from plugin result), SnapshotId (format: "backendName.ParentID.snapshotName"), SourceVolumeId (original), CreationTime (int64 from plugin result converted to timestamppb), and ReadyToUse=true

#### Scenario: Reject CreateSnapshot with missing source volume ID
- **WHEN** the CO sends a CreateSnapshotRequest with an empty SourceVolumeId
- **THEN** the driver returns codes.InvalidArgument error with message "Volume ID missing in request"

#### Scenario: Reject CreateSnapshot with missing snapshot name
- **WHEN** the CO sends a CreateSnapshotRequest with an empty Name
- **THEN** the driver returns codes.InvalidArgument error with message "Snapshot Name missing in request"

#### Scenario: Reject CreateSnapshot when backend doesn't exist
- **WHEN** the CO sends a CreateSnapshotRequest with a SourceVolumeId referencing a non-existent backend
- **THEN** the driver returns codes.Internal error indicating the backend doesn't exist

#### Scenario: Reject CreateSnapshot when plugin creation fails
- **WHEN** the CO sends a CreateSnapshotRequest and backend.Plugin.CreateSnapshot returns an error
- **THEN** the driver returns codes.Internal error with the plugin error message
