## ADDED Requirements

### Requirement: ControllerGetCapabilities RPC must advertise controller capabilities
The ControllerGetCapabilities RPC shall advertise the controller service capabilities supported by the driver. The driver supports: CREATE_DELETE_VOLUME, PUBLISH_UNPUBLISH_VOLUME, EXPAND_VOLUME, CREATE_DELETE_SNAPSHOT, and CLONE_VOLUME.

#### Scenario: CO queries controller capabilities
- **WHEN** the CO sends a ControllerGetCapabilitiesRequest
- **THEN** the driver returns ControllerGetCapabilitiesResponse with capabilities: CREATE_DELETE_VOLUME, PUBLISH_UNPUBLISH_VOLUME, EXPAND_VOLUME, CREATE_DELETE_SNAPSHOT, and CLONE_VOLUME
