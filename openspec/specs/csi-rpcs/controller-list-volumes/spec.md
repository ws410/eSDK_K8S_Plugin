## ADDED Requirements

### Requirement: ListVolumes RPC is not implemented
The ListVolumes RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests volume listing
- **WHEN** the CO sends a ListVolumesRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"
