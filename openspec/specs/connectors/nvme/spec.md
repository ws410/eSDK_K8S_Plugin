## ADDED Requirements

### Requirement: NVMe connector shall connect and disconnect NVMe over Fabrics volumes
The NVMe connector implements the VolumeConnector interface to connect/disconnect NVMe volumes over RoCE or TCP on Kubernetes nodes.

#### Scenario: Connect NVMe over RoCE volume
- **WHEN** the connector receives a ConnectVolume request with NVMe-RoCE parameters (tgtPortals, tgtLunGuid)
- **THEN** the connector extracts tgtLunGuid from connection properties, calls ConnectVolumeCommon with the NVMe driver type and tryConnectVolume function, which discovers NVMe subsystems on the portals, connects to the NVMe target, discovers the namespace by GUID, configures multipath, and returns the device path

#### Scenario: Connect NVMe over TCP volume
- **WHEN** the connector receives a ConnectVolume request with NVMe-TCP parameters
- **THEN** the connector connects via TCP transport instead of RoCE

#### Scenario: Disconnect NVMe volume
- **WHEN** the connector receives a DisConnectVolume request with the device GUID
- **THEN** the connector calls DisConnectVolumeCommon with the NVMe driver type and tryDisConnectVolume function, which disconnects from the NVMe target and cleans up the namespace

#### Scenario: Handle NVMe initiator not found
- **WHEN** the node has no NVMe hostnqn configured (/etc/nvme/hostnqn)
- **THEN** the connector returns an error indicating no NVMe initiator exists

#### Scenario: Reject ConnectVolume without tgtLunGuid
- **WHEN** the connector receives a ConnectVolume request without tgtLunGuid in connection properties
- **THEN** the connector returns error "key tgtLunGuid does not exist in connection properties"

#### Scenario: Connect NVMe volume with connection details
- **WHEN** the connector connects an NVMe volume (RoCE or TCP)
- **THEN** it performs the following:
  - Validates that nvme-cli version is >= 1.9; returns error if too old
  - Pings each portal to test host connectivity, filters to availablePortals; returns error if all portals unreachable
  - Maps protocol to transport type: ProtocolRoce/ProtocolRoceNVMe -> "rdma", ProtocolTCPNVMe -> "tcp"
  - Runs `nvme list-subsys -o json` to check for existing connected sessions, avoiding duplicate connections
  - If HWUltraPathNVMe multipath is enabled, polls `upadmin_plus` with 15-second timeout to discover the virtual device path
  - If NVMeNative multipath is enabled, checks NVMe multipath is enabled, uses findDiskOfNativePath to discover native multipath device, and calls waitAllPathOnline

#### Scenario: Disconnect NVMe volume with session cleanup
- **WHEN** the connector disconnects an NVMe volume
- **THEN** the disconnectSessions function checks each session port for multiple controllers; if only 1 controller remains, runs `nvme disconnect -d <port>`; for multipath devices, sleeps 3 seconds and calls FlushDMDevice with 3 retries at 20-second intervals
