## ADDED Requirements

### Requirement: NFS+ connector shall mount and unmount enhanced NFS shares
The NFS+ connector shall implement the VolumeConnector interface to mount/unmount NFS+ filesystem shares with support for multiple portals and label-based access.

#### Scenario: Mount NFS+ share with multiple portals
- **WHEN** the connector receives a ConnectVolume request with NFS+ parameters (sourcePath, targetPath, protocol=nfs+, portals=[ip1, ip2, ...], mountFlags)
- **THEN** the connector mounts the NFS+ share using multiple portals for redundancy, applies label-based access control, and returns success

#### Scenario: Mount NFS+ share with domain names
- **WHEN** the connector receives a ConnectVolume request with domain names in the portals list
- **THEN** the connector resolves the domain names to IPs and mounts the share

#### Scenario: Unmount NFS+ share
- **WHEN** the connector receives a DisConnectVolume request with the targetPath
- **THEN** the connector unmounts the NFS+ share and cleans up

#### Scenario: Mount NFS+ with HyperMetro
- **WHEN** the connector receives a ConnectVolume request with HyperMetro filesystem mode
- **THEN** the connector combines local and metro portals into a single portals list for the mount operation
