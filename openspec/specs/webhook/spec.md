## Purpose

定义 StorageBackendClaim 准入验证 Webhook 规范，在创建和更新时验证 SBC 资源，确保满足架构和业务规则，包括 Provider 必填校验、ConfigMapMeta/SecretMeta 格式校验和不可变字段校验。

## Requirements

### Requirement: sbc-validation SHALL validate StorageBackendClaim on create and update
准入 Webhook SHALL 验证 StorageBackendClaim 资源，以确保它们在持久化之前满足所需的架构和业务规则。Webhook 路径为 "/storagebackendclaim"，失败策略为 "Fail"。不执行变更操作——Webhook 纯粹是验证性的。

#### Scenario:验证带有必需 Provider 字段的 SBC
- **WHEN** 用户创建不带 Provider 字段的 SBC 时
- **THEN** validateCommonClaim 函数拒绝请求，错误为 "Provider in StorageBackendClaim [%s] can not be empty"

#### 场景：验证 SBC ConfigMapMeta 格式
- **WHEN** 用户创建 ConfigMapMeta 不符合 "<namespace>/<name>" 格式的 SBC 时
- **THEN** validateCommon 函数拒绝请求；格式在 backend.GetStorageBackendInfo() 期间通过 SplitMetaNamespaceKey 间接验证，返回错误 "split configmap meta %s namespace failed"

#### 场景：验证 SBC SecretMeta 格式
- **WHEN** 用户创建 SecretMeta 不符合 "<namespace>/<name>" 格式的 SBC 时
- **THEN** validateCommon 函数拒绝请求；格式在 backend.GetStorageBackendInfo() 期间通过 SplitMetaNamespaceKey 间接验证，返回错误 "split secret meta %s namespace failed"

#### 场景：验证 SBC ConfigMapMeta 不为空
- **WHEN** 用户创建 ConfigMapMeta 为空的 SBC 时
- **THEN** validateCommonClaim 函数拒绝请求，错误为 "StorageBackendClaim %s's configmap [%s] is empty"

#### 场景：验证 SBC SecretMeta 不为空
- **WHEN** 用户创建 SecretMeta 为空的 SBC 时
- **THEN** validateCommonClaim 函数拒绝请求，错误为 "StorageBackendClaim %s's secret [%s] is empty"

#### 场景：验证 SBC 更新不更改不可变字段
- **WHEN** 用户更新 SBC 的 Provider 字段时
- **THEN** validateUpdate 函数拒绝请求，错误为 "[provider] is forbidden changed with StorageBackendClaim %s"；如果仅 Spec 和 Annotations 未更改，则允许更新而无需进一步验证

#### 场景：验证配置有效的 SBC
- **WHEN** 用户创建带有所有必需字段和有效格式的 SBC 时
- **THEN** Webhook 执行完整的后端验证：检索 ConfigMap 和 Secret，构建 Backend 对象，验证存储类型和插件存在性，验证参数，并调用 Plugin.Validate() 执行到存储阵列的登录测试；如果全部通过，则允许请求

#### 场景：验证带有 finalizer 的 SBC 删除
- **WHEN** 用户删除带有 ClaimBoundFinalizer 之外的 finalizer 的 SBC 时
- **THEN** validateDelete 函数拒绝请求，错误为 "forbid delete StorageBackendClaim %s, there are some finalizers [%v]"；仅允许 ClaimBoundFinalizer（"storagebackend.xuanwu.huawei.io/storagebackendclaim-bound-protection"）
