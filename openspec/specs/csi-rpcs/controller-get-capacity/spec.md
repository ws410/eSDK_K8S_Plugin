## ADDED Requirements

### Requirement: GetCapacity RPC is not implemented
The GetCapacity RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests storage capacity information
- **WHEN** the CO sends a GetCapacityRequest
- **THEN** the driver returns codes.Unimplemented error with message "Not implemented"
