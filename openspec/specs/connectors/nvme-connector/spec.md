## ADDED Requirements

### Requirement: NVMe connector shall connect and disconnect NVMe over Fabrics volumes
The NVMe connector shall implement the VolumeConnector interface to connect/disconnect NVMe volumes over RoCE or TCP on Kubernetes nodes.

#### Scenario: Connect NVMe over RoCE volume
- **WHEN** the connector receives a ConnectVolume request with NVMe-RoCE parameters (tgtPortals, tgtLunGuid)
- **THEN** the connector discovers NVMe subsystems on the portals, connects to the NVMe target, discovers the namespace by GUID, configures multipath, and returns the device path

#### Scenario: Connect NVMe over TCP volume
- **WHEN** the connector receives a ConnectVolume request with NVMe-TCP parameters
- **THEN** the connector connects via TCP transport instead of RoCE

#### Scenario: Disconnect NVMe volume
- **WHEN** the connector receives a DisConnectVolume request with the device GUID
- **THEN** the connector disconnects from the NVMe target and cleans up the namespace

#### Scenario: Handle NVMe initiator not found
- **WHEN** the node has no NVMe hostnqn configured (/etc/nvme/hostnqn)
- **THEN** the connector returns an error indicating no NVMe initiator exists
