## ADDED Requirements

### Requirement: FC connector shall connect and disconnect Fibre Channel volumes
The FC connector shall implement the VolumeConnector interface to connect/disconnect Fibre Channel volumes on Kubernetes nodes, supporting multipath.

#### Scenario: Connect FC volume
- **WHEN** the connector receives a ConnectVolume request with FC parameters (tgtWWNs, tgtLunWWN)
- **THEN** the connector triggers FC HBA rescan, discovers the FC LUN by WWN, configures multipath, and returns the device path

#### Scenario: Disconnect FC volume
- **WHEN** the connector receives a DisConnectVolume request with the device WWN
- **THEN** the connector removes the multipath device and flushes the FC HBA cache

#### Scenario: Handle FC HBA not found
- **WHEN** the node has no FC HBAs installed
- **THEN** the connector returns an error indicating no FC initiator exists

#### Scenario: Rescan FC HBA
- **WHEN** the connector needs to discover new FC LUNs
- **THEN** it triggers a rescan on all available FC HBAs by writing to the sysfs issue_lip and scan files

#### Scenario: Clear residual path by LUN ID
- **WHEN** CleanDeviceByLunId is called with host LUN ID and target WWNs
- **THEN** the connector removes stale device paths associated with the LUN ID to prevent conflicts with new connections

#### Scenario: Connect FC volume with HBA rescan retry logic
- **WHEN** the connector triggers FC HBA rescan to discover new LUNs
- **THEN** the waitDeviceDiscovery function performs up to 3 scan attempts with 2-second intervals, waiting up to 60 seconds total for the device to appear; each attempt writes to sysfs issue_lip and scan files on all online FC HBAs

#### Scenario: Connect FC volume with UltraPath device discovery wait
- **WHEN** the connector connects an FC volume with HWUltraPath multipath
- **THEN** the waitUltraPathDeviceDiscovery function polls the UltraPath device manager after the FC HBA rescan, waiting until UltraPath takes over the device before returning the virtual device path

#### Scenario: Connect FC volume with /dev/disk/by-path discovery
- **WHEN** the connector needs to discover the FC LUN device path
- **THEN** the getPossibleVolumePath function constructs /dev/disk/by-path/ paths based on FC HBA port WWN, target WWN, and LUN number, and checks which path exists
