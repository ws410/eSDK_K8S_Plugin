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

#### Scenario: Filter DTree pools by parent name availability
- **WHEN** volumeType=dtree and no parentname is specified in StorageClass
- **THEN** the selectDtreePool function filters candidate pools to only those with non-empty GetDTreeParentName(); returns error if no such pools exist

#### Scenario: Skip DTree pool filtering when parentname is specified
- **WHEN** volumeType=dtree and parentname is explicitly set in StorageClass parameters
- **THEN** the selectDtreePool function returns all candidate pools without filtering (parent name validation is handled elsewhere)

#### Scenario: Build pool pairs for local and remote selection
- **WHEN** hyperMetro or replication is enabled
- **THEN** the SelectPoolPair function iterates through each local pool, selects a corresponding remote pool via SelectRemotePool, builds SelectPoolPair structs (Local + Remote), and passes them to WeightPools for final selection

#### Scenario: Select remote pool returns nil when no match found
- **WHEN** SelectRemotePool is called for a hyperMetro or replication volume but no remote pools pass the SecondaryFilterFuncs
- **THEN** the function returns (nil, nil) -- not an error, allowing the local pool to be used without remote pairing
