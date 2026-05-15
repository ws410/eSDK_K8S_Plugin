## ADDED Requirements

### Requirement: NFS+ connector shall mount and unmount enhanced NFS shares
The NFS+ connector shall implement the VolumeConnector interface to mount/unmount NFS+ filesystem shares with support for multiple portals and label-based access control.

#### Scenario: Mount NFS+ share with multiple portals
- **WHEN** the connector receives a ConnectVolume request with NFS+ parameters (sourcePath, targetPath, portals=[ip1, ip2, ...], mountFlags)
- **THEN** the connector parses the connection info, joins portals with "~" as remoteaddrs, constructs mount command with `-t nfs -o remoteaddrs=<portals~separated>,<mountFlags>`, creates the target directory if needed (permission 0750), checks for existing mounts, and executes the mount command

#### Scenario: Mount NFS+ share with idempotent check
- **WHEN** the connector receives a ConnectVolume request and the targetPath is already mounted
- **THEN** it reads mount points; if the existing mount's source matches the requested sourcePath, or the directory base names match, or ContainSourceDevice confirms the same device, it returns success (idempotent); otherwise returns error about conflicting mount

#### Scenario: Unmount NFS+ share
- **WHEN** the connector receives a DisConnectVolume request with the targetPath
- **THEN** the connector checks if the target path exists; if not, returns success; otherwise executes umount, tolerating "not mounted" or "not found" errors, and removes the target directory using RemoveAll

#### Scenario: Mount NFS+ with HyperMetro
- **WHEN** the connector receives a ConnectVolume request with HyperMetro filesystem mode
- **THEN** the caller (NasManager) combines local and metro portals into a single portals list before passing to the connector; the connector mounts with all portals in the remoteaddrs option

#### Scenario: Reject mount with missing portals
- **WHEN** the connector receives a ConnectVolume request without portals or with empty portals
- **THEN** the connector returns error "there are no portals in the connection info"

#### Scenario: Reject mount with missing source path
- **WHEN** the connector receives a ConnectVolume request without sourcePath
- **THEN** the connector returns error "there are no source path in the connection info"

#### Scenario: Reject mount with missing target path
- **WHEN** the connector receives a ConnectVolume request without targetPath
- **THEN** the connector returns error "there are no target path in the connection info"

#### Scenario: Handle conflicting mount
- **WHEN** the connector detects the targetPath is already mounted to a different source
- **THEN** it returns error "The mount <targetPath> is already exist, source: <requested> realSource: <actual>"

#### Scenario: Mount NFS+ with IP/domain format consistency check
- **WHEN** the connector receives a ConnectVolume request with portals containing a mix of IP addresses and domain names
- **THEN** the parseNFSPlusInfo function validates that all portals are either all IPs or all domains (cannot mix), returning an error if the format is inconsistent

#### Scenario: Mount NFS+ with single portal
- **WHEN** the connector receives a ConnectVolume request with a single portal in the portals list
- **THEN** the connector still constructs the remoteaddrs mount option with the single portal (no "~" separator needed for single portal)
