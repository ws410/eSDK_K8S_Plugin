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

#### Scenario: Connect FC volume with discovery and multipath details
- **WHEN** the connector connects an FC volume
- **THEN** it performs the following:
  - Triggers FC HBA rescan by writing to sysfs issue_lip and scan files on all online FC HBAs
  - Performs up to 3 scan attempts with 2-second intervals, waiting up to 60 seconds total for device discovery
  - Discovers the FC LUN device path via /dev/disk/by-path/ constructed from FC HBA port WWN, target WWN, and LUN number
  - If HWUltraPath multipath is enabled, polls the UltraPath device manager until it takes over the device before returning the virtual device path
  - Calls CleanDeviceByLunId to remove stale device paths associated with the LUN ID before establishing new connections
