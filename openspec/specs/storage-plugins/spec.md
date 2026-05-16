## Purpose

定义存储阵列插件系统规范，为华为存储类型（OceanStor SAN/NAS、DTree、A-Series、DME、FusionStorage、OceanDisk）实现 StoragePlugin 接口，处理阵列特定的卷操作、快照、能力发现和参数验证。

## Requirements

### Requirement: storage-plugins SHALL manage storage array operations
存储插件系统 SHALL 为不同的华为存储类型实现 StoragePlugin 接口。每个插件处理阵列特定的操作：卷创建/删除/扩容、附加/分离、快照管理、能力发现和参数验证。

#### Scenario: 插件注册
- **WHEN** CSI 驱动初始化后端时
- **THEN** 系统 SHALL 根据存储类型加载对应的 StoragePlugin 实现（OceanStor、DTree、A-Series、DME、FusionStorage 或 OceanDisk）

---

### Requirement: OceanStor plugin SHALL manage OceanStor and Dorado storage arrays
OceanStor 插件 SHALL 为 oceanstor-san 和 oceanstor-nas 存储类型实现 StoragePlugin 接口，支持 Dorado V3/V5/V6/V7 和 OceanStor V3/V5。

#### Scenario:初始化 OceanStor 插件
- **WHEN** 插件使用后端配置初始化时
- **THEN** 它创建 OceanStor 客户端，登录，设置系统信息，对于 Dorado V6/V7 切换到 V6 客户端并启用 LIF 管理

#### Scenario:创建 LUN 卷
- **WHEN** 使用 volumeType=lun 调用 CreateVolume 时
- **THEN** 插件在存储阵列上创建具有指定大小、存储池和参数（allocType、QoS 等）的 LUN

#### Scenario:创建 FileSystem 卷
- **WHEN** 使用 volumeType=fs 调用 CreateVolume 时
- **THEN** 插件在存储阵列上创建具有指定参数的 FileSystem

#### Scenario:更新后端能力
- **WHEN** 插件更新能力时
- **THEN** 它查询许可证功能并设置：SupportThin（SmartThin）、SupportThick（非 Dorado）、SupportQoS（SmartQoS）、SupportMetro（HyperMetro）、SupportMetroNAS（HyperMetroNAS）、SupportReplication（HyperReplication）、SupportClone（HyperClone/HyperCopy）、SupportApplicationType（仅 Dorado V6/V7）

#### Scenario:通过 iSCSI/FC 附加卷
- **WHEN** 调用 AttachVolume 时
- **THEN** 插件创建/更新主机，添加启动器，创建映射视图，并返回映射信息（portals、IQNs/WWNs、LUN WWN、主机 LUN ID）

#### Scenario:创建快照
- **WHEN** 调用 CreateSnapshot 时
- **THEN** 插件创建 HyperSnap 快照并返回快照元数据（SizeBytes、ParentID、CreationTime）

#### Scenario:扩容卷
- **WHEN** 调用 ExpandVolume 时
- **THEN** 插件将 LUN/FileSystem 扩容到新大小，并返回是否需要节点扩容

#### Scenario:删除卷
- **WHEN** 调用 DeleteVolume 时
- **THEN** 插件从存储阵列删除 LUN 或 FileSystem

#### Scenario:查询卷
- **WHEN** 调用 QueryVolume 时
- **THEN** 插件返回卷元数据，包括大小、LUN WWN 和其他属性

#### Scenario:分离卷
- **WHEN** 调用 DetachVolume 时
- **THEN** 插件从指定主机移除卷的主机映射

#### Scenario:删除快照
- **WHEN** 使用 snapshotParentId 和 snapshotName 调用 DeleteSnapshot 时
- **THEN** 插件从存储阵列删除 HyperSnap 快照

#### Scenario:处理 Dorado V6/V7 客户端切换
- **WHEN** 插件检测到存储阵列为 Dorado V6 或 V7 时
- **THEN** 它从标准客户端切换到 V6 客户端，并为 NAS 操作配置 LIF（逻辑接口）管理

#### Scenario:支持 QoS 参数验证
- **WHEN** 使用 QoS 配置调用 SupportQoSParameters 时
- **THEN** 插件针对存储阵列支持的 QoS 类型（IOPS、带宽）验证 QoS 参数

#### Scenario:附加卷时处理双活双站点
- **WHEN** 为启用双活的卷调用 AttachVolume 时
- **THEN** 处理方法检查存储在线状态：如果本地和远程均在线，使用 metroHandler 进行双站点附加；如果仅本地在线，回退到使用本地客户端的 commonHandler；如果仅远程在线，回退到使用远程客户端的 commonHandler；如果两者均离线，返回错误

#### Scenario:扩容 NAS 卷时断言逻辑端口站点
- **WHEN** 为 OceanStor NAS 卷调用 ExpandVolume 且 metroRemotePlugin 为 nil 时
- **THEN** 插件调用 assertLogicPortRunOnOwnSite 验证 NAS 逻辑端口在本地站点运行，然后再继续扩容

#### Scenario:DTree 卷操作
- **WHEN** 为 OceanStor DTree 卷调用 CreateVolume 时
- **THEN** 插件使用 getValidParentname 针对后端的 parentname 配置验证 parentname：如果 SC 和后端 parentname 都设置，则必须匹配；如果一个为空，则使用另一个；如果两者都为空，返回错误
- **WHEN** 为启用 nfsAutoAuthClient 的 DTree 卷调用 DetachVolume 时
- **THEN** 插件按 CIDR 过滤 IP，使用 NoAccess 调用 dtree.AutoManageAuthClient 移除 NFS 客户端授权，如果 IOIsolation 为 true，则调用 dtree.CheckAllClientsStatus 验证所有客户端已断开连接

---

### Requirement: DTree plugin SHALL manage DTree quota volumes
DTree 插件（OceanstorDTreePlugin）SHALL 为 oceanstor-dtree 存储类型实现 StoragePlugin 接口，管理 OceanStor NAS 系统上的目录树配额。它扩展了 OceanstorPlugin，添加了 DTree 特定操作。

#### Scenario:初始化 DTree 插件
- **WHEN** 插件初始化时
- **THEN** 它从参数中提取 parentname（可选，如果提供必须是字符串），验证协议和门户，从参数验证并创建 NfsAutoAuthClient，并调用父级 OceanstorPlugin 初始化

#### Scenario:创建 DTree 卷
- **WHEN** 使用 volumeType=dtree 调用 CreateVolume 时
- **THEN** 插件从 PV 名称或参数解析卷名称（仅适用于 Dorado V6/V7），从 StorageClass 参数或后端配置确定有效的 parentname，在参数中设置 vstoreId 和 parentname，创建 DTree 配额，并在卷对象上设置 DTreeParentName

#### Scenario:查询 DTree 卷
- **WHEN** 为 DTree 卷调用 QueryVolume 时
- **THEN** 插件从参数或后端配置确定有效的 parentname，并按名称、parentName 和 vStoreId 查询 DTree

#### Scenario:删除 DTree 卷
- **WHEN** 调用 DeleteDTreeVolume 时
- **THEN** 插件使用卷名称、父名称和 vStoreId 移除 DTree 配额

#### Scenario:扩容 DTree 卷
- **WHEN** 调用 ExpandDTreeVolume 时
- **THEN** 插件使用 TransK8SCapacity 将 spaceHardQuota 从扇区大小转换为字节，在阵列上扩容 DTree 配额，并返回 NodeExpansionRequired=false

#### Scenario:使用自动认证客户端附加 DTree 卷
- **WHEN** 为 DTree 卷调用 AttachVolume 时
- **THEN** 插件在映射信息中返回 DTreeParentName；如果启用了 nfsAutoAuthClient，则按 CIDR 过滤节点 IP，并在 DTree 上使用 ReadWrite 访问调用 AutoManageAuthClient

#### Scenario:分离 DTree 卷时清理自动认证客户端
- **WHEN** 为 DTree 卷调用 DetachVolume 时
- **THEN** 如果启用了 nfsAutoAuthClient，插件按 CIDR 过滤节点 IP，使用 NoAccess 调用 AutoManageAuthClient，如果 IOIsolation 为 true，则在返回之前检查所有客户端状态

#### Scenario:拒绝 DTree 上不支持的操作
- **WHEN** 调用 DeleteVolume（非 DTree 变体）、ExpandVolume（非 DTree 变体）、CreateSnapshot、DeleteSnapshot 或 ModifyVolume 时
- **THEN** 插件返回错误，指示操作未实现（请使用 DTree 特定变体）

#### Scenario:DTree 禁用高级能力
- **WHEN** 调用 UpdateBackendCapabilities 时
- **THEN** 插件继承自 OceanstorPlugin，但设置 SupportMetro=false、SupportMetroNAS=false、SupportReplication=false、SupportClone=false、SupportApplicationType=false、SupportQoS=false；为 Dorado 更新 SupportThin，并从 NFS 服务设置更新 NFS4 能力

#### Scenario:DTree 池容量为零
- **WHEN** 调用 UpdatePoolCapabilities 时
- **THEN** 插件对所有池名称返回零容量（DTree 使用基于配额的容量，而非基于池的容量）

#### Scenario:验证 DTree 参数
- **WHEN** 调用 Validate 时
- **THEN** 插件验证配置参数，验证 DTree 特定参数（protocol、portals、parentname），创建测试客户端，执行 ValidateLogin，然后登出

#### Scenario:使用 parentname 和 Dorado V6/V7 名称模板创建 DTree 卷
- **WHEN** 为 DTree 卷调用 CreateVolume 时
- **THEN** 插件从 PV 名称模板解析卷名称（验证包含 {{.PVCNamespace}} 和 {{.PVCName}}，使用元数据执行并追加 "-{{.PVCUid}}"）；验证 parentname：如果 StorageClass 和后端 parentname 都设置为不同值，则返回错误

#### Scenario:附加 DTree 卷时解析 parentName
- **WHEN** 为 DTree 卷调用 AttachVolume 时
- **THEN** attachDTreeVolume 函数首先检查 volumeContext[DTreeParentKey]；如果找到则返回；否则回退到 p.parentName；如果启用了 nfsAutoAuthClient 且无法确定 parentName，则返回错误 "failed to get parent name"

---

### Requirement: A-Series plugin SHALL manage OceanStor A-Series NAS and DTree
A-Series 插件（OceanstorASeriesPlugin）SHALL 为 oceanstor-a-series-nas 存储类型实现 StoragePlugin 接口，支持 NFS 和 DTFS 协议。

#### Scenario:初始化 A-Series 插件
- **WHEN** 插件初始化时
- **THEN** 它验证协议为 "nfs" 或 "dtfs"，验证门户（NFS 必需且恰好 1 个门户；DTFS 不需要），创建 A-Series 客户端，登录，设置系统信息，并根据 keepLogin 标志可选登出

#### Scenario:创建 A-Series NAS 卷
- **WHEN** 为 oceanstor-a-series-nas 调用 CreateVolume 时
- **THEN** 插件从 PV 名称或参数解析卷名称，将参数转换为 CreateASeriesVolumeParameter 结构，生成带有协议的卷模型，并在 A-Series 存储上创建 FileSystem/DTree

#### Scenario:查询带工作负载类型的 A-Series 卷
- **WHEN** 调用 QueryVolume 时
- **THEN** 插件从参数中提取 applicationType 作为 workloadType，并通过 A-Series API 查询文件系统

#### Scenario:删除带 KvCacheStoreId 的 A-Series NAS 卷
- **WHEN** 为 A-Series NAS 卷调用 DeleteVolume 时
- **THEN** 插件从 params 中提取 KvCacheStoreId（如果存在），并将其包含在 A-Series API 的删除模型中

#### Scenario:扩容 A-Series 卷
- **WHEN** 调用 ExpandVolume 时
- **THEN** 插件通过 A-Series API 扩容文件系统容量，并返回 NodeExpansionRequired=false

#### Scenario:处理 A-Series 特定协议
- **WHEN** 插件初始化时
- **THEN** 它验证 A-Series 存储的协议为 "nfs" 或 "dtfs"

#### Scenario:来自许可证的 A-Series 能力
- **WHEN** 调用 UpdateBackendCapabilities 时
- **THEN** 插件查询许可证功能并设置：SupportThin=true、SupportApplicationType=true、基于 SmartQoS 功能的 SupportQoS、SupportThick=false、SupportMetro=false、SupportReplication=false、SupportClone=false；还查询 NFS 服务设置以更新 NFS 协议能力（SupportNFS3/4/41/42）

#### Scenario:A-Series 规格包含 WWN
- **WHEN** 调用 getBackendSpecifications 时
- **THEN** 插件返回 LocalDeviceSN、VStoreID、VStoreName 和 DeviceWWN

#### Scenario:使用 vStore 配额更新池能力
- **WHEN** 调用 UpdatePoolCapabilities 时
- **THEN** 插件查询 vStore 容量（如果设置了 vStoreName），解析 nasCapacityQuota 和 nasFreeCapacityQuota，并与池数据合并；如果未设置配额（nasCapacityQuota=0 或 nasFreeCapacityQuota=-1），则返回空容量映射

#### Scenario:拒绝 A-Series 上不支持的操作
- **WHEN** 调用 CreateSnapshot、DeleteSnapshot、DeleteDTreeVolume、ExpandDTreeVolume 或 ModifyVolume 时
- **THEN** 插件返回错误，指示存储类型不支持请求的功能

#### Scenario:SupportQoSParameters 验证 A-Series 的范围
- **WHEN** 调用 SupportQoSParameters 时
- **THEN** 插件调用 smartx.CheckQoSParametersValueRange 验证 QoS 参数值

#### Scenario:验证 A-Series 参数
- **WHEN** 调用 Validate 时
- **THEN** 插件验证参数、协议、门户，创建测试客户端，执行 ValidateLogin，然后登出

#### Scenario:使用高级选项创建 A-Series 卷
- **WHEN** 使用设置为 JSON 字符串的 advancedOptions 参数调用 CreateVolume 时
- **THEN** genCreateVolumeModel 函数将 JSON 解组到卷创建模型中，允许传递存储特定选项
- **注意**：如果设置了 EnableKVCache，则传递 EnableKVCache、EnableTimeAwareGC 和 GCTimeThreshold 用于 KVCache 支持；对于 DTFS 协议，authUser 是必需的，authUsers 按 ";" 分割

#### Scenario:使用 parentname 创建 A-Series DTree 卷
- **WHEN** 为 oceanstor-aseries-dtree 调用 CreateVolume 时
- **THEN** OceanstorASeriesDtreePlugin 验证可选的 parentname 参数，将其与协议一起传递给 genCreateDTreeModel，并在 A-Series 存储上创建 DTree

#### Scenario:A-Series DTree 能力和容量覆盖
- **WHEN** 为 oceanstor-aseries-dtree 调用 UpdateBackendCapabilities 时
- **THEN** 插件调用父级 A-Series 方法并覆盖：SupportApplicationType=false、SupportQoS=false、SupportThick=false、SupportQuota=true；UpdatePoolCapabilities 通过 getZeroPoolsCapacities 返回零容量

---

### Requirement: DME plugin SHALL manage A-Series via DME
DME 插件（DMEASeriesPlugin）SHALL 为 oceanstor-a-series-nas-dme 存储类型实现 StoragePlugin 接口，通过 DME（数据管理引擎）API 管理 A-Series 存储。它支持 NFS 和 DTFS 协议。

#### Scenario:初始化 DME A-Series 插件
- **WHEN** 插件初始化时
- **THEN** 它验证协议为 "nfs" 或 "dtfs"，验证门户（NFS 必需且恰好 1 个门户；DTFS 不需要），验证 config 中存在 storageDeviceSN，创建 DME 客户端，登录，使用设备 SN 设置系统信息，并根据 keepLogin 标志可选登出

#### Scenario:通过 DME 创建卷
- **WHEN** 调用 CreateVolume 时
- **THEN** 插件从 PV 名称或参数解析卷名称，将参数转换为 CreateDmeVolumeParameter 结构，生成带有协议和扇区大小的卷模型，并通过 DME API 创建卷

#### Scenario:通过 DME 查询卷
- **WHEN** 调用 QueryVolume 时
- **THEN** 插件通过 DME API 按名称查询卷，并返回卷元数据

#### Scenario:通过 DME 删除卷
- **WHEN** 调用 DeleteVolume 时
- **THEN** 插件通过 DME API 按名称和协议删除卷

#### Scenario:通过 DME 扩容卷
- **WHEN** 调用 ExpandVolume 时
- **THEN** 插件通过 DME API 扩容卷容量（size * sectorSize），并返回 NodeExpansionRequired=false

#### Scenario:为 DME 设置默认 maxClientThreads
- **WHEN** 后端配置中未指定 maxClientThreads 时
- **THEN** CLI 将其设置为 DMEDefaultMaxClientThreads（与标准默认值不同）

#### Scenario:DME 能力是固定的
- **WHEN** 调用 UpdateBackendCapabilities 时
- **THEN** 插件返回固定能力：SupportThin=true，其他所有（SupportApplicationType、SupportQoS、SupportThick、SupportMetro、SupportReplication、SupportClone、SupportMetroNAS、SupportConsistentSnapshot）= false

#### Scenario:拒绝 DME A-Series 上不支持的操作
- **WHEN** 调用 CreateSnapshot、DeleteSnapshot、DeleteDTreeVolume、ExpandDTreeVolume 或 ModifyVolume 时
- **THEN** 插件返回错误，指示存储类型不支持请求的功能

#### Scenario:SupportQoSParameters 对 DME 始终通过
- **WHEN** 调用 SupportQoSParameters 时
- **THEN** 插件返回 nil（始终通过，不进行实际验证）

#### Scenario:通过 DME 更新池能力
- **WHEN** 调用 UpdatePoolCapabilities 时
- **THEN** 插件从 DME 查询 HyperScale 池，使用 DmeCapacityUnitMb 将容量从 MB 转换为字节，并返回池容量映射

#### Scenario:验证 DME 参数
- **WHEN** 调用 Validate 时
- **THEN** 插件验证参数、协议、门户，创建测试客户端，执行 ValidateLogin，然后登出

---

### Requirement: FusionStorage plugin SHALL manage FusionStorage distributed storage
FusionStorage 插件 SHALL 为 fusionstorage-san、fusionstorage-nas 和 fusionstorage-dtree 存储类型实现 StoragePlugin 接口。

#### Scenario:创建 FusionStorage SAN 卷
- **WHEN** 在 fusionstorage-san 上使用 volumeType=lun 调用 CreateVolume 时
- **THEN** 插件在 FusionStorage 集群上创建具有指定大小和池的卷

#### Scenario:创建 FusionStorage NAS 卷
- **WHEN** 在 fusionstorage-nas 上使用 volumeType=fs 调用 CreateVolume 时
- **THEN** 插件在 FusionStorage 集群上创建共享文件系统

#### Scenario:创建 FusionStorage DTree 卷
- **WHEN** 在 fusionstorage-dtree 上使用 volumeType=dtree 调用 CreateVolume 时
- **THEN** 插件创建具有指定容量的 DTree 配额（1 字节分配单元）

#### Scenario:处理 FusionStorage 分配单元
- **WHEN** 为 SAN 卷计算容量时
- **THEN** 插件使用 FusionAllocUnitBytes（1MB）作为分配单元
- **WHEN** 为 NAS 卷计算容量时
- **THEN** 插件使用 FusionFileCapacityUnit（1KB）作为容量单元

#### Scenario:FusionStorage SAN 支持 QoS 和 Clone
- **WHEN** 为 fusionstorage-san 调用 UpdateBackendCapabilities 时
- **THEN** 插件设置 SupportThin=true、SupportThick=false、SupportQoS=true、SupportClone=true

#### Scenario:FusionStorage NAS 支持 Quota
- **WHEN** 为 fusionstorage-nas 调用 UpdateBackendCapabilities 时
- **THEN** 插件设置 SupportThin=true、SupportThick=false、SupportQoS=true、SupportQuota=true、SupportClone=false，并通过 updateNFS4Capability 检查 NFS4/NFS41 能力

#### Scenario:拒绝 FusionStorage 上不支持的操作
- **WHEN** 为 fusionstorage-nas 调用 CreateSnapshot 时
- **THEN** 插件返回错误 "fusionstorage-nas not support snapshot feature"
- **WHEN** 为 fusionstorage-dtree 调用 DeleteVolume 时
- **THEN** 插件返回错误 "fusionstorage-dtree not support DeleteVolume feature"；请使用 DeleteDTreeVolume
- **WHEN** 为 fusionstorage-dtree 调用 ExpandVolume 时
- **THEN** 插件返回错误 "fusionstorage-dtree not support ExpandVolume feature"；请使用 ExpandDTreeVolume

---

### Requirement: OceanDisk 插件应管理 OceanDisk 存储阵列
OceanDisk 插件 SHALL 为 oceandisk-san 存储类型实现 StoragePlugin 接口，支持 iSCSI、FC、RoCE 和 RoCE-NVMe 协议。

#### Scenario:初始化 OceanDisk SAN 插件
- **WHEN** 插件使用后端配置初始化时
- **THEN** 它验证协议为 [iscsi, fc, roce, roce-nvme] 之一，验证需要门户的协议的门户（iscsi、roce、roce-nvme），创建 OceanDisk 客户端，登录，并使用协议、门户和 ALUA 配置初始化 OceandiskAttacher

#### Scenario:创建 OceanDisk 卷
- **WHEN** 在 oceandisk-san 上调用 CreateVolume 时
- **THEN** 插件在 OceanDisk 阵列上创建具有指定参数的命名空间（LUN）

#### Scenario:附加 OceanDisk 卷
- **WHEN** 调用 AttachVolume 时
- **THEN** 插件按名称获取命名空间信息，使用协议特定参数调用附加器的 ControllerAttach，并返回映射信息（portals、IQNs/WWNs、LUN WWN、主机 LUN ID）

#### Scenario:分离 OceanDisk 卷
- **WHEN** 调用 DetachVolume 时
- **THEN** 插件按名称获取命名空间信息；如果命名空间不存在，记录警告并返回成功（幂等）；否则调用 ControllerDetach

#### Scenario:删除 OceanDisk 卷
- **WHEN** 调用 DeleteVolume 时
- **THEN** 插件从 OceanDisk 阵列删除命名空间

#### Scenario:查询 OceanDisk 卷
- **WHEN** 调用 QueryVolume 时
- **THEN** 插件按名称查询命名空间，并返回卷元数据

#### Scenario:扩容 OceanDisk 卷
- **WHEN** 调用 ExpandVolume 时
- **THEN** 插件将命名空间扩容到新大小，并返回是否需要节点扩容

#### Scenario:拒绝 OceanDisk 上不支持的操作
- **WHEN** 在 oceandisk-san 上调用 CreateSnapshot、DeleteSnapshot、DeleteDTreeVolume、ExpandDTreeVolume 或 ModifyVolume 时
- **THEN** 插件返回错误，指示存储类型不支持请求的功能

#### Scenario:验证 OceanDisk SAN 参数
- **WHEN** 使用后端配置调用 Validate 时
- **THEN** 插件验证参数存在，验证协议和门户，创建测试客户端，执行 ValidateLogin，然后登出

#### Scenario:获取 OceanDisk 的扇区大小
- **WHEN** 调用 GetSectorSize 时
- **THEN** 插件返回 512 字节（标准扇区大小）
