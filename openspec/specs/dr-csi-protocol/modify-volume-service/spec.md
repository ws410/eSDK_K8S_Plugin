## ADDED Requirements

### Requirement: DR-CSI ModifyVolume service shall modify volume properties (only hyperMetro implemented)
The DR-CSI ModifyVolume gRPC service provides volume modification operations. Currently ONLY hyperMetro conversion (Local2HyperMetro and HyperMetro2Local) is implemented. QoS, SmartTier, and SmartMigration modification types are NOT implemented -- the ModifyVolumeType enum only has Local2HyperMetro and HyperMetro2Local values.

#### Scenario: Modify volume hyperMetro (Local to HyperMetro)
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with MutableParameters containing hyperMetro="true"
- **THEN** the modifyHyperMetro function validates the hyperMetro value, selects the backend, verifies MetroBackend is configured, selects the remote pool, and calls the plugin's ModifyVolume with ModifyVolumeType=Local2HyperMetro; the OceanstorNasPlugin validates both local and remote WWN are non-empty, devices are different, and logic ports are running on own site

#### Scenario: Modify volume hyperMetro (HyperMetro to Local)
- **WHEN** the volume-modify-controller calls ModifyVolume via DR-CSI with MutableParameters containing hyperMetro="false"
- **THEN** the modifyHyperMetro function processes the request with ModifyVolumeType=HyperMetro2Local, converting the volume from HyperMetro active-active to local-only mode

#### Scenario: Reject ModifyVolume with invalid hyperMetro value
- **WHEN** the volume-modify-controller calls ModifyVolume with MutableParameters containing hyperMetro set to a value other than "true" or "false"
- **THEN** the modifyHyperMetro function returns error "hyperMetro value must be \"true\" or \"false\", \"X\" is invalid."

#### Scenario: Reject ModifyVolume when no metro backend configured
- **WHEN** ModifyVolume is called with hyperMetro="true" but the backend's MetroBackend is nil
- **THEN** the service returns error "have not configured hyper metro backend"

#### Scenario: Reject ModifyVolume when remote pool selection fails
- **WHEN** ModifyVolume is called and SelectRemotePool fails (no metro backend, no compatible pools, or hyperMetro+replication conflict)
- **THEN** the service returns error "select remote pool failed, backend name: X, error: Y"

#### Scenario: Modify volume with no-op for unrecognized parameters
- **WHEN** ModifyVolume is called with MutableParameters that do NOT contain the "hyperMetro" key
- **THEN** the modifyHyperMetro function returns (empty response, nil) immediately without performing any modification

#### Scenario: Reject ModifyVolume on unsupported storage types
- **WHEN** ModifyVolume is called on oceandisk-san, oceanstor-a-series-nas, oceanstor-a-series-nas-dme, or oceanstor-dtree backends
- **THEN** the plugin returns an error indicating the storage type does not support volume modify feature
