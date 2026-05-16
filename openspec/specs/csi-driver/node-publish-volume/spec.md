## ADDED Requirements

### Requirement: NodePublishVolume RPC must publish volumes to the node target path
The NodePublishVolume RPC shall make a staged volume available at the target path on the node. The driver supports both block volume mode (bind mount from staged device) and filesystem volume mode (bind mount from staging path or NFS mount for DTree volumes).

#### Scenario: Publish a block volume to the node
- **WHEN** the CO sends a NodePublishVolumeRequest with VolumeCapability.GetBlock() != nil
- **THEN** the driver sets sourcePath=StagingTargetPath + "/" + VolumeId, calls manage.PublishBlock which performs a bind mount of the raw block device from sourcePath to TargetPath with the mountFlags from VolumeCapability.GetMount().GetMountFlags(), and returns an empty NodePublishVolumeResponse on success

#### Scenario: Publish a filesystem volume (non-DTree)
- **WHEN** the CO sends a NodePublishVolumeRequest with VolumeCapability.GetBlock() == nil for a non-DTree backend
- **THEN** the driver retrieves the backend config (storage, protocol, portals, metroPortals), sets sourcePath=StagingTargetPath, constructs mount options (bind, plus "ro" if Readonly is true), creates connectInfo with srcType=MountFSType, sourcePath, targetPath, mountFlags, protocol, and portals, calls manage.Mount which uses the appropriate connector (NFS or NFS+), and returns an empty NodePublishVolumeResponse

#### Scenario: Publish a DTree filesystem volume
- **WHEN** the CO sends a NodePublishVolumeRequest for a volume on a DTree storage backend
- **THEN** the driver retrieves the backend config, determines the DTree sourcePath from parentName (from PublishContext.publishInfo DTreeParentName if available, otherwise from backend config), generates path prefix by protocol (portal:/" for NFS/NFS+, "/" for DPC/DTFS), constructs sourcePath as prefix + parentName + "/" + volumeName, determines mount options (mountFlags from VolumeCapability, "ro" for ReadOnly access, "cid=deviceWWN" for DTFS on A-Series with deviceWWN), and mounts the DTree share to the target path

#### Scenario: Publish a DTree volume when parentName is missing
- **WHEN** the CO sends a NodePublishVolumeRequest for a DTree volume and the parentName cannot be determined (not in PublishContext and not in backend config)
- **THEN** the driver returns codes.Internal error indicating parentName is missing and suggesting that attachRequired parameter should be enabled

#### Scenario: Reject NodePublishVolume when backend config retrieval fails
- **WHEN** the CO sends a NodePublishVolumeRequest and GetBackendConfig fails (missing parameters, protocol, portals, or storage in backend configmap)
- **THEN** the driver returns codes.Internal error with the specific failure reason

#### Scenario: Reject NodePublishVolume when mount fails
- **WHEN** the CO sends a NodePublishVolumeRequest and the mount operation fails
- **THEN** the driver returns codes.Internal error with the mount failure reason
