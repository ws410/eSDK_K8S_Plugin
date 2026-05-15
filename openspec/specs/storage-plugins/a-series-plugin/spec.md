## ADDED Requirements

### Requirement: A-Series plugin shall manage OceanStor A-Series NAS and DTree
The A-Series plugin (OceanstorASeriesPlugin) shall implement the StoragePlugin interface for oceanstor-a-series-nas storage type, supporting NFS and DTFS protocols.

#### Scenario: Initialize A-Series plugin
- **WHEN** the plugin is initialized
- **THEN** it validates protocol is "nfs" or "dtfs", verifies portals (required for NFS with exactly 1 portal; not required for DTFS), creates an A-Series client, logs in, sets system info, and optionally logs out based on keepLogin flag

#### Scenario: Create A-Series NAS volume
- **WHEN** CreateVolume is called for oceanstor-a-series-nas
- **THEN** the plugin resolves the volume name from PV name or parameters, converts parameters to CreateASeriesVolumeParameter struct, generates the volume model with protocol, and creates the filesystem/DTree on the A-Series storage

#### Scenario: Query A-Series volume with workload type
- **WHEN** QueryVolume is called
- **THEN** the plugin extracts applicationType from parameters as workloadType and queries the filesystem through the A-Series API

#### Scenario: Delete A-Series NAS volume with KvCacheStoreId
- **WHEN** DeleteVolume is called for an A-Series NAS volume
- **THEN** the plugin extracts KvCacheStoreId from params (if present) and includes it in the delete model for the A-Series API

#### Scenario: Expand A-Series volume
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the filesystem capacity through the A-Series API and returns NodeExpansionRequired=false

#### Scenario: Handle A-Series specific protocols
- **WHEN** the plugin is initialized
- **THEN** it validates the protocol is either "nfs" or "dtfs" for A-Series storage

#### Scenario: A-Series capabilities from license
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin queries license features and sets: SupportThin=true, SupportApplicationType=true, SupportQoS based on SmartQoS feature, SupportThick=false, SupportMetro=false, SupportReplication=false, SupportClone=false; also queries NFS service settings to update NFS protocol capabilities (SupportNFS3/4/41/42)

#### Scenario: A-Series specifications include WWN
- **WHEN** getBackendSpecifications is called
- **THEN** the plugin returns LocalDeviceSN, VStoreID, VStoreName, and DeviceWWN

#### Scenario: Update pool capabilities with vStore quota
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin queries vStore capacity (if vStoreName is set), parses nasCapacityQuota and nasFreeCapacityQuota, and combines with pool data; if quota is not set (nasCapacityQuota=0 or nasFreeCapacityQuota=-1), returns empty capacity map

#### Scenario: Reject snapshot operations on A-Series
- **WHEN** CreateSnapshot or DeleteSnapshot is called
- **THEN** the plugin returns error "oceanstor-a-series-nas storage does not support snapshot feature"

#### Scenario: Reject DTree operations on A-Series NAS
- **WHEN** DeleteDTreeVolume or ExpandDTreeVolume is called
- **THEN** the plugin returns error "oceanstor-a-series-nas storage does not support DTree feature"

#### Scenario: Reject ModifyVolume on A-Series
- **WHEN** ModifyVolume is called
- **THEN** the plugin returns error "oceanstor-a-series-nas storage does not support volume modify feature"

#### Scenario: SupportQoSParameters validates range for A-Series
- **WHEN** SupportQoSParameters is called
- **THEN** the plugin calls smartx.CheckQoSParametersValueRange to validate QoS parameter values

#### Scenario: Validate A-Series parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies parameters, protocol, portals, creates a test client, performs ValidateLogin, and logs out
