## ADDED Requirements

### Requirement: ValidateVolumeCapabilities RPC is not implemented
The ValidateVolumeCapabilities RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests volume capability validation
- **WHEN** the CO sends a ValidateVolumeCapabilitiesRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"
