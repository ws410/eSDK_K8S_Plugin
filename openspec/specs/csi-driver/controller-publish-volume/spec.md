## ADDED Requirements

### Requirement: ControllerPublishVolume RPC must attach volumes to nodes
The ControllerPublishVolume RPC shall attach (publish) a volume to a specific node on the Huawei storage array. The nodeId is a JSON-encoded string containing node information (HostName). The driver must call the backend's AttachVolume operation and return publish context containing the mapping information (ControllerPublishInfo) and filesystem mode.

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

#### Scenario: Publish volume with HyperMetro dual-site mapping info merge
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a volume on a HyperMetro-enabled OceanStor SAN backend with both local and remote storage online
- **THEN** the metroHandler calls MetroAttacher.ControllerAttach which merges mapping info from both sites: for iSCSI, appends tgtPortals, tgtIQNs, and tgtHostLUNs arrays; for FC, appends tgtWWNs and tgtHostLUNs arrays, returning combined mapping info for multi-path access

#### Scenario: Publish volume with HyperMetro local-only fallback
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a HyperMetro volume where only the local storage is online (remote is offline)
- **THEN** the handler method falls back to commonHandler using the local client, returning only local-site mapping info

#### Scenario: Publish volume with HyperMetro remote-only fallback
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a HyperMetro volume where only the remote storage is online (local is offline)
- **THEN** the handler method falls back to commonHandler using the remote client, returning only remote-site mapping info

#### Scenario: Reject ControllerPublishVolume when initiator conflicts with another host
- **WHEN** the CO sends a ControllerPublishVolumeRequest and the backend plugin detects the initiator (iSCSI IQN or FC WWPN) is already associated with a different host on the storage array
- **THEN** the plugin returns an error indicating the initiator is already in use by another host

#### Scenario: Publish volume with ALUA configuration update
- **WHEN** the CO sends a ControllerPublishVolumeRequest and the attacher detects the host or initiator ALUA configuration differs from the current storage array configuration
- **THEN** the attacher updates the initiator ALUA settings and/or host ALUA settings before completing the attach operation

#### Scenario: Publish volume with host group and mapping group creation
- **WHEN** the CO sends a ControllerPublishVolumeRequest and the host does not yet exist on the storage array
- **THEN** the attacher creates the host, adds initiators, creates the host group, creates the mapping between the LUN and host group, and creates the namespace group (for Oceandisk), returning the mapping info

#### Scenario: Publish volume with NFS auto-auth client CIDR filtering
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a DTree or NAS volume with nfsAutoAuthClient enabled
- **THEN** the getFilteredIPs function retrieves the node's host IPs from the Secret, filters them by the configured CIDRs, and adds only matching IPs as authorized NFS clients with ReadWrite access

#### Scenario: Reject ControllerPublishVolume when both local and remote storage are offline
- **WHEN** the CO sends a ControllerPublishVolumeRequest for a HyperMetro volume and both local and remote storage are offline
- **THEN** the getLunInfo function returns an error "both local and remote storage not online"
