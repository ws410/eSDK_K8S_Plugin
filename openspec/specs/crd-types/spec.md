## Purpose

定义存储后端和卷管理系统的四个自定义资源定义（CRD）：StorageBackendClaim、StorageBackendContent、VolumeModifyClaim 和 VolumeModifyContent，包括其字段规范、状态机和验证规则。

## Requirements

### Requirement: CRD types SHALL define custom resources for backend and volume management
系统 SHALL 定义四个自定义资源定义：StorageBackendClaim（SBC）、StorageBackendContent（SBCT）、VolumeModifyClaim（VMC）和 VolumeModifyContent。SBC/SBCT 遵循 PVC-PV 模式进行后端管理；VMC/VMContent 遵循类作业模式进行批量卷修改。

#### Scenario: CRD 类型注册
- **WHEN** CSI 驱动启动时
- **THEN** 系统 SHALL 注册四个 CRD 类型到 Kubernetes API 服务器：StorageBackendClaim、StorageBackendContent、VolumeModifyClaim 和 VolumeModifyContent

---

### Requirement: StorageBackendClaim CRD SHALL define user-facing backend requests
StorageBackendClaim（SBC）SHALL 是一个命名空间级别的自定义资源，代表用户配置存储后端的请求。它遵循 PVC-PV 模式，其中 Claim 是用户Facing的请求，Content 是集群范围的实际配置。

#### Scenario:SBC Spec 字段
- **WHEN** 用户创建 StorageBackendClaim 时
- **THEN** Spec 必须包含：Provider（必需，过滤提供方），并可可选包含：ConfigMapMeta（存储管理信息的 namespace/name 格式）、SecretMeta（敏感信息的 namespace/name 格式）、MaxClientThreads（限制存储客户端连接数）、Parameters（用户定义的扩展参数）、UseCert（布尔值，默认 false，启用证书使用）、CertSecret（证书 Secret 的名称）

#### Scenario:SBC Status 字段
- **WHEN** SBC 被控制器处理时
- **THEN** Status 被填充：Phase（Pending/Bound/Unavailable）、StorageBackendId（唯一后端标识符）、ConfigmapMeta（当前 configmap 的 namespace/name）、SecretMeta（当前 secret 的 namespace/name）、MaxClientThreads（当前值）、BoundContentName（绑定的 StorageBackendContent 引用）、StorageType（例如 oceanstor-san）、Protocol（例如 iscsi, nfs）、MetroBackend（双活伙伴后端）、UseCert、CertSecret

#### Scenario:SBC 生命周期阶段
- **WHEN** 创建新的 SBC 时
- **THEN** 其 Phase 设置为 "Pending"
- **WHEN** 控制器将其绑定到 StorageBackendContent 时
- **THEN** 其 Phase 设置为 "Bound"
- **WHEN** 后端登录失败（例如密码错误）时
- **THEN** 其 Phase 设置为 "Unavailable"

#### Scenario:SBC 打印列
- **WHEN** 用户运行 `kubectl get sbc` 时
- **THEN** 输出包含列：StorageBackendContentName、Status、Age（使用 -o wide 时还包括：StorageType、Protocol、MetroBackend）

#### Scenario:SBC 短名称
- **WHEN** 用户使用短名称时
- **THEN** `sbc` 被接受为 StorageBackendClaim 的别名

#### Scenario:SBC 不可变的 Provider 字段
- **WHEN** 用户尝试更新现有 SBC 的 Provider 字段时
- **THEN** 更新被准入 Webhook 拒绝（Provider 在创建后不可变）

#### Scenario:启用 UseCert 的 SBC
- **WHEN** 用户创建 UseCert=true 的 SBC 时
- **THEN** CertSecret 字段必须填充证书 Secret 名称；控制器将使用证书进行后端认证

---

### Requirement: StorageBackendContent CRD SHALL represent actual backend configuration
StorageBackendContent（SBCT）SHALL 是一个集群范围的自定义资源，表示实际的存储后端配置。它绑定到 StorageBackendClaim，包含池信息、容量、能力和在线状态。

#### Scenario:SBCT Spec 字段
- **WHEN** 控制器创建 StorageBackendContent 时
- **THEN** Spec 包含：Provider（必需，匹配 SBC）、ConfigmapMeta（当前 configmap 的 namespace/name）、SecretMeta（当前 secret 的 namespace/name）、BackendClaim（绑定的 SBC namespace/name）、MaxClientThreads、Parameters（扩展参数）、UseCert、CertSecret

#### Scenario:SBCT Status 字段
- **WHEN** sidecar 控制器更新 SBCT 状态时
- **THEN** Status 包含：ContentName（标识：provider-name@backend-name#pool-name）、VendorName（例如 Huawei）、ProviderVersion（CSI 驱动版本）、Pools（池名称和容量数组）、Capacity（TotalCapacity/UsedCapacity/FreeCapacity 映射）、Capabilities（能力名称到布尔值的映射：SupportThin、SupportThick、SupportQoS、SupportMetro、SupportReplication、SupportClone 等）、Specification（设备 SN、VStoreID 等）、ConfigmapMeta、SecretMeta、Online（登录成功标志）、MaxClientThreads、SN（存储设备序列号）、UseCert、CertSecret

#### Scenario:SBCT 打印列
- **WHEN** 用户运行 `kubectl get sbct` 时
- **THEN** 输出包含列：Claim、SN、VendorName、ProviderVersion、Online、Age

#### Scenario:SBCT 短名称
- **WHEN** 用户使用短名称时
- **THEN** `sbct` 被接受为 StorageBackendContent 的别名

#### Scenario:SBCT 是集群范围的
- **WHEN** 用户查询 SBCT 时
- **THEN** 它无需指定命名空间即可访问（集群范围资源）

---

### Requirement: VolumeModifyClaim CRD SHALL define volume modification requests
VolumeModifyClaim（VMC）SHALL 是一个集群范围的自定义资源，代表修改卷的请求（例如 QoS、SmartTier、SmartMigration）。它遵循类作业模式，其中 Claim 跟踪总体进度，Contents 跟踪单个卷修改。

#### Scenario:VMC Spec 字段
- **WHEN** 用户创建 VolumeModifyClaim 时
- **THEN** Spec 必须包含：Source（必需，包含 Kind - 默认 "StorageClass"、Name，以及可选的 Namespace），以及 Parameters（CSI 驱动特定的不透明键值对，用于修改设置）

#### Scenario:VMC Status 字段
- **WHEN** VMC 被控制器处理时
- **THEN** Status 包含：Phase（Pending/Creating/Completed/Rollback/Deleting）、Contents（ModifyContents 数组，包含 ModifyContentName、SourceVolume 和 Status）、Ready（进度指示器，例如 "2/5"）、Parameters（spec 参数的回显）、StartedAt（时间戳）、CompletedAt（时间戳）

#### Scenario:VMC 生命周期阶段
- **WHEN** 创建新的 VMC 时
- **THEN** 其 Phase 设置为 "Pending"
- **WHEN** VContents 正在创建但尚未全部完成时
- **THEN** 其 Phase 设置为 "Creating"
- **WHEN** 所有关联的 VContents 完成时
- **THEN** 其 Phase 设置为 "Completed"
- **WHEN** VMC 收到删除请求并开始回滚时
- **THEN** 其 Phase 设置为 "Rollback"
- **WHEN** VMC 开始删除时
- **THEN** 其 Phase 设置为 "Deleting"

#### Scenario:VMC 打印列
- **WHEN** 用户运行 `kubectl get vmc` 时
- **THEN** 输出包含列：Status、Ready、Age（使用 -o wide 时还包括：SourceKind、SourceName、StartedAt、CompletedAt）

#### Scenario:VMC 短名称
- **WHEN** 用户使用短名称时
- **THEN** `vmc` 被接受为 VolumeModifyClaim 的别名

#### Scenario:VMC 是集群范围的
- **WHEN** 用户查询 VMC 时
- **THEN** 它无需指定命名空间即可访问（集群范围资源）

---

### Requirement: VolumeModifyContent CRD SHALL track individual volume modifications
VolumeModifyContent SHALL 是一个集群范围的自定义资源，跟踪单个卷的修改状态。每个 VolumeModifyClaim 为每个匹配的 PersistentVolume 创建一个 VolumeModifyContent。

#### Scenario:VolumeModifyContent Spec 字段
- **WHEN** 控制器创建 VolumeModifyContent 时
- **THEN** Spec 包含：VMCName（父 VolumeModifyClaim 的引用）、SourceVolume（PVC namespace/name）、Parameters（CSI 驱动特定的修改参数）

#### Scenario:VolumeModifyContent Status 字段
- **WHEN** VolumeModifyContent 被处理时
- **THEN** Status 包含：Phase（Pending/InProgress/Completed/Failed），以及失败时的错误消息

#### Scenario:VolumeModifyContent 生命周期阶段
- **WHEN** 创建 VolumeModifyContent 时
- **THEN** 其 Phase 设置为 "Pending"
- **WHEN** 通过 DR-CSI gRPC 应用修改时
- **THEN** 其 Phase 设置为 "InProgress"
- **WHEN** 修改成功完成时
- **THEN** 其 Phase 设置为 "Completed"
- **WHEN** 修改失败时
- **THEN** 其 Phase 设置为 "Failed"

#### Scenario:VolumeModifyContent 是集群范围的
- **WHEN** 用户查询 VolumeModifyContent 时
- **THEN** 它无需指定命名空间即可访问（集群范围资源）
