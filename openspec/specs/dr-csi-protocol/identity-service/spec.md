## ADDED Requirements

### Requirement: DR-CSI Identity service shall provide provider identity
The DR-CSI Identity gRPC service shall provide provider identification for the disaster recovery CSI protocol.

#### Scenario: Get provider name
- **WHEN** the DR-CSI client calls GetProviderName
- **THEN** the service returns the CSI driver name (e.g., csi.huawei.com)

#### Scenario: Get provider version
- **WHEN** the DR-CSI client queries provider version
- **THEN** the service returns the CSI driver version
