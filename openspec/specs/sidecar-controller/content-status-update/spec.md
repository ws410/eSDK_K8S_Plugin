## ADDED Requirements

### Requirement: content-status-update shall update SBCT status fields
The sidecar controller shall update StorageBackendContent status fields including Online, Capacity, Capabilities, Pools, Specification, SN, VendorName, and ProviderVersion.

#### Scenario: Update Online status
- **WHEN** the backend login succeeds
- **THEN** the controller sets SBCT.Status.Online = true
- **WHEN** the backend login fails
- **THEN** the controller sets SBCT.Status.Online = false

#### Scenario: Update pool capacities
- **WHEN** the sidecar polls backend pool information
- **THEN** the controller updates SBCT.Status.Pools with each pool's name and capacities (FreeCapacity, TotalCapacity, UsedCapacity), and updates SBCT.Status.Capacity with aggregate values

#### Scenario: Update backend capabilities
- **WHEN** the sidecar polls backend license features
- **THEN** the controller updates SBCT.Status.Capabilities with: SupportThin, SupportThick, SupportQoS, SupportMetro, SupportReplication, SupportClone, SupportApplicationType, SupportMetroNAS, and NFS protocol support flags

#### Scenario: Update device specifications
- **WHEN** the sidecar polls device information
- **THEN** the controller updates SBCT.Status.Specification with LocalDeviceSN, RemoteDevicesSN, VStoreID, VStoreName, and updates SBCT.Status.SN with the device serial number

#### Scenario: Update vendor and version
- **WHEN** the sidecar polls provider information
- **THEN** the controller updates SBCT.Status.VendorName (e.g., "Huawei") and SBCT.Status.ProviderVersion (CSI driver version)
