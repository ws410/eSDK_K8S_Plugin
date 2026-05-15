## ADDED Requirements

### Requirement: Local connector shall attach and detach local/SCSI volumes
The Local connector shall implement the VolumeConnector interface for FusionStorage direct-attach (local SCSI) volumes.

#### Scenario: Attach local volume
- **WHEN** the connector receives a ConnectVolume request with local parameters (tgtLunWWN)
- **THEN** the connector discovers the local SCSI device by WWN and returns the device path

#### Scenario: Detach local volume
- **WHEN** the connector receives a DisConnectVolume request with the device WWN
- **THEN** the connector flushes and removes the local SCSI device
