## Purpose

定义 storage-backend-controller 和 sidecar-controller 的行为规范，包括 SBC/SBCT 生命周期管理、状态同步、finalizer 管理以及通过 DR-CSI 获取后端状态。

## Requirements

### Requirement: storage-backend-controller SHALL manage SBC lifecycle
storage-backend-controller SHALL 监听 StorageBackendClaim 资源，验证引用的 ConfigMap 和 Secret 资源，创建绑定到 SBC 的 StorageBackendContent，管理 finalizer，并更新 SBC 状态。

#### Scenario:同步新的 StorageBackendClaim
- **WHEN** 创建新的 SBC，且设置了 Provider、ConfigMapMeta 和 SecretMeta 时
- **THEN** 控制器执行同步任务流：将状态设置为 Pending，移除过时的 ConfigMap/Secret finalizer，如果设置了 DeletionTimestamp 则处理删除，添加 ClaimBoundFinalizer，创建 StorageBackendContent（命名为 "content-<claim-UID>"），从 Content 状态更新 SBC 状态，并将 spec 变更传播到 Content；Content 的 Spec 包括 Provider、ConfigmapMeta、SecretMeta、BackendClaim、MaxClientThreads 和 Parameters

#### Scenario:同步 SBC 更新
- **WHEN** 更新现有 SBC（例如 MaxClientThreads、SecretMeta、UseCert、CertSecret）时
- **THEN** 控制器通过 NeedChangeContent 检测变更，更新绑定的 StorageBackendContent 的 Spec 字段（MaxClientThreads、SecretMeta、UseCert、CertSecret）以匹配 SBC 的 Spec；控制器不会更新外部 ConfigMap 或 Secret 资源本身——这些由 CLI 工具或外部进程管理

#### Scenario:同步 SBC 时 Content 创建的幂等性
- **WHEN** createContentTask 创建 StorageBackendContent 但它已存在（竞态条件）时
- **THEN** 控制器捕获创建错误，调用 GetContent 验证存在性，并重用现有 Content 而不返回错误

#### Scenario:仅当 Content 就绪时将 SBC 的 Phase 设置为 Bound
- **WHEN** updateClaimStatusTask 运行且绑定的 Content 的 Status.VendorName 已设置（非空）时
- **THEN** isUpdateFinalClaimStatus 函数将 SBC Phase 设置为 "Bound"；如果 Content.Status 为 nil 或 ContentName/VendorName 为空，则 Phase 保持不变

---

#### Scenario:删除带有绑定 Content 的 SBC
- **WHEN** 带有 BoundContentName 的 StorageBackendClaim 被删除时
- **THEN** 控制器删除绑定的 StorageBackendContent，移除 ConfigMap 上的 finalizer，并删除 ConfigMap 和 Secret

#### Scenario:删除不带绑定 Content 的 SBC
- **WHEN** 不带 BoundContentName 的 StorageBackendClaim 被删除时
- **THEN** 控制器清理任何部分创建的 ConfigMap 和 Secret 资源

#### Scenario:删除带证书的 SBC
- **WHEN** 设置了 CertSecret 的 StorageBackendClaim 被删除时
- **THEN** 控制器还会删除证书 Secret 资源

---

#### Scenario:同步 Content 时管理 finalizer
- **WHEN** 创建或更新 StorageBackendContent 时
- **THEN** 主控制器检查 Content 是否需要添加 ContentBoundFinalizer（DeletionTimestamp 为 nil 且 finalizer 不存在）；如果需要，则添加 finalizer 并通过 API 更新 Content

#### Scenario:同步 Content 时在删除时移除 finalizer
- **WHEN** StorageBackendContent 设置了 DeletionTimestamp 且带有 ContentBoundFinalizer 时
- **THEN** 主控制器移除 finalizer，通过 API 更新 Content（带有 ResourceExpired 重试），并更新本地 contentStore

#### Scenario:同步 Content 时触发 claim 状态更新
- **WHEN** 主控制器的 content-sync 运行且绑定的 SBC 需要状态更新（claim.Status 为 nil、BoundContentName 为空、Content.Status.ContentName 已设置时 StorageBackendId 为空，或 Phase 不是 Bound）时
- **THEN** 控制器将 claim 键入队到 claimQueue 进行重新处理

#### Scenario:同步 Content 时不直接查询后端
- **WHEN** 主控制器的 content-sync 运行时
- **THEN** 主控制器不会直接查询存储阵列获取能力、容量或池；这些 exclusively 由 sidecar 控制器通过 DR-CSI GetBackendStats gRPC 调用更新

---

#### Scenario:删除带有绑定 Claim 的 Content
- **WHEN** 带有 BackendClaim 引用的 StorageBackendContent 被删除时
- **THEN** 控制器清除绑定的 SBC 的 BoundContentName 并将其 Phase 重置为 "Pending"

#### Scenario:删除不带绑定 Claim 的 Content
- **WHEN** 不带 BackendClaim 引用的 StorageBackendContent 被删除时
- **THEN** 控制器完成删除而无需额外清理

---

### Requirement: sidecar-controller SHALL sync backend status via DR-CSI
sidecar 控制器 SHALL 处理 StorageBackendContent 监听器事件（Add/Update/Delete），并通过 DR-CSI 提供方的 GetBackendStats gRPC 服务触发状态更新。

#### Scenario:通过 DR-CSI 轮询后端状态
- **WHEN** sidecar 控制器的 worker 从 content 队列获取 SBCT 时
- **THEN** syncContent 任务流运行：initContentStatusTask 在状态为 nil 时初始化状态，deleteContentTask 在设置了 DeletionTimestamp 时处理删除，createContentTask 在未就绪时向提供方注册后端，updateContentTask 在 spec 变更时更新后端凭据，getContentTask 通过 GetBackendStats gRPC 调用获取统计信息

#### Scenario:仅同步现有 SBCT 的后端状态
- **WHEN** sidecar 控制器处理 SBCT 监听器事件时
- **THEN** 它仅处理作为 Kubernetes CRD 存在的 SBCT；sidecar 不会从 DR-CSI 提供方发现没有对应 SBCT CRD 的新后端

#### Scenario:处理 DR-CSI 连接失败
- **WHEN** sidecar 控制器的 GetBackendStats gRPC 调用因连接错误失败时
- **THEN** 错误沿任务流向上传播，条目以指数退避重新入队（起始 5 秒，最大 5 分钟），并将在下一个 worker 周期重试

#### Scenario:跳过不匹配提供方的 sidecar 处理
- **WHEN** sidecar 控制器收到 SBCT 事件时
- **THEN** isMatchProvider 函数检查 content.Spec.Provider 是否匹配 sidecar 的提供方名称；如果不匹配，则跳过该 SBCT（每个 sidecar 实例仅处理其自身提供方的 SBCT）

#### Scenario:使用 needEnQueue 优化入队
- **WHEN** sidecar 监听器收到 SBCT Update 事件时
- **THEN** needEnQueue 函数检查是否仅状态中的 Pools、Capabilities 或 Specification 字段发生了变更；如果是，则跳过入队以防止无限循环（因为控制器自身更新这些字段）

---

#### Scenario:从提供方响应更新 Online 状态
- **WHEN** sidecar 控制器的 getContentTask 通过 DR-CSI 提供方调用 GetBackendStats 时
- **THEN** 提供方在获取统计信息之前检查 IsSBCTOnline；如果在线，响应包含 online=true；shouldUpdateContentStatus 函数比较响应的 Online 字段与当前 SBCT 状态，并在变更时更新

#### Scenario:后端不可达时将 Online 状态更新为 false
- **WHEN** 存储插件客户端（OceanStor、FusionStorage 等）无法与存储阵列通信时
- **THEN** 插件调用 SetStorageBackendContentOnlineStatus 将 SBCT.Status.Online 设置为 false 并发布 BackendStatus 事件；这由存储插件处理，而非直接由 sidecar 控制器处理

#### Scenario:更新池容量（无聚合 Capacity）
- **WHEN** sidecar 控制器收到带有池数据的 GetBackendStatsResponse 时
- **THEN** shouldUpdateContentStatus 函数使用每个池的名称和容量（FreeCapacity、TotalCapacity、UsedCapacity）更新 SBCT.Status.Pools；聚合的 SBCT.Status.Capacity 映射（TotalCapacity/UsedCapacity/FreeCapacity）永远不会被 sidecar 或主控制器填充

#### Scenario:更新后端能力
- **WHEN** sidecar 控制器收到带有能力数据的 GetBackendStatsResponse 时
- **THEN** shouldUpdateContentStatus 函数通过 DeepEqual 比较更新 SBCT.Status.Capabilities，包含：SupportThin、SupportThick、SupportQoS、SupportMetro、SupportReplication、SupportClone、SupportApplicationType、SupportMetroNAS 和 NFS 协议支持标志

#### Scenario:更新设备规格
- **WHEN** sidecar 控制器收到带有规格数据的 GetBackendStatsResponse 时
- **THEN** shouldUpdateContentStatus 函数使用 LocalDeviceSN、RemoteDevicesSN、VStoreID、VStoreName 更新 SBCT.Status.Specification，并使用设备序列号更新 SBCT.Status.SN

#### Scenario:更新供应商和版本
- **WHEN** sidecar 控制器收到 GetBackendStatsResponse 时
- **THEN** shouldUpdateContentStatus 函数更新 SBCT.Status.VendorName（例如 "Huawei"）和 SBCT.Status.ProviderVersion（CSI 驱动版本）

---

#### Scenario:SBCT CRD 删除时删除 Content
- **WHEN** sidecar 监听器收到 StorageBackendContent 的 Delete 事件时
- **THEN** syncContentByKey 函数检测到内容在监听器 lister 中未找到但存在于本地 contentStore 中，并调用 deleteContentCache 将其从本地存储移除

#### Scenario:Content 删除时从提供方移除后端
- **WHEN** sidecar 的 deleteContentTask 对设置了 DeletionTimestamp 的 Content 运行时
- **THEN** 如果 Content 有已注册的 backendId（Status.ContentName 非空），removeProviderBackend 函数调用 DR-CSI 提供方的 RemoveStorageBackend 注销后端，然后清除 Content 状态

#### Scenario:处理后端从未注册时的 Content 删除
- **WHEN** deleteContentTask 对 Status=nil 或 Status.ContentName="" 的 Content 运行时
- **THEN** removeProviderBackend 函数返回 nil 而不调用 RemoveStorageBackend（无需清理）
