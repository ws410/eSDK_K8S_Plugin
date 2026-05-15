## ADDED Requirements

### Requirement: ControllerExpandVolume RPC must expand storage volumes
The ControllerExpandVolume RPC shall expand a volume's capacity on Huawei storage backends. The driver must validate the capacity range, verify expansion support based on storage type and access mode, adjust size to sector alignment, and indicate whether node-side expansion is required.

#### Scenario: Expand a standard LUN/FileSystem volume
- **WHEN** the CO sends a ControllerExpandVolumeRequest with a valid VolumeId and CapacityRange (RequiredBytes, optional LimitBytes)
- **THEN** the driver validates the capacity range (LimitBytes > 0 must be >= RequiredBytes), retrieves the backend and volume name from VolumeId, verifies sector size capacity availability if disableVerifyCapacity is not set in volume attributes, adjusts size to sector alignment using TransVolumeCapacity, calls backend.Plugin.ExpandVolume with volName and adjusted size, and returns ControllerExpandVolumeResponse with CapacityBytes (aligned) and NodeExpansionRequired flag from the plugin result

#### Scenario: Expand a DTree volume
- **WHEN** the CO sends a ControllerExpandVolumeRequest for a volume on a DTree storage backend (identified by constants.IsDtreeStorage)
- **THEN** the driver retrieves the DTree parentName from the volume ID mapping via GetDTreeParentNameByVolumeId, calls backend.Plugin.ExpandDTreeVolume with volName, parentName, and adjusted size, and returns the response with NodeExpansionRequired from the plugin result

#### Scenario: Reject expand with missing capacity range
- **WHEN** the CO sends a ControllerExpandVolumeRequest without a CapacityRange
- **THEN** the driver returns codes.InvalidArgument error with message "no capacity range provided"

#### Scenario: Reject expand with invalid capacity range
- **WHEN** the CO sends a ControllerExpandVolumeRequest where LimitBytes is set and is smaller than RequiredBytes
- **THEN** the driver returns codes.InvalidArgument error with message "limitBytes is smaller than requiredBytes"

#### Scenario: Reject expand of RWX LUN filesystem volume
- **WHEN** the CO sends a ControllerExpandVolumeRequest for a volume with accessMode=MULTI_NODE_MULTI_WRITER, volumeMode=FileSystem, and volumeType=lun
- **THEN** the driver returns codes.InvalidArgument error indicating this combination does not support expansion

#### Scenario: Reject expand of read-only volume
- **WHEN** the CO sends a ControllerExpandVolumeRequest for a volume with accessMode=MULTI_NODE_READER_ONLY
- **THEN** the driver returns codes.InvalidArgument error indicating read-only volumes cannot be expanded

#### Scenario: Expand volume when backend doesn't exist
- **WHEN** the CO sends a ControllerExpandVolumeRequest with a VolumeId referencing a non-existent backend
- **THEN** the driver returns codes.Internal error indicating the backend doesn't exist

#### Scenario: Reject expand with missing volume ID
- **WHEN** the CO sends a ControllerExpandVolumeRequest with an empty VolumeId
- **THEN** the driver returns codes.InvalidArgument error with message "no volume ID provided"

#### Scenario: Expand NAS volume (node expansion not required)
- **WHEN** the CO sends a ControllerExpandVolumeRequest for a NAS storage type
- **THEN** the driver processes the expansion normally but the NAS manager's ExpandVolume returns nil (no node-side expansion needed), and NodeExpansionRequired is set to false in the controller response

#### Scenario: Expand volume with sector size alignment
- **WHEN** the CO sends a ControllerExpandVolumeRequest with RequiredBytes that is not an integer multiple of the sector size
- **THEN** the driver rounds up to the next sector size multiple, verifies the aligned capacity is available (unless disableVerifyCapacity is set), and returns the aligned CapacityBytes in the response

#### Scenario: Expand volume with missing VolumeCapability
- **WHEN** the CO sends a ControllerExpandVolumeRequest for a non-NAS storage type without VolumeCapability in the request
- **THEN** the driver returns codes.InvalidArgument error indicating VolumeCapability is empty

#### Scenario: Skip sector size verification when volume attrs retrieval fails
- **WHEN** the CO sends a ControllerExpandVolumeRequest and GetVolumeAttrsByVolumeId fails to retrieve volume attributes
- **THEN** the verifySectorSize function logs a warning and skips the capacity verification (returns nil), allowing the expansion to proceed

#### Scenario: Skip sector size verification when volume attrs are empty
- **WHEN** the CO sends a ControllerExpandVolumeRequest and the volume attributes list is empty
- **THEN** the verifySectorSize function skips verification (returns nil) as there are no attributes to check

#### Scenario: Skip sector size verification when disableVerifyCapacity conflicts across PVs
- **WHEN** the CO sends a ControllerExpandVolumeRequest and multiple PVs with the same volumeId have conflicting disableVerifyCapacity values
- **THEN** the verifySectorSize function logs a warning about the conflict and skips verification (returns nil)

#### Scenario: Handle SelectBackend returning nil with error for expand
- **WHEN** the CO sends a ControllerExpandVolumeRequest and SelectBackend returns (nil, error)
- **THEN** the driver condition `backend == nil || err != nil` evaluates to true, and returns codes.Internal error "Backend <name> doesn't exist"
