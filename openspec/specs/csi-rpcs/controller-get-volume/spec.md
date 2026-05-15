## ADDED Requirements

### Requirement: ControllerGetVolume RPC is not implemented
The ControllerGetVolume RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests volume information
- **WHEN** the CO sends a ControllerGetVolumeRequest
- **THEN** the driver returns codes.Unimplemented error with an empty message
