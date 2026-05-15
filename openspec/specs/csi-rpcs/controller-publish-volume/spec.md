## ADDED Requirements

### Requirement: ControllerPublishVolume RPC must attach volumes to nodes
The ControllerPublishVolume RPC shall attach (publish) a volume to a specific node on the Huawei storage array. The nodeId is a JSON-encoded string containing node information (HostName). The driver must call the backend's AttachVolume operation and return publish context containing the mapping information (ControllerPublishInfo) and filesystem mode. Different protocols return different fields in the publishInfo.

#### Scenario: Publish volume to a node (iSCSI protocol)
- **WHEN** the CO sends a ControllerPublishVolumeRequest with VolumeId, NodeId (JSON-encoded with HostName), and optional VolumeContext
- **THEN** the driver splits the VolumeId to get backendName and volName, unmarshals the NodeId JSON to extract HostName, adds VolumeContext to parameters, calls backend.Plugin.AttachVolume, and the plugin returns mappingInfo containing TgtPortals, TgtIQNs, TgtHostLUNs, and TgtLunWWN; the driver marshals the mappingInfo into publishInfo, determines the filesystem mode via getBackendFilesystemMode, and returns ControllerPublishVolumeResponse with PublishContext containing "publishInfo" (JSON string of ControllerPublishInfo) and "filesystemMode"

#### Scenario: Publish volume to a node (FC protocol)
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on an FC backend
- **THEN** the plugin returns mappingInfo containing TgtLunWWN, TgtWWNs, and TgtHostLUNs in the publishInfo

#### Scenario: Publish volume to a node (FC-NVMe protocol)
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on an FC-NVMe backend
- **THEN** the plugin returns mappingInfo containing PortWWNList (array of PortWWNPair) and TgtLunGuid in the publishInfo

#### Scenario: Publish volume to a node (RoCE/NVMe protocol)
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on a RoCE or NVMe backend
- **THEN** the plugin returns mappingInfo containing TgtPortals and TgtLunGuid in the publishInfo

#### Scenario: Publish volume to a node (SCSI protocol)
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on a SCSI backend
- **THEN** the plugin returns mappingInfo containing TgtLunWWN in the publishInfo

#### Scenario: Publish volume to a node (DTree storage)
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on a DTree storage backend
- **THEN** the plugin returns mappingInfo containing DTreeParentName, which is wrapped in DTreePublishInfo and passed through PublishContext

#### Scenario: Publish volume when backend doesn't exist
- **WHEN** the CO sends a ControllerPublishVolumeRequest with a VolumeId referencing a non-existent backend
- **THEN** the driver returns codes.Internal error indicating the backend doesn't exist

#### Scenario: Publish volume with NfsPlus protocol
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on a backend with protocol=nfs-plus (and not DTree storage)
- **THEN** the driver queries the volume from the backend to retrieve its filesystem mode (local or HyperMetro) and includes it in the PublishContext

#### Scenario: Reject ControllerPublishVolume with invalid nodeId JSON
- **WHEN** the CO sends a ControllerPublishVolumeRequest with a NodeId that cannot be unmarshaled as JSON
- **THEN** the driver returns codes.Internal error with the unmarshal error message
