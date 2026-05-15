## ADDED Requirements

### Requirement: GetPluginInfo RPC must return driver identity
The GetPluginInfo RPC shall return the CSI driver name and vendor version to the CO (Container Orchestrator). The driver name is configured at startup and the version is derived from the build constants.

#### Scenario: CO requests plugin info
- **WHEN** the CO sends a GetPluginInfoRequest
- **THEN** the driver returns a GetPluginInfoResponse containing the driver name (from app.GetGlobalConfig().DriverName) and vendor version (from build constants)
