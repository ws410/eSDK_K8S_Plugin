## Purpose

定义 CSI ControllerDeleteVolume RPC 的接口规范，用于从华为存储后端删除卷，支持 DTree、NAS 和标准 LUN/FileSystem 存储类型。

## Requirements

### Requirement: DeleteVolume RPC 必须删除存储卷
DeleteVolume RPC SHALL 从华为存储后端删除卷。VolumeId 格式为 "backendName.volumeName"。驱动必须处理不同的存储类型（DTree、OceanStor A-Series NAS 和标准 LUN/FileSystem），并使用适当的删除参数。如果后端不再存在，请求返回成功并带有警告（幂等行为）。

#### Scenario:删除标准 LUN/FileSystem 卷
- **WHEN** CO 发送带有有效 VolumeId（格式："backendName.volumeName"）的 DeleteVolumeRequest 时
- **THEN** 驱动分割 VolumeId，选择后端，并使用 volName 和 nil params 调用 backend.Plugin.DeleteVolume，成功后返回空的 DeleteVolumeResponse

#### Scenario:删除 DTree 卷
- **WHEN** CO 发送 DTree 存储后端（由 constants.IsDtreeStorage 标识）上卷的 DeleteVolumeRequest 时
- **THEN** 驱动通过 GetDTreeParentNameByVolumeId 从卷 ID 映射中获取 DTree parentName，并使用 volName 和 parentName 调用 backend.Plugin.DeleteDTreeVolume

#### Scenario:删除 OceanStor A-Series NAS 卷
- **WHEN** CO 发送 OceanStorASeriesNas 存储上卷的 DeleteVolumeRequest 时
- **THEN** 驱动通过 GetKvCacheStoreIdByVolumeId 按 volumeId 获取 KvCacheStoreId，如果存在则构造包含 KvCacheStoreId 的删除参数（键：constants.KvCacheStoreId），并使用 volName 和参数调用 backend.Plugin.DeleteVolume

#### Scenario:后端不存在时删除卷
- **WHEN** CO 发送 VolumeId 引用不再存在后端的 DeleteVolumeRequest 时
- **THEN** 驱动记录警告，返回成功（codes.OK）并带有空的 DeleteVolumeResponse，并注明需要从存储阵列手动清理

#### Scenario:后端选择失败时删除卷
- **WHEN** CO 发送 DeleteVolumeRequest 且后端选择返回错误（后端存在但选择失败）时
- **THEN** 驱动将其视为与后端未找到相同的情况，返回成功并带有空响应，并记录警告
