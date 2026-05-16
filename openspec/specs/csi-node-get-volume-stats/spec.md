## Purpose

定义 CSI NodeGetVolumeStats RPC 的接口规范，用于返回卷使用统计信息，包括字节和 inode 使用情况指标。

## Requirements

### Requirement: NodeGetVolumeStats RPC 必须返回卷使用指标
NodeGetVolumeStats RPC SHALL 返回卷使用统计信息，包括字节和 inode 使用情况。驱动使用 utils.GetVolumeMetrics 从卷路径收集指标，并返回 BYTES 和 INODES 使用信息。

#### Scenario:使用有效卷路径获取卷统计
- **WHEN** CO 发送带有有效 VolumeId 和 VolumePath 的 NodeGetVolumeStatsRequest 时
- **THEN** 驱动从路径收集卷指标（Available, Capacity, Used, InodesFree, Inodes, InodesUsed），验证所有指标可转换为 int64，并返回 NodeGetVolumeStatsResponse，包含两个 VolumeUsage 条目：一个用于 BYTES（Available, Total, Used），一个用于 INODES（Available, Total, Used）

#### Scenario:拒绝卷 ID 为空的 NodeGetVolumeStats
- **WHEN** CO 发送 VolumeId 为空的 NodeGetVolumeStatsRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "no volume ID provided"

#### Scenario:拒绝卷路径为空的 NodeGetVolumeStats
- **WHEN** CO 发送 VolumePath 为空的 NodeGetVolumeStatsRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "no volume Path provided"

#### Scenario:拒绝指标收集失败的 NodeGetVolumeStats
- **WHEN** CO 发送 NodeGetVolumeStatsRequest 且 utils.GetVolumeMetrics 对卷路径失败时
- **THEN** 驱动返回 codes.Internal 错误，包含失败原因

#### Scenario:拒绝指标值无效的 NodeGetVolumeStats
- **WHEN** CO 发送 NodeGetVolumeStatsRequest 且任何收集的指标（Available, Capacity, Used, InodesFree, Inodes, InodesUsed）无法转换为 int64 时
- **THEN** 驱动返回 codes.Internal 错误，指示哪个指标无效
