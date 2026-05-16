## Purpose

定义 CSI ControllerUnpublishVolume RPC 的接口规范，用于从华为存储阵列上将卷从节点分离，支持 DTree 卷和幂等操作。

## Requirements

### Requirement: ControllerUnpublishVolume RPC 必须从节点分离卷
ControllerUnpublishVolume RPC SHALL 在华为存储阵列上将卷从特定节点分离（取消发布）。如果后端不再存在，请求返回成功并带有警告（幂等行为）。对于 DTree 卷，驱动必须在分离参数中包含 parentName。

#### Scenario:从节点取消发布卷
- **WHEN** CO 发送带有 VolumeId 和 NodeId 的 ControllerUnpublishVolumeRequest 时
- **THEN** 驱动分割 VolumeId 获取 backendName 和 volName，解组 NodeId JSON 提取节点参数，调用 backend.Plugin.DetachVolume（传入参数），成功后返回空的 ControllerUnpublishVolumeResponse

#### Scenario:从节点取消发布 DTree 卷
- **WHEN** CO 发送 DTree 存储后端（由 constants.IsDtreeStorage 标识）上卷的 ControllerUnpublishVolumeRequest 时
- **THEN** 驱动通过 GetDTreeParentNameByVolumeId 从卷 ID 映射中获取 DTree parentName，将其以键 constants.DTreeParentKey 添加到参数中，并使用更新后的参数调用 backend.Plugin.DetachVolume

#### Scenario:后端不存在时取消发布卷
- **WHEN** CO 发送 VolumeId 引用不存在后端的 ControllerUnpublishVolumeRequest 时
- **THEN** 驱动记录警告，返回成功并带有空的 ControllerUnpublishVolumeResponse，并注明需要从存储阵列手动分离

#### Scenario:拒绝 nodeId JSON 无效的 ControllerUnpublishVolume
- **WHEN** CO 发送 NodeId 无法解组为 JSON 的 ControllerUnpublishVolumeRequest 时
- **THEN** 驱动返回 codes.Internal 错误，包含解组错误消息
