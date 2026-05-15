## ADDED Requirements

### Requirement: NodeExpandVolume RPC must expand volumes on the node
The NodeExpandVolume RPC shall perform node-side volume expansion after the controller has expanded the volume on the storage array. The driver creates a manage.Manager for the backend and delegates the expansion. For SAN volumes, this involves resizing the block device and optionally the filesystem. For NAS volumes, no expansion is needed (NodeExpansionRequired is always false for NAS).

#### Scenario: Expand a SAN block volume on the node
- **WHEN** the CO sends a NodeExpandVolumeRequest with VolumeId, VolumePath, CapacityRange (RequiredBytes > 0), and StagingTargetPath
- **THEN** the driver validates CapacityRange (RequiredBytes > 0) and VolumePath (non-empty), splits the VolumeId to get backendName, creates a SanManager, retrieves the device WWN from the staging path (without checking device reference or saving to disk), calls connector.ResizeBlock with WWN and RequiredBytes to resize the block device, and returns an empty NodeExpandVolumeResponse on success

#### Scenario: Expand a SAN filesystem volume on the node
- **WHEN** the CO sends a NodeExpandVolumeRequest with VolumeCapability.GetMount() != nil
- **THEN** the driver performs the same block device resize as block volumes, then additionally calls connector.ResizeMountPath with VolumePath to resize the filesystem, and returns an empty NodeExpandVolumeResponse

#### Scenario: Expand a NAS volume on the node
- **WHEN** the CO sends a NodeExpandVolumeRequest for a volume on a NAS backend
- **THEN** the driver creates a NasManager, and the NasManager.ExpandVolume returns nil immediately (no node-side expansion needed for NAS volumes)

#### Scenario: Reject NodeExpandVolume with missing capacity range
- **WHEN** the CO sends a NodeExpandVolumeRequest with missing or invalid CapacityRange (RequiredBytes <= 0)
- **THEN** the driver returns codes.Internal error with message "NodeExpandVolume CapacityRange must be provided"

#### Scenario: Reject NodeExpandVolume with missing volume path
- **WHEN** the CO sends a NodeExpandVolumeRequest with an empty VolumePath
- **THEN** the driver returns codes.Internal error with message "NodeExpandVolume volumePath must be provided"

#### Scenario: Reject NodeExpandVolume when manager creation fails
- **WHEN** the CO sends a NodeExpandVolumeRequest and the manage.NewManager call fails for the backend
- **THEN** the driver returns codes.Internal error with a message indicating the backend and failure reason

#### Scenario: Reject NodeExpandVolume when block resize fails
- **WHEN** the CO sends a NodeExpandVolumeRequest and connector.ResizeBlock fails
- **THEN** the driver returns codes.Internal error with the resize failure reason

#### Scenario: Reject NodeExpandVolume when filesystem resize fails
- **WHEN** the CO sends a NodeExpandVolumeRequest for a filesystem volume and connector.ResizeMountPath fails
- **THEN** the driver returns codes.Internal error with the mount path resize failure reason
