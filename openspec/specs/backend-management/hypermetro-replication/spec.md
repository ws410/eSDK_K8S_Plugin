## ADDED Requirements

### Requirement: hypermetro-replication shall manage active-active and remote replication
The hyperMetro-replication system shall manage hyperMetro (active-active) and replication (remote replication) configurations for storage pools, including remote pool selection and pair management.

#### Scenario: Select remote pool for hyperMetro
- **WHEN** hyperMetro=true is specified in CreateVolume
- **THEN** the SelectRemotePool function loads the metroBackend from cache, filters its pools using SecondaryFilterFuncs, and selects the optimal remote pool by free capacity

#### Scenario: Select remote pool for replication
- **WHEN** replication=true is specified in CreateVolume
- **THEN** the SelectRemotePool function loads the replicaBackend from cache, filters its pools, and selects the optimal remote pool

#### Scenario: Reject hyperMetro without metroBackend
- **WHEN** hyperMetro=true but the backend has no metroBackend configured
- **THEN** the system returns an error "no metro backend exists for volume"

#### Scenario: Reject replication without replicaBackend
- **WHEN** replication=true but the backend has no replicaBackend configured
- **THEN** the system returns an error "no replica backend exists for volume"

#### Scenario: Configure metro domain and vStore pair
- **WHEN** a volume is created with hyperMetro
- **THEN** the processCreateVolumeParametersAfterSelect function sets metroDomain, metrovStorePairID, and remoteStoragePool in the parameters for the plugin

#### Scenario: Reject hyperMetro with incomplete configuration
- **WHEN** a backend has metroBackend set but missing hyperMetroDomain or metrovStorePairID, OR has hyperMetroDomain/metrovStorePairID set but missing metroBackend
- **THEN** the NewBackend function returns error "hyperMetro configuration in backend <name> is incorrect"

#### Scenario: SelectRemotePool rejects hyperMetro with nil MetroBackend
- **WHEN** SelectRemotePool is called with hyperMetro=true but the local backend's MetroBackend field is nil
- **THEN** the function returns error "no metro backend of <name> exists for volume: <params>"
