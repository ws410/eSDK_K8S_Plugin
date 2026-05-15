## ADDED Requirements

### Requirement: NodeStageVolume RPC must stage volumes on the node
The NodeStageVolume RPC shall prepare a volume for use on the node by staging it to a staging target path. The driver creates a manage.Manager (SanManager or NasManager based on backend protocol) and delegates the staging operation to it. SAN staging involves connecting the volume to the host via the appropriate protocol connector (iSCSI, FC, FC-NVMe, RoCE, NVMe, SCSI), while NAS staging involves mounting the NFS/NFS+/DPC/DTFS share.

#### Scenario: Stage a SAN volume (iSCSI/FC/FC-NVMe/RoCE/NVMe/SCSI)
- **WHEN** the CO sends a NodeStageVolumeRequest with VolumeId, StagingTargetPath, VolumeCapability, PublishContext (containing publishInfo), and VolumeContext
- **THEN** the driver splits the VolumeId to get backendName, creates a SanManager for the backend's protocol, builds parameters using WithProtocol, WithConnector, WithVolumeCapability, WithControllerPublishInfo, and WithMultiPathType, executes a task flow: clearResidualPathWithWwn → clearResidualPathWithLunId → connectVolume → (stageForBlock or stageForMount based on volumeMode) → saveWwnToDisk, and returns an empty NodeStageVolumeResponse on success

#### Scenario: Stage a SAN block volume
- **WHEN** the CO sends a NodeStageVolumeRequest with VolumeCapability.GetBlock() != nil
- **THEN** the driver sets volumeMode="Block" and stagingPath=StagingTargetPath + "/" + VolumeId in parameters, executes the stageForBlock task which creates the block device at the staging path, and saves the WWN to disk

#### Scenario: Stage a SAN filesystem volume
- **WHEN** the CO sends a NodeStageVolumeRequest with VolumeCapability.GetMount() != nil
- **THEN** the driver extracts fsType from the mount capability, validates fsType (must be ext2, ext3, ext4, or xfs if specified), extracts mountFlags, determines accessMode (ReadOnly adds "ro" flag), sets targetPath=StagingTargetPath, fsType, mountFlags, and accessMode in parameters, executes the stageForMount task which creates a filesystem and mounts it, and saves the WWN to disk

#### Scenario: Stage a NAS volume (NFS/NFS+/DPC/DTFS)
- **WHEN** the CO sends a NodeStageVolumeRequest for a volume on a NAS backend (protocol: nfs, nfs+, dpc, dtfs)
- **THEN** the driver creates a NasManager, builds parameters using WithProtocol, WithPortals, WithVolumeCapability, and WithDeviceWWN; for DTree storage, staging is skipped (returns immediately); for other NAS types, generates sourcePath from protocol and portals (e.g., "portal:/" for NFS, "/" for DPC/DTFS), concatenates with volumeName, and mounts the share to the staging target path

#### Scenario: Stage a NAS volume with HyperMetro filesystem mode
- **WHEN** the CO sends a NodeStageVolumeRequest for a NAS volume with protocol=nfs+ and filesystemMode=HyperMetro in PublishContext
- **THEN** the driver combines local portals and metro portals into a single portals list for the mount operation

#### Scenario: Stage a DTFS volume with device WWN
- **WHEN** the CO sends a NodeStageVolumeRequest for a volume on DTFS protocol with a deviceWWN
- **THEN** the driver adds "cid=deviceWWN" to the mountFlags for the mount operation

#### Scenario: Reject NodeStageVolume when manager creation fails
- **WHEN** the CO sends a NodeStageVolumeRequest and the manage.NewManager call fails for the backend (e.g., unsupported protocol)
- **THEN** the driver returns codes.Internal error with a message indicating the backend and failure reason

#### Scenario: Reject NodeStageVolume when publishInfo is missing and auto-attach fails
- **WHEN** the CO sends a NodeStageVolumeRequest without publishInfo in PublishContext and the auto-attach operation fails (backend offline, build backend failed, hostname retrieval failed, attach failed, or marshal failed)
- **THEN** the driver returns codes.Internal error with the specific failure reason

#### Scenario: Reject NodeStageVolume with invalid fsType
- **WHEN** the CO sends a NodeStageVolumeRequest with fsType that is not one of ext2, ext3, ext4, or xfs (for mount type)
- **THEN** the driver returns codes.Internal error with the list of supported file types

#### Scenario: Reject NodeStageVolume with invalid volume capability
- **WHEN** the CO sends a NodeStageVolumeRequest with VolumeCapability that is neither Block nor Mount type
- **THEN** the driver returns codes.Internal error indicating invalid volume capability
