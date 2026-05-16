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

#### Scenario: Auto-attach when publishInfo is missing (backend offline)
- **WHEN** the CO sends a NodeStageVolumeRequest without publishInfo in PublishContext and the StorageBackendContent status is nil or Online=false
- **THEN** the attachVolume function returns error "attach volume failed cause backend offline, backend name: <name>" and the driver returns codes.Internal

#### Scenario: Auto-attach when publishInfo is missing (build backend failed)
- **WHEN** the CO sends a NodeStageVolumeRequest without publishInfo and backend.BuildBackend fails (invalid config, missing credentials, plugin init failure)
- **THEN** the attachVolume function returns error "attach volume failed while building backend, backend name: <name>, err: <error>" and the driver returns codes.Internal

#### Scenario: Auto-attach when publishInfo is missing (hostname retrieval failed)
- **WHEN** the CO sends a NodeStageVolumeRequest without publishInfo and utils.GetHostName fails
- **THEN** the attachVolume function returns error "attach volume failed while getting hostname, err: <error>" and the driver returns codes.Internal

#### Scenario: Auto-attach when publishInfo is missing (attach to array failed)
- **WHEN** the CO sends a NodeStageVolumeRequest without publishInfo and buildBackend.Plugin.AttachVolume fails
- **THEN** the attachVolume function returns error "attach volume failed while attaching volume, volume name: <name>, err: <error>" and the driver returns codes.Internal

#### Scenario: Clear residual path with LUN ID for UltraPath only
- **WHEN** the SanManager StageVolume runs clearResidualPathWithLunId task
- **THEN** the function checks if VolumeUseMultiPath=true AND MultiPathType=HWUltraPath AND protocol is "iscsi" or "fc"; if all conditions met, it calls connector.CleanDeviceByLunId to clean stale devices by LUN ID before connecting

#### Scenario: Stage volume with NVMe CLI version check
- **WHEN** the CO sends a NodeStageVolumeRequest for a volume on an NVMe protocol backend
- **THEN** the NVMe connector validates that the installed nvme-cli version is >= 1.9 before proceeding with connection; if the version is too old, returns an error

#### Scenario: Stage SAN filesystem volume with XFS nouuid mount option
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume with fsType=xfs
- **THEN** the mountDisk function automatically adds "nouuid" to the mount options to allow cloned volumes with the same UUID to be mounted on the same node

#### Scenario: Stage block volume with legacy symlink handling
- **WHEN** the CO sends a NodeStageVolumeRequest for a block volume and the staging target path is a symlink (legacy pre-V4.6.0 format)
- **THEN** the BindMountRawBlockDevice function removes the symlink and recreates the target path as a regular file before performing the bind mount

#### Scenario: Stage SAN filesystem volume with disk size-based mkfs strategy
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume on an unformatted block device
- **THEN** the getDiskSizeType function determines the mkfs template based on device size: <=0.5TiB="default", 0.5-1TiB="big", 1-10TiB="huge", 10-100TiB="large", 100-512TiB="veryLarge", >512TiB returns error; the formatDisk function applies the corresponding mkfs template (e.g., "-T big" for ext filesystems)

#### Scenario: Reject stage volume when disk size exceeds maximum
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume on an unformatted block device larger than 512TiB
- **THEN** the getDiskSizeType function returns an error "the disk size does not support"

#### Scenario: Stage SAN filesystem volume with concurrent formatting detection
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume and another process is already formatting the device
- **THEN** the formatDisk function detects "in use by the system" in the mkfs output, sleeps 10 seconds, and returns an error

#### Scenario: Stage SAN filesystem volume with partition device skipping
- **WHEN** the clearResidualPathWithWwn task scans /dev/disk/by-id/ for device paths
- **THEN** partition devices (e.g., sdc1, nvme0n1p1, dm-1 with trailing digits) are explicitly skipped during residual path detection

#### Scenario: Stage SAN filesystem volume with blkid exit code 2 handling
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume and blkid returns exit code 2
- **THEN** the getFSType function calls connector.IsDeviceFormatted to verify if the device is actually formatted; if formatted but blkid fails, returns an ambiguous error

#### Scenario: Stage SAN filesystem volume with resize after mount
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume on an already-formatted device with accessMode that is not MULTI_NODE_MULTI_WRITER or MULTI_NODE_READER_ONLY
- **THEN** after mounting, the connector.ResizeMountPath function is called to resize the filesystem (resize2fs for ext*, xfs_growfs for xfs)

#### Scenario: Stage SAN filesystem volume skipping resize for multi-node access
- **WHEN** the CO sends a NodeStageVolumeRequest for a SAN filesystem volume with accessMode=MULTI_NODE_MULTI_WRITER or MULTI_NODE_READER_ONLY
- **THEN** the resize step is skipped after mounting to prevent conflicts with other nodes

#### Scenario: Stage volume with DPC/DTFS protocol mount option
- **WHEN** the CO sends a NodeStageVolumeRequest for a NAS volume with protocol=dpc or protocol=dtfs
- **THEN** the parseNFSInfo function sets mntDashT to the appropriate protocol type for the mount command
