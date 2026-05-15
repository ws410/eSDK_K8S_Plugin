## ADDED Requirements

### Requirement: FC-NVMe connector shall connect and disconnect FC-NVMe volumes
The FC-NVMe connector shall implement the VolumeConnector interface to connect/disconnect NVMe over Fibre Channel volumes.

#### Scenario: Connect FC-NVMe volume
- **WHEN** the connector receives a ConnectVolume request with FC-NVMe parameters (PortWWNList, TgtLunGuid)
- **THEN** the connector discovers FC-NVMe subsystems, connects via FC-NVMe transport, discovers the namespace by GUID, and returns the device path

#### Scenario: Disconnect FC-NVMe volume
- **WHEN** the connector receives a DisConnectVolume request
- **THEN** the connector disconnects from the FC-NVMe target and cleans up
