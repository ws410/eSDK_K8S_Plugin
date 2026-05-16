## ADDED Requirements

### Requirement: ControllerUnpublishVolume RPC must detach volumes from nodes
The ControllerUnpublishVolume RPC shall detach (unpublish) a volume from a specific node on the Huawei storage array. If the backend no longer exists, the request returns success with a warning (idempotent behavior). For DTree volumes, the driver must include the parentName in the detach parameters.

#### Scenario: Unpublish volume from a node
- **WHEN** the CO sends a ControllerUnpublishVolumeRequest with VolumeId and NodeId
- **THEN** the driver splits the VolumeId to get backendName and volName, unmarshals the NodeId JSON to extract node parameters, calls backend.Plugin.DetachVolume with the parameters, and returns an empty ControllerUnpublishVolumeResponse on success

#### Scenario: Unpublish a DTree volume from a node
- **WHEN** the CO sends a ControllerUnpublishVolumeRequest for a volume on a DTree storage backend (identified by constants.IsDtreeStorage)
- **THEN** the driver retrieves the DTree parentName from the volume ID mapping via GetDTreeParentNameByVolumeId, adds it to the parameters with key constants.DTreeParentKey, and calls backend.Plugin.DetachVolume with the updated parameters

#### Scenario: Unpublish volume when backend doesn't exist
- **WHEN** the CO sends a ControllerUnpublishVolumeRequest with a VolumeId referencing a non-existent backend
- **THEN** the driver logs a warning, returns success with an empty ControllerUnpublishVolumeResponse, and notes that manual detach from the array is required

#### Scenario: Reject ControllerUnpublishVolume with invalid nodeId JSON
- **WHEN** the CO sends a ControllerUnpublishVolumeRequest with a NodeId that cannot be unmarshaled as JSON
- **THEN** the driver returns codes.Internal error with the unmarshal error message
