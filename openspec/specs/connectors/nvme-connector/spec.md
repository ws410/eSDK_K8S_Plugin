## ADDED Requirements

### Requirement: NVMe connector shall connect and disconnect NVMe over Fabrics volumes
The NVMe connector shall implement the VolumeConnector interface to connect/disconnect NVMe volumes over RoCE or TCP on Kubernetes nodes.

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

#### Scenario: Reject DisConnectVolume without tgtLunGuid
- **WHEN** the connector receives a DisConnectVolume request
- **THEN** the connector uses the tgtLunGuid parameter as the identifier for disconnecting

#### Scenario: Connect NVMe volume with NVMe CLI version check
- **WHEN** the connector initializes or processes NVMe operations
- **THEN** the connector validates that the installed nvme-cli version is >= 1.9; if the version is too old, returns an error

#### Scenario: Connect NVMe volume with portal ping filtering
- **WHEN** the connector receives a ConnectVolume request with multiple NVMe portals
- **THEN** the parseNVMeInfo function pings each portal to test host connectivity, filters to availablePortals, and returns an error "No portal available" if all portals are unreachable

#### Scenario: Connect NVMe volume with session existence check
- **WHEN** the connector connects to NVMe portals
- **THEN** the getExistSessions function runs `nvme list-subsys -o json` to check for existing connected sessions, avoiding duplicate connections to portals that are already connected

#### Scenario: Connect NVMe volume with UltraPath NVMe multipath
- **WHEN** the connector connects an NVMe volume with HWUltraPathNVMe multipath type
- **THEN** the findDiskOfUltraPath function polls `upadmin_plus` with a 15-second timeout to discover the UltraPath NVMe virtual device path

#### Scenario: Connect NVMe volume with NVMe-Native multipath
- **WHEN** the connector connects an NVMe volume with NVMeNative multipath type
- **THEN** the connector checks if NVMe multipath is enabled, uses findDiskOfNativePath to discover the native multipath device, and calls waitAllPathOnline to ensure all paths are online before proceeding

#### Scenario: Disconnect NVMe volume with session cleanup
- **WHEN** the connector disconnects an NVMe volume
- **THEN** the disconnectSessions function checks each session port for multiple controllers; if only 1 controller remains, runs `nvme disconnect -d <port>`; for multipath devices, sleeps 3 seconds and calls FlushDMDevice with 3 retries at 20-second intervals

#### Scenario: Connect NVMe volume with transport type mapping
- **WHEN** the connector parses NVMe connection info
- **THEN** the protocol is mapped to transport type: ProtocolRoce/ProtocolRoceNVMe -> "rdma", ProtocolTCPNVMe -> "tcp"
