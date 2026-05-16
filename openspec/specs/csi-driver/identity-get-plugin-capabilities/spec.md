## ADDED Requirements

### Requirement: GetPluginCapabilities RPC must advertise driver capabilities
The GetPluginCapabilities RPC shall advertise that the driver supports the CONTROLLER_SERVICE and VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capabilities, enabling the CO to understand what features the driver provides.

#### Scenario: CO queries plugin capabilities
- **WHEN** the CO sends a GetPluginCapabilitiesRequest
- **THEN** the driver returns capabilities including CONTROLLER_SERVICE (indicating controller service is available) and VOLUME_ACCESSIBILITY_CONSTRAINTS (indicating topology-aware volume placement is supported)
