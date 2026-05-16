## ADDED Requirements

### Requirement: iSCSI connector shall connect and disconnect iSCSI volumes
The iSCSI connector implements the VolumeConnector interface to connect/disconnect iSCSI volumes on Kubernetes nodes, supporting both UltraPath and DM-Multipath.

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

#### Scenario: Connect iSCSI volume with connection details
- **WHEN** the connector connects an iSCSI volume
- **THEN** it performs the following:
  - Pings each iSCSI portal to test host connectivity, filters out unreachable portals, and only attempts login to available portals
  - Spawns a goroutine per portal for concurrent discovery, login, and device scan; each goroutine has panic recovery and updates shared data atomically
  - Configures iscsiadm with manual scan mode to prevent automatic device scanning
  - Uses exponential backoff when scanning: echoes "scan" to /sys/class/scsi_host/hostX/scan, waits with increasing intervals, and retries until device is discovered or timeout
  - If HWUltraPath multipath is enabled, polls the UltraPath device manager to discover the virtual device path
  - Calls clearResidualPath to remove stale device paths associated with the WWN before establishing new connections
