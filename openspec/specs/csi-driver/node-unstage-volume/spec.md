## ADDED Requirements

### Requirement: NodeUnstageVolume RPC must unstage volumes from the node
The NodeUnstageVolume RPC shall remove a staged volume from the node's staging target path. The driver creates a manage.Manager for the backend and delegates the unstage operation. For SAN volumes, the driver must retrieve the device WWN (from disk file or target path), unmount the staging path, disconnect the volume, and clean up the WWN file. For NAS volumes, only unmount is needed. DTree volumes don't require staging/unstaging.

#### Scenario: Unstage a SAN volume
- **WHEN** the CO sends a NodeUnstageVolumeRequest with VolumeId and StagingTargetPath
- **THEN** the driver splits the VolumeId to get backendName, creates a SanManager, retrieves the device WWN (first from WWN file, then from target path via /proc/mounts if file doesn't exist), writes WWN to disk if retrieved from target path (for idempotent retry), checks if raw block staging path exists and unmounts accordingly, calls manager.UnStageWithWwn to disconnect the volume by WWN, removes the WWN file, and returns an empty NodeUnstageVolumeResponse on success

#### Scenario: Unstage a SAN volume when WWN retrieval fails
- **WHEN** the CO sends a NodeUnstageVolumeRequest and the device WWN cannot be retrieved (file doesn't exist and target path doesn't contain WWN info)
- **THEN** the driver logs a warning and returns success (idempotent - retry is unlikely to help without WWN)

#### Scenario: Unstage a NAS volume
- **WHEN** the CO sends a NodeUnstageVolumeRequest for a volume on a NAS backend
- **THEN** the driver creates a NasManager; for DTree storage, returns immediately (no unstage needed); for other NAS types, unmounts the staging target path and returns an empty NodeUnstageVolumeResponse

#### Scenario: Unstage a NAS volume when already unmounted
- **WHEN** the CO sends a NodeUnstageVolumeRequest and the staging target path is already unmounted
- **THEN** the driver returns success (idempotent behavior)

#### Scenario: Reject NodeUnstageVolume when manager creation fails
- **WHEN** the CO sends a NodeUnstageVolumeRequest and the manage.NewManager call fails for the backend
- **THEN** the driver returns codes.Internal error with a message indicating the backend and failure reason

#### Scenario: Reject NodeUnstageVolume when unmount fails
- **WHEN** the CO sends a NodeUnstageVolumeRequest and the unmount operation fails
- **THEN** the driver returns codes.Internal error with the unmount failure reason

#### Scenario: Unstage SAN volume with WWN retrieval from target path fallback
- **WHEN** the CO sends a NodeUnstageVolumeRequest and the WWN file does not exist at the expected path
- **THEN** the driver reads /proc/mounts to find the device associated with the staging target path, extracts the WWN from the device path, writes the WWN to disk for idempotent retry, and proceeds with unstage

#### Scenario: Unstage SAN volume with raw block staging path detection
- **WHEN** the CO sends a NodeUnstageVolumeRequest for a block volume
- **THEN** the driver checks if the staging target path exists as a file (raw block bind mount) and unmounts it accordingly before disconnecting the volume
