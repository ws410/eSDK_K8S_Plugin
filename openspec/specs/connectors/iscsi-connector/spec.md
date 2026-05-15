## ADDED Requirements

### Requirement: iSCSI connector shall connect and disconnect iSCSI volumes
The iSCSI connector shall implement the VolumeConnector interface to connect/disconnect iSCSI volumes on Kubernetes nodes, supporting both UltraPath and DM-Multipath.

#### Scenario: Connect iSCSI volume with DM-Multipath
- **WHEN** the connector receives a ConnectVolume request with iSCSI parameters (tgtPortals, tgtIQNs, tgtLunWWN)
- **THEN** the connector discovers iSCSI sessions to each portal, logs in to the targets, rescans the SCSI bus, discovers the multipath device by WWN, configures DM-Multipath, and returns the device path

#### Scenario: Connect iSCSI volume with UltraPath
- **WHEN** the connector receives a ConnectVolume request with UltraPath enabled
- **THEN** the connector configures Huawei UltraPath multipath instead of DM-Multipath

#### Scenario: Disconnect iSCSI volume
- **WHEN** the connector receives a DisConnectVolume request with the device WWN
- **THEN** the connector removes the multipath device, logs out from iSCSI targets, and cleans up the SCSI sessions

#### Scenario: Handle iSCSI login failure
- **WHEN** the iSCSI target is unreachable or authentication fails
- **THEN** the connector returns an error with the login failure reason

#### Scenario: Handle CHAP authentication
- **WHEN** the iSCSI target requires CHAP authentication
- **THEN** the connector configures CHAP credentials before logging in
