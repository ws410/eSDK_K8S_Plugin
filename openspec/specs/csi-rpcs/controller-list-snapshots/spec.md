## ADDED Requirements

### Requirement: ListSnapshots RPC is not implemented
The ListSnapshots RPC is not implemented by this driver. The driver returns codes.Unimplemented for all requests.

#### Scenario: CO requests snapshot listing
- **WHEN** the CO sends a ListSnapshotsRequest
- **THEN** the driver returns codes.Unimplemented error with an empty message
