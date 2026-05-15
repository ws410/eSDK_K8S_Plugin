## ADDED Requirements

### Requirement: iSCSI connector shall connect and disconnect iSCSI volumes
The iSCSI connector shall implement the VolumeConnector interface to connect/disconnect iSCSI volumes on Kubernetes nodes, supporting both UltraPath and DM-Multipath.

#### Scenario: Connect iSCSI volume with DM-Multipath
- **WHEN** the connector receives a ConnectVolume request with iSCSI parameters (tgtPortals, tgtIQNs, tgtLunWWN)
- **THEN** the connector discovers iSCSI sessions to each portal, logs in to the targets, rescans the SCSI bus, discovers the multipath device by WWN, configures DM-Multipath, and returns the device path

#### Scenario: Connect iSCSI volume with UltraPath
- **WHEN** the connector receives a ConnectVolume request with UltraPath enabled
- **THEN** the connector configures Huawei UltraPath multipath instead of DM-Multipath

#### Scenario: Disconnect iSCSI volume
- **WHEN** the connector receives a DisConnectVolume request with the device WWN
- **THEN** the connector removes the multipath device, logs out from iSCSI targets, and cleans up the SCSI sessions

#### Scenario: Handle iSCSI login failure
- **WHEN** the iSCSI target is unreachable or authentication fails
- **THEN** the connector returns an error with the login failure reason

#### Scenario: Handle CHAP authentication
- **WHEN** the iSCSI target requires CHAP authentication
- **THEN** the connector configures CHAP credentials before logging in

#### Scenario: Clear residual path by WWN
- **WHEN** clearResidualPath is called with a device WWN
- **THEN** the connector checks for stale device paths associated with the WWN and removes them before establishing a new connection

#### Scenario: Resize block device
- **WHEN** ResizeBlock is called with device WWN and new size
- **THEN** the connector rescans the SCSI device to pick up the new size and verifies the block device reflects the updated capacity

#### Scenario: Resize filesystem
- **WHEN** ResizeMountPath is called with a mount point path
- **THEN** the connector runs filesystem resize (e.g., resize2fs for ext*, xfs_growfs for xfs) to expand the filesystem to fill the resized block device

#### Scenario: Connect iSCSI volume with portal ping filtering
- **WHEN** the connector receives a ConnectVolume request with multiple iSCSI portals
- **THEN** the constructISCSIInfo function pings each portal to test host connectivity, filters out unreachable portals, and only attempts login to available portals

#### Scenario: Connect iSCSI volume with concurrent goroutine connections
- **WHEN** the connector connects to multiple iSCSI portals
- **THEN** for each portal, a goroutine is spawned to perform discovery, login, and device scan concurrently; each goroutine has panic recovery and updates shared data atomically

#### Scenario: Connect iSCSI volume with device scan exponential backoff
- **WHEN** the connector scans for iSCSI devices after login
- **THEN** the deviceScan.scan function uses exponential backoff: echoes "scan" to /sys/class/scsi_host/hostX/scan, waits with increasing intervals, and retries until the device is discovered or timeout

#### Scenario: Connect iSCSI volume with manual scan mode
- **WHEN** the connector logs in to an iSCSI target
- **THEN** iscsiadm is configured with manual scan mode to prevent automatic device scanning, giving the connector control over when to scan for new devices

#### Scenario: Connect iSCSI volume with UltraPath device discovery
- **WHEN** the connector connects an iSCSI volume with HWUltraPath multipath
- **THEN** the findDiskOfUltraPath function polls the UltraPath device manager to discover the virtual device path, waiting until the device is taken over by UltraPath
