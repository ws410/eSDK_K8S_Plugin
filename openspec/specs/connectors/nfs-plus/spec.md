## ADDED Requirements

### Requirement: NFS+ connector shall mount and unmount enhanced NFS shares
The NFS+ connector implements the VolumeConnector interface to mount/unmount NFS+ filesystem shares with support for multiple portals and label-based access control.

#### Scenario: Mount NFS+ share with multiple portals
- **WHEN** the connector receives a ConnectVolume request with NFS+ parameters (sourcePath, targetPath, portals=[ip1, ip2, ...], mountFlags)
- **THEN** the connector parses the connection info, joins portals with "~" as remoteaddrs, constructs mount command with `-t nfs -o remoteaddrs=<portals~separated>,<mountFlags>`, creates the target directory if needed (permission 0750), checks for existing mounts, and executes the mount command

#### Scenario: Unmount NFS+ share
- **WHEN** the connector receives a DisConnectVolume request with the targetPath
- **THEN** the connector checks if the target path exists; if not, returns success; otherwise executes umount, tolerating "not mounted" or "not found" errors, and removes the target directory using RemoveAll

#### Scenario: Reject mount with missing portals
- **WHEN** the connector receives a ConnectVolume request without portals or with empty portals
- **THEN** the connector returns error "there are no portals in the connection info"

#### Scenario: Reject mount with missing source path
- **WHEN** the connector receives a ConnectVolume request without sourcePath
- **THEN** the connector returns error "there are no source path in the connection info"

#### Scenario: Reject mount with missing target path
- **WHEN** the connector receives a ConnectVolume request without targetPath
- **THEN** the connector returns error "there are no target path in the connection info"

#### Scenario: Mount NFS+ share with connection details
- **WHEN** the connector mounts an NFS+ share
- **THEN** it performs the following:
  - Validates that all portals are either all IPs or all domains (cannot mix formats)
  - Joins portals with "~" as remoteaddrs, constructs mount command with `-t nfs -o remoteaddrs=<portals~separated>,<mountFlags>`
  - For single portal, still constructs the remoteaddrs option (no "~" separator needed)
  - Creates target directory if needed (permission 0750)
  - Checks for existing mounts: if source matches requested sourcePath, or directory base names match, or ContainSourceDevice confirms same device, returns success (idempotent); otherwise returns error about conflicting mount
