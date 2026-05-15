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
