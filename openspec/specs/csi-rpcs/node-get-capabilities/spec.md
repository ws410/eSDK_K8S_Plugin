## ADDED Requirements

### Requirement: NodeGetCapabilities RPC must advertise node capabilities
The NodeGetCapabilities RPC shall advertise the node service capabilities supported by the driver. The driver supports: STAGE_UNSTAGE_VOLUME, EXPAND_VOLUME, and GET_VOLUME_STATS.

#### Scenario: CO queries node capabilities
- **WHEN** the CO sends a NodeGetCapabilitiesRequest
- **THEN** the driver returns NodeGetCapabilitiesResponse with capabilities: STAGE_UNSTAGE_VOLUME, EXPAND_VOLUME, and GET_VOLUME_STATS
