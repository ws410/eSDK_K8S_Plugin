## ADDED Requirements

### Requirement: content-status-update shall update SBCT status fields via DR-CSI provider
The sidecar controller shall update StorageBackendContent status fields by polling the DR-CSI provider's GetBackendStats gRPC service. The Online status comes from the provider response (which in turn gets it from storage plugin clients), not from direct login attempts. The aggregate Capacity map is NEVER populated -- only the Pools array is updated.

#### Scenario: Update Online status from provider response
- **WHEN** the sidecar controller's getContentTask calls GetBackendStats via the DR-CSI provider
- **THEN** the provider checks IsSBCTOnline before fetching stats; if online, the response includes online=true; the shouldUpdateContentStatus function compares the response's Online field with the current SBCT status and updates if changed

#### Scenario: Update Online status to false when backend is unreachable
- **WHEN** a storage plugin client (OceanStor, FusionStorage, etc.) fails to communicate with the storage array
- **THEN** the plugin calls SetStorageBackendContentOnlineStatus to set SBCT.Status.Online=false and publishes a BackendStatus event; this is handled by the storage plugin, NOT by the sidecar controller directly

#### Scenario: Update pool capacities (no aggregate Capacity)
- **WHEN** the sidecar controller receives GetBackendStatsResponse with pool data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Pools with each pool's name and capacities (FreeCapacity, TotalCapacity, UsedCapacity); the aggregate SBCT.Status.Capacity map (TotalCapacity/UsedCapacity/FreeCapacity) is NEVER populated by either the sidecar or the main controller

#### Scenario: Update backend capabilities
- **WHEN** the sidecar controller receives GetBackendStatsResponse with capability data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Capabilities via DeepEqual comparison with: SupportThin, SupportThick, SupportQoS, SupportMetro, SupportReplication, SupportClone, SupportApplicationType, SupportMetroNAS, and NFS protocol support flags

#### Scenario: Update device specifications
- **WHEN** the sidecar controller receives GetBackendStatsResponse with specification data
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.Specification with LocalDeviceSN, RemoteDevicesSN, VStoreID, VStoreName, and updates SBCT.Status.SN with the device serial number

#### Scenario: Update vendor and version
- **WHEN** the sidecar controller receives GetBackendStatsResponse
- **THEN** the shouldUpdateContentStatus function updates SBCT.Status.VendorName (e.g., "Huawei") and SBCT.Status.ProviderVersion (CSI driver version)

#### Scenario: shouldUpdateContentStatus always triggers API update
- **WHEN** the sidecar controller processes GetBackendStatsResponse and calls shouldUpdateContentStatus
- **THEN** the function always returns true, even when no fields have changed, triggering an unnecessary Kubernetes API update call on every sync cycle
