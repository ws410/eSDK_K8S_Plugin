## ADDED Requirements

### Requirement: Local connector shall attach and detach local/SCSI volumes
The Local connector shall implement the VolumeConnector interface for FusionStorage direct-attach (local SCSI) volumes. It uses the ConnectVolumeCommon and DisConnectVolumeCommon patterns with the LocalDriver type.

#### Scenario: Attach local volume
- **WHEN** the connector receives a ConnectVolume request with local parameters (tgtLunWWN)
- **THEN** the connector extracts tgtLunWWN from connection properties, validates it exists, calls ConnectVolumeCommon with the LocalDriver type and tryConnectVolume function, which discovers the local SCSI device by WWN (e.g., /dev/disk/by-id/wwn-0x*) and returns the device path

#### Scenario: Detach local volume
- **WHEN** the connector receives a DisConnectVolume request with the device WWN
- **THEN** the connector calls DisConnectVolumeCommon with the LocalDriver type and tryDisConnectVolume function, which flushes and removes the local SCSI device

#### Scenario: Reject ConnectVolume without tgtLunWWN
- **WHEN** the connector receives a ConnectVolume request without tgtLunWWN in connection properties
- **THEN** the connector returns error "key tgtLunWWN does not exist in connection properties"

#### Scenario: Attach local volume with device polling
- **WHEN** the connector connects a local SCSI volume
- **THEN** the waitDevOnline function polls /dev/disk/by-id/wwn-0x<tgtLunWWN> up to 30 times with 2-second intervals (60 seconds total), running `ls -l` to check if the symlink exists; returns empty string (not error) if timeout

#### Scenario: Detach local volume is intentional no-op
- **WHEN** the connector receives a DisConnectVolume request for a local volume
- **THEN** the tryDisConnectVolume function is intentionally a no-op (returns nil without any action); this is by design for local/passthrough scenarios where the host owns the device and should not be disconnected by the CSI driver

#### Scenario: Attach local volume with symlink resolution
- **WHEN** the connector discovers the local SCSI device via /dev/disk/by-id/wwn-0x*
- **THEN** the waitDevOnline function resolves the symlink to get the actual device path (e.g., /dev/sdX) for device verification
