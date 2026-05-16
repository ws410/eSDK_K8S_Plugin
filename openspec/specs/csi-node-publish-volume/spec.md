## Purpose

定义 CSI NodePublishVolume RPC 的接口规范，用于在节点目标路径上发布已暂存的卷，支持块卷和文件系统卷模式。

## Requirements

### Requirement: NodePublishVolume RPC 必须将卷发布到节点目标路径
NodePublishVolume RPC SHALL 使暂存卷在节点的目标路径上可用。驱动支持块卷模式（从暂存设备绑定挂载）和文件系统卷模式（从暂存路径绑定挂载或 DTree 卷的 NFS 挂载）。

#### Scenario:将块卷发布到节点
- **WHEN** CO 发送 VolumeCapability.GetBlock() != nil 的 NodePublishVolumeRequest 时
- **THEN** 驱动设置 sourcePath=StagingTargetPath + "/" + VolumeId，调用 manage.PublishBlock，该函数执行从 sourcePath 到 TargetPath 的裸块设备绑定挂载，使用来自 VolumeCapability.GetMount().GetMountFlags() 的 mountFlags，成功后返回空的 NodePublishVolumeResponse

#### Scenario:发布文件系统卷（非 DTree）
- **WHEN** CO 发送 VolumeCapability.GetBlock() == nil 的 NodePublishVolumeRequest 且为非 DTree 后端时
- **THEN** 驱动获取后端配置（storage, protocol, portals, metroPortals），设置 sourcePath=StagingTargetPath，构造挂载选项（bind，如果 Readonly 为 true 则加上 "ro"），创建包含 srcType=MountFSType、sourcePath、targetPath、mountFlags、protocol 和 portals 的 connectInfo，调用 manage.Mount，该函数使用适当的连接器（NFS 或 NFS+），并返回空的 NodePublishVolumeResponse

#### Scenario:发布 DTree 文件系统卷
- **WHEN** CO 发送 DTree 存储后端上卷的 NodePublishVolumeRequest 时
- **THEN** 驱动获取后端配置，从 parentName 确定 DTree 源路径（如果 PublishContext.publishInfo 中有 DTreeParentName 则使用，否则从后端配置获取），按协议生成路径前缀（NFS/NFS+ 为 "portal:/"，DPC/DTFS 为 "/"），将 sourcePath 构造为 prefix + parentName + "/" + volumeName，确定挂载选项（来自 VolumeCapability 的 mountFlags，ReadOnly 访问使用 "ro"，A 系列上的 DTFS 使用 "cid=deviceWWN"），并将 DTree 共享挂载到目标路径

#### Scenario:发布 DTree 卷时缺少 parentName
- **WHEN** CO 发送 DTree 卷的 NodePublishVolumeRequest 且无法确定 parentName（不在 PublishContext 中也不在后端配置中）时
- **THEN** 驱动返回 codes.Internal 错误，指示 parentName 缺失，并建议应启用 attachRequired 参数

#### Scenario:拒绝获取后端配置失败的 NodePublishVolume
- **WHEN** CO 发送 NodePublishVolumeRequest 且 GetBackendConfig 失败（后端 ConfigMap 中缺少参数、protocol、portals 或 storage）时
- **THEN** 驱动返回 codes.Internal 错误，包含具体的失败原因

#### Scenario:拒绝挂载失败的 NodePublishVolume
- **WHEN** CO 发送 NodePublishVolumeRequest 且挂载操作失败时
- **THEN** 驱动返回 codes.Internal 错误，包含挂载失败原因
