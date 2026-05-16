## Purpose

定义 CSI ControllerCreateSnapshot RPC 的接口规范，用于在华为存储后端上创建卷快照，支持快照 ID 格式化和同步创建。

## Requirements

### Requirement: CreateSnapshot RPC 必须创建卷快照
CreateSnapshot RPC SHALL 在华为存储后端上创建现有卷的快照。快照 ID 格式为 "backendName.parentID.snapshotName"。快照同步创建并立即可用（ReadyToUse=true）。

#### Scenario:从卷创建快照
- **WHEN** CO 发送带有 SourceVolumeId、快照 Name 和可选 Parameters 的 CreateSnapshotRequest 时
- **THEN** 驱动分割 SourceVolumeId 获取 backendName 和 volName，选择后端，复制请求参数，使用 volName、snapshotName 和参数调用 backend.Plugin.CreateSnapshot，并返回 CreateSnapshotResponse，包含 Snapshot：SizeBytes（来自插件结果的 int64）、SnapshotId（格式："backendName.ParentID.snapshotName"）、SourceVolumeId（原始值）、CreationTime（从插件结果转换的 int64 转换为 timestamppb），以及 ReadyToUse=true

#### Scenario:拒绝源卷 ID 缺失的 CreateSnapshot
- **WHEN** CO 发送 SourceVolumeId 为空的 CreateSnapshotRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "Volume ID missing in request"

#### Scenario:拒绝快照名称缺失的 CreateSnapshot
- **WHEN** CO 发送 Name 为空的 CreateSnapshotRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "Snapshot Name missing in request"

#### Scenario:拒绝后端不存在时的 CreateSnapshot
- **WHEN** CO 发送 SourceVolumeId 引用不存在后端的 CreateSnapshotRequest 时
- **THEN** 驱动返回 codes.Internal 错误，指示后端不存在

#### Scenario:拒绝插件创建失败的 CreateSnapshot
- **WHEN** CO 发送 CreateSnapshotRequest 且 backend.Plugin.CreateSnapshot 返回错误时
- **THEN** 驱动返回 codes.Internal 错误，包含插件错误消息
