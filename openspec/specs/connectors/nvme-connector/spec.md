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
