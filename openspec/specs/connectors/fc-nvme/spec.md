## ADDED Requirements

### Requirement: FC-NVMe connector shall connect and disconnect FC-NVMe volumes
The FC-NVMe connector implements the VolumeConnector interface to connect/disconnect NVMe over Fibre Channel volumes. It uses a mutex for thread-safe operations.

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

#### Scenario: Connect FC-NVMe volume with discovery details
- **WHEN** the connector connects an FC-NVMe volume
- **THEN** it performs the following:
  - Runs `nvme list-subsys -o json` to discover FC-NVMe channels, filtering for Transport="fc" and State="live", matching by initiator and target WWN pairs
  - Runs `nvme ns-rescan /dev/<channel>` for each channel to discover new namespaces
  - Polls up to 5 times with 1-second intervals for UltraPath-NVMe device discovery, checking for residual paths via IsUpNVMeResidualPath before returning

#### Scenario: Disconnect FC-NVMe volume without session cleanup
- **WHEN** the connector disconnects an FC-NVMe volume
- **THEN** unlike NVMe over Fabrics, the FC-NVMe connector does NOT perform explicit session disconnect (the FC fabric handles this automatically); it only removes virtual and physical devices, and flushes DM device if multipath is enabled
