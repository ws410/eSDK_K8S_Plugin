## ADDED Requirements

### Requirement: FC-NVMe connector shall connect and disconnect FC-NVMe volumes
The FC-NVMe connector shall implement the VolumeConnector interface to connect/disconnect NVMe over Fibre Channel volumes. It uses a mutex for thread-safe operations.

#### Scenario: Connect FC-NVMe volume
- **WHEN** the connector receives a ConnectVolume request with FC-NVMe parameters (PortWWNList, TgtLunGuid)
- **THEN** the connector extracts tgtLunGuid from connection properties, validates it exists, calls ConnectVolumeCommon with the FCNVMe driver type and tryConnectVolume function, which discovers FC-NVMe subsystems, connects via FC-NVMe transport, discovers the namespace by GUID, and returns the device path

#### Scenario: Disconnect FC-NVMe volume
- **WHEN** the connector receives a DisConnectVolume request with the device GUID
- **THEN** the connector calls DisConnectVolumeCommon with the FCNVMe driver type and tryDisConnectVolume function, which disconnects from the FC-NVMe target and cleans up

#### Scenario: Reject ConnectVolume without tgtLunGuid
- **WHEN** the connector receives a ConnectVolume request without tgtLunGuid in connection properties
- **THEN** the connector returns error "there is no Lun GUID in connect info"

#### Scenario: Thread-safe FC-NVMe operations
- **WHEN** multiple goroutines call ConnectVolume or DisConnectVolume simultaneously
- **THEN** the connector's mutex ensures serialized access to FC-NVMe operations
