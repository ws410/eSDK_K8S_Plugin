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

#### Scenario: Connect FC-NVMe volume with channel matching by WWN
- **WHEN** the connector discovers FC-NVMe channels
- **THEN** the getAllChannel function runs `nvme list-subsys -o json`, filters for paths with Transport="fc" and State="live", and matches channels by checking if both initiator and target WWN pairs appear in the address string

#### Scenario: Connect FC-NVMe volume with device rescan
- **WHEN** the connector has identified FC-NVMe channels
- **THEN** the scanDevice function runs `nvme ns-rescan /dev/<channel>` for each channel to discover new namespaces

#### Scenario: Connect FC-NVMe volume with virtual device polling
- **WHEN** the connector needs to discover the FC-NVMe virtual device with multipath
- **THEN** the getVirtualDeviceUseMultipath function polls up to 5 times with 1-second intervals for GetDevNameByLunWWN with UltraPath-NVMe, checking for residual paths via IsUpNVMeResidualPath before returning

#### Scenario: Disconnect FC-NVMe volume without session cleanup
- **WHEN** the connector disconnects an FC-NVMe volume
- **THEN** unlike NVMe over Fabrics, the FC-NVMe connector does NOT perform explicit session disconnect (the FC fabric handles this automatically); it only removes virtual and physical devices, and flushes DM device if multipath is enabled
