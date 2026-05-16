## ADDED Requirements

### Requirement: NFS connector shall mount and unmount NFS shares
The NFS connector shall implement the VolumeConnector interface to mount/unmount NFS filesystem shares on Kubernetes nodes. It supports both block device mounting (srcType=block) and filesystem mounting (srcType=fs).

#### Scenario: Mount NFS share (filesystem type)
- **WHEN** the connector receives a ConnectVolume request with srcType=fs, sourcePath, targetPath, and protocol
- **THEN** the connector parses the connection info (defaulting fsType to ext4 if not set), determines mount type based on protocol (dpc for DPC, dtfs for DTFS), and calls mountFS which uses MountToDir to mount the NFS share

#### Scenario: Mount block device with formatting
- **WHEN** the connector receives a ConnectVolume request with srcType=block
- **THEN** the connector reads the device, checks if it has a filesystem using blkid; if unformatted, checks if another process is formatting it, determines disk size type (default/big/huge/large/veryLarge based on size thresholds: 0.5TiB/1TiB/10TiB/100TiB/512TiB), formats the disk with the appropriate mkfs options, and mounts it

#### Scenario: Mount block device with existing filesystem
- **WHEN** the connector receives a ConnectVolume request with srcType=block and the device already has a filesystem
- **THEN** the connector mounts the device and, if accessMode is not MULTI_NODE_MULTI_WRITER or MULTI_NODE_READER_ONLY, calls ResizeMountPath to expand the filesystem

#### Scenario: Unmount NFS share
- **WHEN** the connector receives a DisConnectVolume request with the targetPath
- **THEN** the connector unmounts the target path using Unmount and removes the target directory

#### Scenario: Handle NFS mount failure
- **WHEN** the NFS server is unreachable or the share doesn't exist
- **THEN** the connector returns an error with the mount failure reason

#### Scenario: Mount NFS with read-only option
- **WHEN** the mountFlags include "ro"
- **THEN** the connector mounts the NFS share as read-only

#### Scenario: Reject unsupported source type
- **WHEN** the connector receives a ConnectVolume request with srcType other than "block" or "fs"
- **THEN** the connector returns error "not support source type"

#### Scenario: Reject mount with missing source path
- **WHEN** the connector receives a ConnectVolume request without sourcePath
- **THEN** the connector returns error "there are no source path in the connection info"

#### Scenario: Reject mount with missing target path
- **WHEN** the connector receives a ConnectVolume request without targetPath
- **THEN** the connector returns error "there are no target path in the connection info"

#### Scenario: Mount block device with formatting details
- **WHEN** the connector mounts a block device (srcType=block)
- **THEN** it performs the following:
  - Reads the device and checks if it has a filesystem using blkid; if unformatted, checks if another process is formatting it (waits 10 seconds and returns error if so)
  - Determines disk size type (default/big/huge/large/veryLarge based on size thresholds: 0.5TiB/1TiB/10TiB/100TiB/512TiB); returns error if size exceeds 512TiB
  - Formats the disk with the appropriate mkfs options and mounts it
  - If fsType=xfs and source is a /dev/* device, automatically adds "nouuid" to mount options to allow cloned volumes with same UUID to be mounted

#### Scenario: Mount NFS share with protocol-specific options
- **WHEN** the connector mounts an NFS share (srcType=fs)
- **THEN** it parses connection info (defaulting fsType to ext4 if not set), determines mount type based on protocol (dpc for DPC, dtfs for DTFS), and passes it as the -t option to the mount command
