## ADDED Requirements

### Requirement: NodeGetInfo RPC must return node identification and topology
The NodeGetInfo RPC shall return the node's identification information to the CO. The NodeId is a JSON-encoded string containing the HostName. If topology information is available from the node's labels, it is included in the response as AccessibleTopology.

#### Scenario: Get node info without topology
- **WHEN** the CO sends a NodeGetInfoRequest and the driver's nodeName is empty (not configured via CSI_NODENAME)
- **THEN** the driver retrieves the hostname via utils.GetHostName, marshals it as JSON {"HostName": "<hostname>"}, and returns NodeGetInfoResponse with NodeId (the JSON string), MaxVolumesPerNode (from app.GetGlobalConfig().MaxVolumesPerNode), and no AccessibleTopology

#### Scenario: Get node info with topology
- **WHEN** the CO sends a NodeGetInfoRequest and the driver's nodeName is configured (via CSI_NODENAME environment variable)
- **THEN** the driver retrieves the hostname, retrieves topology segments from the node's labels via k8sUtils.GetNodeTopology (these are labels with the topology prefix), and returns NodeGetInfoResponse with NodeId, MaxVolumesPerNode, and AccessibleTopology containing the topology segments

#### Scenario: Reject NodeGetInfo when hostname retrieval fails
- **WHEN** the CO sends a NodeGetInfoRequest and utils.GetHostName fails
- **THEN** the driver returns codes.Internal error

#### Scenario: Reject NodeGetInfo when topology retrieval fails
- **WHEN** the CO sends a NodeGetInfoRequest with nodeName configured and k8sUtils.GetNodeTopology fails
- **THEN** the driver returns codes.Internal error

#### Scenario: Reject NodeGetInfo when node info marshaling fails
- **WHEN** the CO sends a NodeGetInfoRequest and json.Marshal of the node info fails
- **THEN** the driver returns codes.Internal error
