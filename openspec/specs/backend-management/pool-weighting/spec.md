## ADDED Requirements

### Requirement: pool-weighting shall select the optimal storage pool by free capacity
The pool weighting system shall select the optimal storage pool from filtered candidates based on free capacity, with support for local/remote pool pair selection for hyperMetro and replication scenarios.

#### Scenario: Select pool by maximum free capacity
- **WHEN** multiple pools pass all filter stages
- **THEN** the weightByFreeCapacity function selects the pool with the highest FreeCapacity

#### Scenario: Select local and remote pool pair
- **WHEN** hyperMetro or replication is enabled
- **THEN** the WeightPools function first selects the local pool by free capacity, then finds the corresponding remote pool pair from the metroBackend or replicaBackend

#### Scenario: Select remote pool independently
- **WHEN** only remote pool selection is needed
- **THEN** the SelectRemotePool function filters remote pools using SecondaryFilterFuncs (volumeType, allocType, qos, replication, applicationType) and selects by free capacity

#### Scenario: Reject hyperMetro and replication together
- **WHEN** both hyperMetro=true and replication=true are specified
- **THEN** the system returns an error "cannot create volume with hyperMetro and replication properties"

#### Scenario: Decrement free capacity for thick allocation
- **WHEN** a thick volume is created
- **THEN** the updateSelectPool function decrements the selected pool's FreeCapacity by the requestSize to prevent over-allocation
