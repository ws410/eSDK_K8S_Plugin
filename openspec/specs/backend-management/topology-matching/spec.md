## ADDED Requirements

### Requirement: topology-matching shall match volume requests to backend topologies
The topology matching system shall match CreateVolume requests with accessibility requirements to backend-supported topologies, including protocol-specific topology combinations.

#### Scenario: Match requisite topologies
- **WHEN** a CreateVolume request has AccessibilityRequirements with Requisite topologies
- **THEN** the filterPoolsOnTopology function selects only pools whose backend's SupportedTopologies match at least one requisite topology

#### Scenario: Sort pools by preferred topologies
- **WHEN** a CreateVolume request has Preferred topologies
- **THEN** the sortPoolsByPreferredTopologies function orders pools by preference match, shuffling pools randomly within each preference bucket to prevent hotspots

#### Scenario: Match protocol topology
- **WHEN** a topology requirement includes a protocol key (topology.kubernetes.io/protocol.iscsi)
- **THEN** the isTopologySupportedByBackend function extracts the protocol topology from the requirement and checks if the backend supports it via the protocol-specific topology label

#### Scenario: Add protocol topology to backend
- **WHEN** a backend is initialized with a protocol (e.g., iscsi)
- **THEN** the addProtocolTopology function adds protocol topology combinations to the backend's SupportedTopologies (e.g., topology.kubernetes.io/protocol.iscsi = csi.huawei.com)

#### Scenario: Handle backend without supported topologies
- **WHEN** a backend has no SupportedTopologies configured
- **THEN** the topology filter passes all pools (no topology constraint)

#### Scenario: Handle empty requisite topologies
- **WHEN** a CreateVolume request has AccessibilityRequirements but empty Requisite list
- **THEN** the topology filter passes all pools (no requisite constraint)
