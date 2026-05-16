## ADDED Requirements

### Requirement: DR-CSI Identity service shall provide provider identity
The DR-CSI Identity gRPC service shall provide provider identification for the disaster recovery CSI protocol, implementing the identity server interface registered on the DR-CSI gRPC server.

#### Scenario: Get provider name
- **WHEN** the DR-CSI client calls GetProviderName
- **THEN** the service returns the CSI driver name (e.g., csi.huawei.com) from the provider's name field

#### Scenario: Get provider version
- **WHEN** the DR-CSI client queries provider version
- **THEN** the service returns the CSI driver version from the provider's version field

#### Scenario: DR-CSI Identity server registration
- **WHEN** the CSI controller starts
- **THEN** registerDRCSIServer creates a Provider with driver name and version, and registers the IdentityServer, StorageBackendServer, and ModifyVolumeInterfaceServer on the DR-CSI gRPC server listening on the DR endpoint

#### Scenario: Probe always returns success
- **WHEN** the DR-CSI client calls Probe
- **THEN** the service always returns an empty ProbeResponse with the ready field unset (nil); no health checking is performed
