## Purpose

定义 CSI ControllerDeleteSnapshot RPC 的接口规范，用于从华为存储后端删除卷快照，支持幂等删除操作。

## Requirements

### Requirement: DeleteSnapshot RPC 必须删除卷快照
DeleteSnapshot RPC SHALL 从华为存储后端删除快照。SnapshotId 格式为 "backendName.parentID.snapshotName"。如果后端不再存在，请求返回成功并带有警告（幂等行为）。

#### Scenario:删除快照
- **WHEN** CO 发送带有有效 SnapshotId（格式："backendName.parentID.snapshotName"）的 DeleteSnapshotRequest 时
- **THEN** 驱动使用 utils.SplitSnapshotId 分割 SnapshotId 获取 backendName、snapshotParentId 和 snapshotName，选择后端，使用 snapshotParentId 和 snapshotName 调用 backend.Plugin.DeleteSnapshot，成功后返回空的 DeleteSnapshotResponse

#### Scenario:后端不存在时删除快照
- **WHEN** CO 发送 SnapshotId 引用不存在后端的 DeleteSnapshotRequest 时
- **THEN** 驱动记录警告，返回成功并带有空的 DeleteSnapshotResponse，并注明需要从存储阵列手动清理

#### Scenario:拒绝快照 ID 缺失的 DeleteSnapshot
- **WHEN** CO 发送 SnapshotId 为空的 DeleteSnapshotRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "Snapshot ID missing in request"

#### Scenario:拒绝插件删除失败的 DeleteSnapshot
- **WHEN** CO 发送 DeleteSnapshotRequest 且 backend.Plugin.DeleteSnapshot 返回错误时
- **THEN** 驱动返回 codes.Internal 错误，包含插件错误消息

---

### Requirement: ListSnapshots RPC 未实现
ListSnapshots RPC SHALL NOT 由此驱动实现。驱动对所有请求返回 codes.Unimplemented。

#### Scenario:CO 请求快照列表
- **WHEN** CO 发送 ListSnapshotsRequest 时
- **THEN** 驱动返回 codes.Unimplemented 错误，消息为空
