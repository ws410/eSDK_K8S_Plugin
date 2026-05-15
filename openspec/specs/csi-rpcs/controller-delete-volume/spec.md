## ADDED Requirements

### Requirement: DeleteVolume RPC must delete storage volumes
The DeleteVolume RPC shall delete a volume from Huawei storage backends. The VolumeId format is "backendName.volumeName". The driver must handle different storage types (DTree, OceanStor A-Series NAS, and standard LUN/FileSystem) with appropriate deletion parameters. If the backend no longer exists, the request returns success with a warning (idempotent behavior).

#### Scenario: Delete a standard LUN/FileSystem volume
- **WHEN** the CO sends a DeleteVolumeRequest with a valid VolumeId (format: "backendName.volumeName")
- **THEN** the driver splits the VolumeId, selects the backend, and calls backend.Plugin.DeleteVolume with volName and nil params, returning an empty DeleteVolumeResponse on success

#### Scenario: Delete a DTree volume
- **WHEN** the CO sends a DeleteVolumeRequest for a volume on a DTree storage backend (identified by constants.IsDtreeStorage)
- **THEN** the driver retrieves the DTree parentName from the volume ID mapping via GetDTreeParentNameByVolumeId, and calls backend.Plugin.DeleteDTreeVolume with volName and parentName

#### Scenario: Delete an OceanStor A-Series NAS volume
- **WHEN** the CO sends a DeleteVolumeRequest for a volume on OceanStorASeriesNas storage
- **THEN** the driver retrieves the KvCacheStoreId by volumeId via GetKvCacheStoreIdByVolumeId, constructs delete params with the KvCacheStoreId if present (key: constants.KvCacheStoreId), and calls backend.Plugin.DeleteVolume with volName and the params

#### Scenario: Delete volume when backend doesn't exist
- **WHEN** the CO sends a DeleteVolumeRequest with a VolumeId referencing a backend that no longer exists
- **THEN** the driver logs a warning, returns success (codes.OK) with an empty DeleteVolumeResponse, and notes that manual cleanup from the array is required

#### Scenario: Delete volume when backend selection fails
- **WHEN** the CO sends a DeleteVolumeRequest and the backend selection returns an error (backend exists but selection fails)
- **THEN** the driver treats it the same as backend not found, returns success with empty response, and logs a warning

#### Scenario: Delete volume when backend selector returns error but backend exists
- **WHEN** the CO sends a DeleteVolumeRequest and SelectBackend returns (nil, error) - meaning the backend exists in SBCT but registration/building failed
- **THEN** the driver condition `bk == nil || err != nil` evaluates to true, logs a warning about the backend not existing, and returns success (idempotent behavior)
