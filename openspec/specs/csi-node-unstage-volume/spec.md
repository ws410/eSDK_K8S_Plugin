## Purpose

定义 CSI NodeUnstageVolume RPC 的接口规范，用于从节点暂存目标路径移除已暂存的卷，支持 SAN 断开连接和 NAS 卸载操作。

## Requirements

### Requirement: NodeUnstageVolume RPC 必须从节点取消暂存卷
NodeUnstageVolume RPC SHALL 从节点的暂存目标路径移除已暂存的卷。驱动为后端创建 manage.Manager 并委托取消暂存操作。对于 SAN 卷，驱动必须检索设备 WWN（从磁盘文件或目标路径），卸载暂存路径，断开卷连接，并清理 WWN 文件。对于 NAS 卷，仅需卸载。DTree 卷不需要暂存/取消暂存。

#### Scenario:取消暂存 SAN 卷
- **WHEN** CO 发送带有 VolumeId 和 StagingTargetPath 的 NodeUnstageVolumeRequest 时
- **THEN** 驱动分割 VolumeId 获取 backendName，创建 SanManager，检索设备 WWN（首先从 WWN 文件获取，如果文件不存在则通过 /proc/mounts 从目标路径获取），如果从目标路径获取则将 WWN 写入磁盘（用于幂等重试），检查裸块暂存路径是否存在并相应卸载，调用 manager.UnStageWithWwn 按 WWN 断开卷连接，移除 WWN 文件，成功后返回空的 NodeUnstageVolumeResponse

#### Scenario:WWN 检索失败时取消暂存 SAN 卷
- **WHEN** CO 发送 NodeUnstageVolumeRequest 且无法检索设备 WWN（文件不存在且目标路径不包含 WWN 信息）时
- **THEN** 驱动记录警告并返回成功（幂等——没有 WWN 重试不太可能有帮助）

#### Scenario:取消暂存 NAS 卷
- **WHEN** CO 发送 NAS 后端上卷的 NodeUnstageVolumeRequest 时
- **THEN** 驱动创建 NasManager；对于 DTree 存储，立即返回（不需要取消暂存）；对于其他 NAS 类型，卸载暂存目标路径并返回空的 NodeUnstageVolumeResponse

#### Scenario:已卸载时取消暂存 NAS 卷
- **WHEN** CO 发送 NodeUnstageVolumeRequest 且暂存目标路径已卸载时
- **THEN** 驱动返回成功（幂等行为）

#### Scenario:拒绝创建管理器失败的 NodeUnstageVolume
- **WHEN** CO 发送 NodeUnstageVolumeRequest 且后端的 manage.NewManager 调用失败时
- **THEN** 驱动返回 codes.Internal 错误，消息指示后端和失败原因

#### Scenario:拒绝卸载失败的 NodeUnstageVolume
- **WHEN** CO 发送 NodeUnstageVolumeRequest 且卸载操作失败时
- **THEN** 驱动返回 codes.Internal 错误，包含卸载失败原因

#### Scenario:从目标路径回退检索 WWN 取消暂存 SAN 卷
- **WHEN** CO 发送 NodeUnstageVolumeRequest 且 WWN 文件在预期路径不存在时
- **THEN** 驱动读取 /proc/mounts 以查找与暂存目标路径关联的设备，从设备路径提取 WWN，将 WWN 写入磁盘用于幂等重试，然后继续取消暂存

#### Scenario:检测裸块暂存路径取消暂存 SAN 卷
- **WHEN** CO 发送块卷的 NodeUnstageVolumeRequest 时
- **THEN** 驱动检查暂存目标路径是否作为文件存在（裸块绑定挂载），并在断开卷连接之前相应地卸载它
