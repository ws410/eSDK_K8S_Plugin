## ADDED Requirements

### Requirement: NodeGetVolumeStats RPC must return volume usage metrics
The NodeGetVolumeStats RPC shall return volume usage statistics including bytes and inodes usage. The driver collects metrics from the volume path using utils.GetVolumeMetrics and returns both BYTES and INODES usage information.

#### Scenario: Get volume stats with valid volume path
- **WHEN** the CO sends a NodeGetVolumeStatsRequest with valid VolumeId and VolumePath
- **THEN** the driver collects volume metrics from the path (Available, Capacity, Used, InodesFree, Inodes, InodesUsed), validates all metrics can be converted to int64, and returns NodeGetVolumeStatsResponse with two VolumeUsage entries: one for BYTES (Available, Total, Used) and one for INODES (Available, Total, Used)

#### Scenario: Reject NodeGetVolumeStats with empty volume ID
- **WHEN** the CO sends a NodeGetVolumeStatsRequest with an empty VolumeId
- **THEN** the driver returns codes.InvalidArgument error with message "no volume ID provided"

#### Scenario: Reject NodeGetVolumeStats with empty volume path
- **WHEN** the CO sends a NodeGetVolumeStatsRequest with an empty VolumePath
- **THEN** the driver returns codes.InvalidArgument error with message "no volume Path provided"

#### Scenario: Reject NodeGetVolumeStats when metrics collection fails
- **WHEN** the CO sends a NodeGetVolumeStatsRequest and utils.GetVolumeMetrics fails for the volume path
- **THEN** the driver returns codes.Internal error with the failure reason

#### Scenario: Reject NodeGetVolumeStats when metric values are invalid
- **WHEN** the CO sends a NodeGetVolumeStatsRequest and any of the collected metrics (Available, Capacity, Used, InodesFree, Inodes, InodesUsed) cannot be converted to int64
- **THEN** the driver returns codes.Internal error indicating which metric is invalid
