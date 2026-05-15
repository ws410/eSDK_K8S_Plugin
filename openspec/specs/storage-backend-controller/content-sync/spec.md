## ADDED Requirements

### Requirement: content-sync shall update StorageBackendContent status
The content-sync process shall watch StorageBackendContent resources and update their status based on the backend's actual state (capabilities, capacity, pools, online status).

#### Scenario: Sync Content with backend capabilities
- **WHEN** a StorageBackendContent is created or updated
- **THEN** the controller queries the backend storage array, updates the Content's Capabilities (SupportThin, SupportThick, SupportQoS, SupportMetro, SupportReplication, SupportClone, SupportApplicationType, SupportMetroNAS, SupportNFS3/4/41/42), and updates the Online status

#### Scenario: Sync Content with pool capacities
- **WHEN** a StorageBackendContent is synced
- **THEN** the controller queries all storage pools, updates the Content's Pools array with each pool's name and capacities (FreeCapacity, TotalCapacity, UsedCapacity), and updates the aggregate Capacity map

#### Scenario: Sync Content with device specifications
- **WHEN** a StorageBackendContent is synced
- **THEN** the controller queries the device specifications (LocalDeviceSN, RemoteDevicesSN, VStoreID, VStoreName) and updates the Content's Specification map and SN field

#### Scenario: Sync Content when backend is offline
- **WHEN** the backend storage array is unreachable (wrong password, network issue)
- **THEN** the controller sets the Content's Online status to false and updates the bound SBC's Phase to "Unavailable"

#### Scenario: Sync Content when backend comes online
- **WHEN** a previously offline backend becomes reachable
- **THEN** the controller sets the Content's Online status to true and updates the bound SBC's Phase to "Bound"
