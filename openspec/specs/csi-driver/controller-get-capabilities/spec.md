## ADDED Requirements

### Requirement: ControllerGetCapabilities RPC must advertise controller capabilities
The ControllerGetCapabilities RPC shall advertise the controller service capabilities supported by the driver. The driver supports: CREATE_DELETE_VOLUME, PUBLISH_UNPUBLISH_VOLUME, EXPAND_VOLUME, CREATE_DELETE_SNAPSHOT, and CLONE_VOLUME.

#### Scenario: CO queries controller capabilities
- **WHEN** the CO sends a ControllerGetCapabilitiesRequest
- **THEN** the driver returns ControllerGetCapabilitiesResponse with capabilities: CREATE_DELETE_VOLUME, PUBLISH_UNPUBLISH_VOLUME, EXPAND_VOLUME, CREATE_DELETE_SNAPSHOT, and CLONE_VOLUME

---

### Requirement: Unimplemented Controller RPCs
The following Controller RPCs are not implemented by this driver and return codes.Unimplemented for all requests: ListVolumes, ControllerGetVolume, GetCapacity, ValidateVolumeCapabilities.

#### Scenario: CO requests volume listing
- **WHEN** the CO sends a ListVolumesRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"

#### Scenario: CO requests volume information
- **WHEN** the CO sends a ControllerGetVolumeRequest
- **THEN** the driver returns codes.Unimplemented error with an empty message

#### Scenario: CO requests storage capacity information
- **WHEN** the CO sends a GetCapacityRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"

#### Scenario: CO requests volume capability validation
- **WHEN** the CO sends a ValidateVolumeCapabilitiesRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"
